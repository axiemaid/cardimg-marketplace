# CARDIMG Marketplace — Covenant-Based Trading Protocol

> **Status:** Spec phase
> **Date:** 2026-10-15
> **Repo:** `~/.openclaw/cardimg-marketplace/`
> **Concept:** Trustless on-chain marketplace for trading CARDIMG ownership assertions via BSV covenants

## Core Concept

A marketplace where:
- **One seller (the operator)** lists cards at fixed prices — no untrusted sellers
- **Bid covenants** provide always-on liquidity — buyers can sell anytime
- **Bids expire** at chosen block heights — capital isn't locked forever
- **No physical fulfillment** — trades are CARDID ownership transfers, purely on-chain
- **Card images never move** — only ownership assertions change hands

## Architecture

### Protocols Used

| Protocol | Role |
|---|---|
| CARDIMG | Base asset — image data on-chain |
| CARDID | Ownership layer — who owns/claims a CARDIMG tx |
| **MKT2** (this) | Trading layer — covenant-based orderbook for bids and asks |

### Participants

- **Operator** (you) — the sole seller, lists asks, seeds bid liquidity
- **Bidders** — anyone who places bid covenants (market makers, collectors)
- **Sellers** — anyone holding a CARDID claim who wants to sell into a bid
- **Buyers** — anyone who matches an ask covenant

---

## Covenant Design

### Ask Covenant (Operator Lists a Card for Sale)

The operator locks a card listing on-chain. The UTXO can only be spent by:

**Spend path 1 — Match (sale):**
- Buyer pays `ask_price` sats to operator's address
- Buyer provides a CARDID txid proving ownership transfer to their key
- CARDIMG txid is referenced in the covenant

**Spend path 2 — Cancel:**
- Operator reclaims the listing (delists the card)
- Only the operator's key can cancel

```
Ask Covenant:
  Input: operator funding UTXO
  Output 0: OP_FALSE OP_RETURN "MKT2" "ASK" <cardimg_txid> <ask_price> <operator_pubkey>
  Output 1: covenant enforcing:
    - Match: buyer pays ask_price → operator, provides CARDID transfer to buyer's key
    - Cancel: operator signs, reclaims
```

### Bid Covenant (Anyone Places a Bid)

A bidder locks BSV on-chain as a standing offer. The UTXO can only be spent by:

**Spend path 1 — Accept (seller fills the bid):**
- Seller provides a CARDID transfer to the bidder's pubkey
- Seller receives the locked BSV
- CARDIMG txid must match the bid's target

**Spend path 2 — Expiry (bidder reclaims):**
- After `expiry_block`, the bidder reclaims their locked BSV
- Enforced via block-height introspection (OP_PUSH_TX) or OP_CHECKLOCKTIMEVERIFY

```
Bid Covenant:
  Input: bidder funding UTXO (bid_amount sats)
  Output 0: OP_FALSE OP_RETURN "MKT2" "BID" <cardimg_txid> <bid_amount> <bidder_pubkey> <expiry_block>
  Output 1: covenant enforcing:
    - Accept: seller provides CARDID transfer to bidder's key → seller gets bid_amount
    - Expiry: after expiry_block, bidder reclaims bid_amount
```

### Trade Transaction (Bid Acceptance)

```
Trade TX (Bid Acceptance):
  Input 0: Bid covenant UTXO (being spent via Accept path)
  Input 1: Seller's CARDID assertion UTXO (proving current ownership)
  Output 0: OP_FALSE OP_RETURN "CARDID" <cardimg_txid> <bidder_pubkey> <sig>
           (new ownership assertion — seller signs transfer to bidder)
  Output 1: bid_amount sats → seller's address
  Output 2: change → bidder (if any)
```

The trade is atomic — single transaction. Either the whole thing succeeds or nothing does. No counterparty risk.

---

## Liquidity Model

### Always-On Bids

The operator seeds the market with bid covenants at or below market value:
- Creates a bid floor for each card (or category of cards)
- Bids are spread across expiry blocks (rolling expiry)
- When a bid expires, the operator (or an automated agent) re-places it

### Rolling Expiry

Bids don't lock capital forever:
- **Short-term bids:** expire in ~144 blocks (24 hours)
- **Medium-term bids:** expire in ~1008 blocks (1 week)
- **Long-term bids:** expire in ~4320 blocks (1 month)
- An automated agent rolls expired bids — re-places them at current market price
- Capital is recycled, not permanently locked

### Bid Expiry Schedule

```
Day 1:  Place bids expiring in 24h
Day 2:  Day 1 bids expire → re-place at adjusted price
Day 3:  Day 2 bids expire → re-place at adjusted price
...
```

This creates a continuous orderbook where:
- Sellers can always sell into a bid
- Bidders (including operator) aren't locked forever
- Prices adjust based on market activity

---

## Trust Model

### Why This Works Without Trusted Sellers

1. **Single seller (operator)** — Only the operator lists asks. No untrusted third-party sellers.
2. **Bid covenants enforce payment** — A seller selling into a bid doesn't trust the bidder. The covenant guarantees: provide CARDID transfer → receive BSV. Atomic.
3. **Ask covenants enforce delivery** — A buyer matching an ask doesn't trust the operator. The covenant guarantees: pay ask_price → receive CARDID transfer. Atomic.
4. **Expiry protects bidders** — Capital isn't locked forever. Bids auto-unlock.
5. **CARDID provides provenance** — Ownership transfers are verifiable on-chain. The chain of custody is auditable.

### What Could Go Wrong

- **Fake CARDID** — Someone provides a CARDID transfer for a card they don't really own. Mitigation: the covenant should verify the CARDID is signed by the key that created the CARDIMG tx (or a verifiable transfer chain). This is the hardest part.
- **Operator abandons** — Operator stops rolling bids. Market goes illiquid. Not catastrophic — existing bids remain until expiry, then unlock. Asks remain until cancelled or matched.
- **Price manipulation** — Operator sets artificial prices. Market is transparent — all bids/asks are on-chain. Buyers can see the orderbook and decide.

---

## Implementation Plan

### Phase 1: Protocol Spec & Contract
- [ ] Define MKT2 OP_RETURN format precisely
- [ ] Write sScript covenant contract (`ask.scrypt`, `bid.scrypt`)
- [ ] Test covenants on testnet
- [ ] Define CARDID verification logic within covenants

### Phase 2: Indexer & API
- [ ] Build MKT2 indexer — scan for MKT2 OP_RETURNs, track active/expired bids/asks
- [ ] Build API — `/bids/:cardimg_txid`, `/asks`, `/orderbook`, `/trades`
- [ ] Integrate with CARDID indexer for ownership verification

### Phase 3: Trading Tools
- [ ] CLI: `place-bid`, `place-ask`, `accept-bid`, `match-ask`, `cancel-ask`
- [ ] Automated bid-rolling agent (cron-based, replaces expired bids)
- [ ] Wallet integration (uses existing BSV wallet skill)

### Phase 4: UI
- [ ] Marketplace page in cardimg-viewer
- [ ] Orderbook view (bids/asks per card)
- [ ] Trade history
- [ ] "Buy" and "Sell" buttons on card detail pages
- [ ] Portfolio view (owned cards, open bids/asks)

### Phase 5: Automation
- [ ] Market maker agent — adjusts bid prices based on activity
- [ ] Auto-roll expired bids
- [ ] Notifications — alert operator when bids are filled or expire
- [ ] Analytics — volume, spread, liquidity depth

---

## Technical Notes

### CARDID Verification in Covenants

The hardest part: how does a covenant verify that the CARDID transfer is legitimate?

**Option A — Simplified (v1):**
- Bid covenant only checks that a CARDID OP_RETURN exists in the same tx, signed by the seller's key
- Doesn't verify the full provenance chain — app-layer verification
- Simpler contract, faster to ship

**Option B — Full (v2):**
- Covenant verifies the CARDID is signed by the CARDIMG tx creator's key (original ownership)
- Or verifies a transfer chain where each transfer is signed by the previous owner
- Requires the covenant to parse and verify signatures — complex sCrypt work

**Recommendation:** Start with Option A, upgrade to B later. The app layer (indexer) can do full provenance verification and flag suspicious transfers.

### Fee Structure

- **Ask listing:** operator pays tx fee for the ask covenant
- **Bid placement:** bidder pays tx fee for the bid covenant
- **Trade (bid accept):** seller pays tx fee for the trade tx
- **Trade (ask match):** buyer pays tx fee for the match tx
- **Bid expiry/reclaim:** bidder pays tx fee for reclaim tx
- **No marketplace fees** — pure P2P, no platform cut

### Block Height Introspection

BSV covenants can use `OP_PUSH_TX` to inspect the current block height. This enables:
- Time-locked expiry: `if (currentHeight >= expiryBlock) { allow reclaim }`
- Rolling expiry: multiple bids with staggered expiry blocks

---

## Relationship to Existing Work

- **agent-orderbook** — Covenant-based orderbook work already exists in the ecosystem. MKT2 builds on this pattern.
- **CARDID** — Ownership layer. MKT2 trades CARDID assertions, not raw images.
- **CARDIMG** — Base asset. Never moves, just referenced.
- **bsv-wallet** — Used for funding/signing covenants.
- **cardimg-viewer** — UI integration point for marketplace views.

## Open Questions

- Should MKT2 support batch bids (one bid for any card in a set/category)?
- How to handle partial fills? Or is each trade all-or-nothing?
- Should there be a minimum bid lifetime? (Too short = no real liquidity, too long = capital locked)
- How does the automated bid-rolling agent price bids? (Fixed spread? Last trade price? Oracle?)
- Should bids be cancelable before expiry? (Would need an additional spend path)
- How to handle CARDID disputes in the covenant? (Probably can't — app layer only)
