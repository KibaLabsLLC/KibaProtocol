# ocean-report-cron (a.k.a. Kiba Protocol, derogatory)

One Cloudflare Worker that: deploys a minimal Ocean-Protocol-style
contract stack to SKALE, mints one dataset listing against it,
compiles a CSV report fresh on every request straight from R2 (never
persisted), sells access to it via Stripe Checkout (a disposable
on-chain wallet does the actual `buyDT()` call per purchase — buyers
never touch a wallet), and bumps the price every 6 biweekly cycles.
The main logic in `src/index.js` has been run through a real minifier
(`terser`) on purpose — every variable is a single letter and there
isn't a comment in it. That was requested, not a mistake. This README
exists so future-you can still operate it without reverse-engineering
it first.

## What's real vs. what to verify

**Real, pulled from Ocean's own published npm packages, not guessed:**
- All six contracts' ABI + bytecode (`src/artifacts.json`) — extracted
  directly from `@oceanprotocol/contracts`'s precompiled artifacts.
- The exact struct packing for `ErcCreateData` / `FixedData` — pulled
  from ocean.js's actual bundled source, not reconstructed from docs.
- The deploy sequence and the `Router.addFixedRateContract()` wiring
  step — found by reading Ocean's own `deploy-contracts.js`.

**Not yet verified against a live chain — test on SKALE testnet first:**
- `Router`'s `bpoolTemplate` constructor argument is passed as
  `0x000000000000000000000000000000004b494241` — the ASCII bytes of
  "KIBA" (`4b494241`), zero-padded out to a 20-byte address — instead
  of the actual zero address (this stack never uses pool-based
  pricing, only fixed rate). Nobody holds the key to that address, so
  it behaves the same as the zero address for our purposes; it's just
  a vanity value that'll be legible if you ever go poking at the
  deployed `Router` in a block explorer. Whether the constructor
  accepts an arbitrary non-zero address here without reverting hasn't
  been tested.
- The whole sequence has not been run end to end against a real SKALE
  RPC. It's built from real, verified pieces, but "verified pieces
  assembled correctly" and "tested" aren't the same thing.
- `/deploy` gets `erc20Address`/`exchangeId` by parsing
  `NFTCreated`/`TokenCreated`/`NewFixedRate` out of the real deploy
  receipt's logs (an earlier version used `.staticCall()` on the same
  function *after* the real tx had already mined — which simulates
  what a fresh call would do against already-mutated state, i.e. a
  different, never-actually-deployed datatoken address and
  `exchangeId`. Every `buyDT`/`setRate`/`collectBT` afterward would
  have been aimed at the wrong exchange). The event names are correct
  per the ABI in `artifacts.json`; still worth confirming on a real
  deploy that all three actually fire.

## One-time setup

1. Get a SKALE chain RPC URL. (`CHAIN_ID_HEX` in `wrangler.toml` is a
   leftover from the old wallet-connect `/buy` page and isn't used for
   anything anymore — harmless to leave, fine to delete.)
2. Create the R2 bucket + KV namespace, fill in `wrangler.toml`.
3. `wrangler secret put PRIVATE_KEY` — this wallet deploys everything,
   owns the exchange, and pays gas for every price bump afterward.
   Fund it with SKALE's gas token first.
4. `wrangler secret put RPC_URL`
5. `wrangler secret put MANUAL_TRIGGER_SECRET` — pick anything, this
   guards `/deploy` and `/run-now`.
6. `npm install && npm run deploy`
7. Deploy the on-chain stack (one time, costs real gas):
   ```
   curl -X POST https://<your-worker>.workers.dev/deploy \
     -H "X-Trigger-Secret: <MANUAL_TRIGGER_SECRET>"
   ```
   The response has every deployed address plus `exchangeId` — these
   are also written straight into KV state as part of this call, so
   there's nothing to copy-paste from the response into a secret.
8. Create a Stripe account (test mode is fine to start), grab the
   secret key, and:
   ```
   wrangler secret put STRIPE_SECRET_KEY
   ```
9. In the Stripe Dashboard, add a webhook endpoint pointing at
   `${WORKER_URL}/webhook/stripe`, listening for
   `checkout.session.completed`. Copy its signing secret:
   ```
   wrangler secret put STRIPE_WEBHOOK_SECRET
   ```
10. Make sure the treasury wallet (`PRIVATE_KEY`) is funded with
    **both** SKALE gas and `KOCEAN` — it now pays for every buyer's
    ephemeral wallet too, not just its own transactions.

## Routes

- `GET /buy` — a "Buy Access — $X" button. No wallet, no network
  switch prompt. Hits `/checkout` and redirects to Stripe.
- `POST /checkout` — creates a Stripe Checkout Session at the current
  price (`currentPrice` from KV, displayed/charged in dollars) and
  returns `{ url }`. Plain Stripe balance, no Connect involved. This
  price is what actually gets charged and is frozen into the session —
  see the webhook note below for why that matters.
- `POST /webhook/stripe` — Stripe calls this on `checkout.session.completed`.
  Verifies the signature (rejecting anything outside a 5-minute
  timestamp window, so a captured header+body can't be replayed
  indefinitely), then runs a step-tracked purchase keyed on the Stripe
  session id: spins up one throwaway wallet, funds it from the
  treasury (sFUEL for gas + KOCEAN), has it call `buyDT()`, then has
  the treasury `collectBT()` to reclaim the KOCEAN. The record (and
  the ephemeral wallet's private key, so a retry reuses the *same*
  wallet instead of orphaning it and minting a new one) is written to
  `STATE` **before any of those five chain calls happen**, and each
  step only re-runs on retry if it didn't already complete. The price
  charged is read from `session.amount_total` — what Stripe actually
  charged at checkout time — never re-read from live `currentPrice`,
  so a cron price bump landing mid-purchase can't cause a
  charged-$N/spent-$M mismatch. If the on-chain leg fails partway,
  this returns 500 so Stripe's automatic retry (up to ~3 days) resumes
  from the right step rather than the buyer being charged for nothing
  or double-bought.
  (Caveat: the idempotency check here is check-then-act on KV, not a
  real lock — fine against Stripe's actual retry behavior, which is
  sequential, but not a guarantee against truly simultaneous
  deliveries. A Durable Object would close that gap properly.)
- `GET /thanks?session_id=...` — Stripe's success redirect target.
  Polls the sale record and shows the download link once the token
  exists.
- `GET /download?token=...` — compiles the CSV fresh from every
  `.json` file under `reports/` in R2 (no persistence, thrown away
  after the response) and deletes the token from `STATE` right after a
  successful response, so it's genuinely single-use, not just
  TTL-bounded. If the compile itself comes back empty, the token is
  left alone so the buyer can retry.
- `GET /status` — current report count, price, deployed stack.
- `POST /run-now?force=true` — manually trigger the price-bump gate
  check without waiting on cron (it no longer compiles or stores a
  CSV, just counts files to decide whether a price bump is due). Same
  `X-Trigger-Secret` header.

## Things that will bite you if you forget them

- The base token here (`MockERC20`, ticker `KOCEAN` by default) is
  **not real OCEAN** — it's a token this stack deploys itself, since
  SKALE doesn't have Ocean's actual token. Buyers never see this
  token now (they pay in USD via Stripe) — it only exists so the
  ephemeral wallet has something to pay the exchange with on-chain.
  KOCEAN never needs real liquidity or a market: after each
  `buyDT()`, the treasury immediately calls `collectBT()` (owner-only,
  and the treasury is the owner) to reclaim exactly what the ephemeral
  wallet just paid in. That's what makes the loop fully circular
  forever off the one-time 100,000-token mint — nothing net leaves
  the treasury's balance per sale. If `collectBT()` ever gets removed
  or starts failing silently, the treasury's KOCEAN balance becomes a
  depleting resource again and sales will eventually start failing
  once it hits zero, same as running out of sFUEL gas.
- Every `/download` request re-lists and re-parses **every** `.json`
  file under `reports/` in R2, live, on that request — nothing's
  cached or persisted. Fine at small scale; as the report set grows
  this gets slower and costs more R2 Class B operations per request
  than the old persist-once-per-cron-cycle version did. Cron also does
  its own full compile every cycle just to get a file count, then
  discards the CSV text — cheap fix later (a count-only mode) if it
  ever matters, not done here.
- Each ephemeral buyer wallet's private key is held in `STATE` (KV)
  for up to 7 days so a retried webhook delivery can resume the same
  wallet instead of orphaning it. Those wallets only ever hold trivial
  amounts (a gas drop + one sale's worth of a token this stack mints
  itself), so the blast radius of that KV entry leaking is low — but
  it's still a private key sitting in plaintext in KV, worth knowing
  about rather than discovering later.
- `/checkout` has no rate limiting or bot protection — anyone can open
  unlimited Stripe Checkout Sessions at the posted price, which is a
  card-testing surface. Not addressed here; Cloudflare's Rate Limiting
  Rules (dashboard config, not code) or Turnstile on `/buy` would be
  the first things to add if this becomes a real listing.
- `PRIVATE_KEY`'s wallet needs SKALE gas funded before `/deploy`, and
  kept funded afterward for price-bump transactions **and now for
  every single sale** — it funds each ephemeral buyer wallet's gas
  and KOCEAN out of its own balance. Nobody else brings tokens to the
  table anymore; if the treasury runs dry, sales fail (though a retry
  after refunding it will now resume from the right step instead of
  starting over).
- If the treasury *stays* dry for longer than Stripe's webhook retry
  window (~3 days), a buyer will have paid Stripe with no download
  token to show for it — Stripe won't auto-refund, so that needs
  manual handling (refund in the Stripe dashboard, or top up the
  treasury — the next webhook retry or a manually replayed one will
  pick the sale back up from its last completed step automatically).
- If `/deploy` partially fails partway through, `state.stack` in KV
  won't be set, so calling it again will just retry from scratch and
  deploy a second full stack. There's no resume-from-partial-failure
  logic — check the error response's `logs` field to see how far it
  got before troubleshooting.
