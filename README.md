# widespread-solana

PYUSD bounty settlement on Solana — the Encode/Aurora leg of the
widespread verified-work protocol.

A bounty market locks **PYUSD** (PayPal's own dollar, Token-2022 on Solana)
in an escrow keypair. When the bridge verdict accepts a claim, the release
sends PYUSD **plus a `sha256(claim | verdict)` memo in the same transaction**
— the payout and its audit anchor are one atomic on-chain record. Nothing is
mocked: every receipt resolves on devnet explorer.

## Run

```bash
npm i
node src/demo.js          # dry run — prints the escrow + what it would send
```

### Go live on devnet (~2 min)

1. `node src/demo.js` → copy the printed `escrow` address
2. Fund it with devnet SOL at https://faucet.solana.com (gas only)
3. Fund it with devnet PYUSD at the Google Cloud Web3 faucet
   (mint `CXk2AMBfi3TwaEL2468s6zP8xq9NxTXjp9gjMgzeUynM`)
4. `CONTRIBUTOR=<recipient-address> DRY=0 node src/demo.js`
   → prints `anchor sig`, `payout sig`, and an explorer link for the receipt

## PYUSD devnet payout dry-run

The payout path has two independent legs, one demo script each:

| Leg | Script | What moves |
|---|---|---|
| Solana escrow payout | `src/demo.js` (+ `src/lib.js`) | PYUSD (Token-2022) from an escrow keypair to the contributor, with memo anchors |
| Cross-chain settlement | `src/intents-demo.js` (+ `src/intents.js`) | NEAR Intents 1Click quote that settles the contributor on Solana |

Both demos are dry by default: they print exactly what they *would*
send, and nothing settles until you fund and confirm for real.

### Memo-anchor format

Every payout is bound to its claim by a content hash (`contentHash`
in `src/lib.js`):

```
sha256("widespread|<repo>|<seq>|<claimHash>|<verdict>")  →  hex
```

For the demo's built-in claim (`slop/protocol`, seq `0`, verdict
`accept`) that is:

```
e876cb1c9ee9388cb865a78d63ae7afda37699f8925e10eda320bdfe19dc373a
```

The hash appears on-chain twice, as memos to the SPL Memo program
(`MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`), in two separate
transactions:

1. **Anchor tx** — a memo-only transaction sent first:
   `widespread:anchor:<full 64-char hash>`
2. **Payout tx** — the memo is the *first* instruction of the PYUSD
   transfer transaction, so memo and payment land atomically:
   `widespread:payout:<first 24 chars of the hash>`
   The same transaction creates the recipient's PYUSD associated
   token account if it doesn't exist yet, then does a
   `TransferChecked` of `amountUi × 10^6` base units (PYUSD has 6
   decimals) against the devnet mint
   `CXk2AMBfi3TwaEL2468s6zP8xq9NxTXjp9gjMgzeUynM` under the
   Token-2022 program.

Anyone can verify a payout after the fact: recompute the content hash
from the claim fields, then check the payout transaction on the
explorer carries the matching `widespread:payout:…` memo and amount.

### Dry run — Solana leg

```bash
npm i
node src/demo.js        # DRY is anything except "0", so this is dry
```

Prints the freshly generated escrow address and its SOL balance
(this still queries devnet RPC — the only network call in dry mode),
the content hash, and the exact commands to go live. It also writes
`/tmp/ws-escrow.json` (the escrow secret) and `/tmp/ws-out.json`
(`{escrow, dry: true}`). No transaction is sent.

The claim is currently hard-coded in `src/demo.js`: 5.0 PYUSD to
`$CONTRIBUTOR`. Going live is
`CONTRIBUTOR=<recipient-address> DRY=0 node src/demo.js`, which sends
the anchor tx, then the payout tx, and prints `anchor sig`,
`payout sig`, and the explorer receipt link. `CONTRIBUTOR` must be
set — the payout throws on an empty recipient address.

### Dry quote — Intents leg

```bash
node src/intents-demo.js
# env overrides: RECIPIENT=<solana-addr> REFUND_TO=<near-acct> AMOUNT_UNITS=<units>
```

This calls `POST https://1click.chaindefuser.com/v0/quote`
(unauthenticated) with `dry: true`, so the API returns a signed
quote and fee breakdown but no deposit is made. The request is
`EXACT_OUTPUT`: `AMOUNT_UNITS` (default `1000000` = 1.0 USDC, 6 dp)
is what the contributor should *receive*, and the quote grosses up
the deposit side. Other fixed fields: `depositType`/`refundType` =
`ORIGIN_CHAIN`, `recipientType` = `DESTINATION_CHAIN`,
`slippageTolerance` = 100 (1%), deadline = now + 1 hour,
`refundTo` defaults to `widespread.near`.

Note the pair the demo actually quotes: **USDC on NEAR → USDC on
Solana** (`nep141:17208628f…` → `TOKENS.SOL_USDC`). This leg is the
cross-chain settlement rail, not a PYUSD quote — PYUSD only appears
in the Solana escrow leg above. (`TOKENS.SOL_USDT` is defined in
`src/intents.js` but not used by the demo.)

Sample output (live quote, amounts vary per quote):

```
payer deposits   : 1.239427 USDC (NEAR)
contributor gets : 1.0 USDC (Solana)
min out / est    : 1000000 / 17 s
quote signature  : ed25519:4nXSy22bp89TKDrESpLdbpi56uY1KeAPBgX1FUgN…
fees             : refund 4428 / withdraw 229203
```

### Settlement verification

A quote or a deposit is **not** a payout. The rule in
`src/intents.js` (post-Oct-2026-incident discipline) is: credit a
payout only after a confirmed final status.

- **Intents leg**: take the `depositAddress` from the quote, make
  the deposit outside this demo, then poll
  `GET /v0/status?depositAddress=<addr>` — `waitSettled()` polls
  every 8 s, up to 20 tries (~160 s). Settled only when the status
  matches `success|settled|completed`; `refund|fail` means it did
  not settle; anything else after 20 tries returns
  `{settled: false, status: 'timeout'}`.
- **Solana leg**: `payWithMemo()` returns the signature only after
  `confirmTransaction(sig, 'confirmed')`. Verify the receipt at
  `https://explorer.solana.com/tx/<sig>?cluster=devnet`: the payout
  memo prefix, the TransferChecked amount, and the recipient ATA
  should all match the claim, and the anchor tx sent just before it
  should carry the full-hash `widespread:anchor:…` memo.

### Known quirks

- **The escrow keypair is regenerated on every run.** `src/demo.js`
  calls `Keypair.generate()` unconditionally and overwrites
  `/tmp/ws-escrow.json`; it never loads the saved keypair back
  (`loadKeypair` in `src/lib.js` is unused by the demo). So the
  address printed by a dry run is *not* the signer of a later
  `DRY=0` run — funding a printed escrow and re-running will not
  spend those funds. Treat the live path as a spec of the intended
  flow until the demo loads a persisted escrow.
- `package.json` declares `payout` and `anchor` scripts pointing at
  `src/payout.js` / `src/anchor.js`, which are not in the repo yet;
  the flow lives entirely in `src/demo.js` for now.

## Why it matters

- **PYUSD is PayPal's regulated dollar on-chain** — the same asset at
  market funding and at settlement; PayPal Payouts stays the fiat off-ramp.
- **Solana is the integrity layer**: content-hash anchors bind
  repo SHA → attestation → payout into permanent on-chain evidence.
- **No platform custody**: the escrow is a plain keypair; release conditions
  are enforced by code, not by an operator. Aurora Intents settles the
  contributor on any chain they choose.
