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

## PYUSD payout flow (devnet)

This section documents the exact bytes and calls behind the demo, so a
reviewer can reproduce the flow without reading the source first.

### 1. What gets anchored, and where

`contentHash()` in `src/lib.js` computes:

```text
sha256("widespread|" + repo + "|" + seq + "|" + claimHash + "|" + verdict)
```

The demo then writes two memos, both through the SPL Memo program:

| Step | Memo text | When |
|---|---|---|
| Anchor | `widespread:anchor:<full 64-hex hash>` | before release, bound to the escrow |
| Payout | `widespread:payout:<first 24 hex of the same hash>` | in the same transaction as the PYUSD transfer |

The payout transaction is atomic: `payWithMemo()` puts the memo, an optional
recipient-ATA creation, and the Token-2022 `transferChecked` in one
transaction, then waits for `confirmed`. The receipt is the transaction on
the devnet explorer, not an off-chain log.

The demo uses the devnet PYUSD mint
`CXk2AMBfi3TwaEL2468s6zP8xq9NxTXjp9gjMgzeUynM` (6 decimals, Token-2022).

### 2. Run the Solana dry run

```bash
npm i
node src/demo.js
```

`src/demo.js` generates a **fresh escrow keypair on every invocation** and
writes its secret to `/tmp/ws-escrow.json`. It prints the escrow address and
stops in dry-run mode by default. To go live on devnet:

```bash
# fund the printed escrow with devnet SOL (gas only)
solana airdrop 1 <escrow>

# fund it with devnet PYUSD from the Google Cloud Web3 faucet
# mint: CXk2AMBfi3TwaEL2468s6zP8xq9NxTXjp9gjMgzeUynM

CONTRIBUTOR=<recipient-address> DRY=0 node src/demo.js
```

The live path prints `anchor sig`, `payout sig`, and the explorer link. Because
the keypair is regenerated each run, do not treat `/tmp/ws-escrow.json` as a
persistent market wallet; it is a demo artifact.

### 3. Intents dry quote

`src/intents-demo.js` exercises the NEAR Intents 1Click quote without
depositing anything:

```bash
node src/intents-demo.js
```

It calls `POST https://1click.chaindefuser.com/v0/quote` with `dry: true`:

```json
{
  "dry": true,
  "swapType": "EXACT_OUTPUT",
  "depositType": "ORIGIN_CHAIN",
  "originAsset": "nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",
  "destinationAsset": "nep141:sol-5ce3bf3a31af18be40ba30f721101b4341690186.omft.near",
  "amount": "1000000",
  "slippageTolerance": 100,
  "refundTo": "widespread.near",
  "refundType": "ORIGIN_CHAIN",
  "recipient": "<Solana recipient>",
  "recipientType": "DESTINATION_CHAIN",
  "deadline": "<UTC ISO-8601, one hour ahead>"
}
```

The default demo quotes **NEAR USDC → Solana USDC**, not PYUSD, and uses the
default recipient `2ZkuVi8kuQsSfnnytvPx5YdhfsqL3eF61p9KhFrtyjVH`. The response
contains the quote, `depositAddress`, estimated output and fees; the demo
stops there. Swap the asset IDs and recipient to quote a different leg.

### 4. Settlement verification

A deposit is not a settlement. `waitSettled()` in `src/intents.js` polls:

```text
GET https://1click.chaindefuser.com/v0/status?depositAddress=<address>
```

Accept only `success`, `settled`, or `completed`. Treat `refund`, `fail`, and
the local `timeout` as **not settled**. For the Solana leg, additionally
confirm the payout transaction (`Connection.confirmTransaction(..., "confirmed")`)
and check both the PYUSD transfer and the memo on the devnet explorer. This is
the post-Oct-2026 discipline: never credit work on deposit alone.

## Why it matters

- **PYUSD is PayPal's regulated dollar on-chain** — the same asset at
  market funding and at settlement; PayPal Payouts stays the fiat off-ramp.
- **Solana is the integrity layer**: content-hash anchors bind
  repo SHA → attestation → payout into permanent on-chain evidence.
- **No platform custody**: the escrow is a plain keypair; release conditions
  are enforced by code, not by an operator. Aurora Intents settles the
  contributor on any chain they choose.

  Note: the current Intents demo quotes **USDC (NEAR) → USDC (Solana)** —
  PYUSD-for-PYUSD on both legs lands when the Intents quote supports it.
