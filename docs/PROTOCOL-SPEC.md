# MKT2 Protocol Specification

> **Status:** Draft
> **Date:** 2026-10-15
> **Layer:** Protocol (Layer 1) — trading overlay on CARDIMG/CARDID

## OP_RETURN Format

### ASK — List a card for sale

```
OP_FALSE OP_RETURN "MKT2" "ASK" <cardimg_txid:32B> <ask_price:8B LE> <operator_pubkey:33B>
```

| Field | Size | Description |
|---|---|---|
| Prefix | 4B | `MKT2` |
| Action | 3B | `ASK` |
| cardimg_txid | 32B | The CARDIMG tx being listed |
| ask_price | 8B LE | Price in satoshis |
| operator_pubkey | 33B | Operator's public key (sole seller) |

### BID — Place a bid on a card

```
OP_FALSE OP_RETURN "MKT2" "BID" <cardimg_txid:32B> <bid_amount:8B LE> <bidder_pubkey:33B> <expiry_block:4B LE>
```

| Field | Size | Description |
|---|---|---|
| Prefix | 4B | `MKT2` |
| Action | 3B | `BID` |
| cardimg_txid | 32B | The CARDIMG tx being bid on |
| bid_amount | 8B LE | Bid amount in satoshis |
| bidder_pubkey | 33B | Bidder's public key |
| expiry_block | 4B LE | Block height after which bid can be reclaimed |

### TRADE — Record a completed trade

```
OP_FALSE OP_RETURN "MKT2" "TRADE" <cardimg_txid:32B> <price:8B LE> <buyer_pubkey:33B> <seller_pubkey:33B> <bid_or_ask_txid:32B>
```

| Field | Size | Description |
|---|---|---|
| Prefix | 4B | `MKT2` |
| Action | 5B | `TRADE` |
| cardimg_txid | 32B | The CARDIMG tx that was traded |
| price | 8B LE | Trade price in satoshis |
| buyer_pubkey | 33B | Buyer's public key |
| seller_pubkey | 33B | Seller's public key |
| bid_or_ask_txid | 32B | Reference to the bid/ask covenant that was filled |

### CANCEL — Cancel an ask

```
OP_FALSE OP_RETURN "MKT2" "CANCEL" <ask_txid:32B>
```

## Covenant Spend Paths

### Ask Covenant

```
Spend path 1 — MATCH (buyer fills the ask):
  Require: signature(buyer_pubkey) AND payment(ask_price → operator_addr)
           AND CARDID transfer in same tx (buyer becomes new owner)
  
Spend path 2 — CANCEL (operator delists):
  Require: signature(operator_pubkey)
```

### Bid Covenant

```
Spend path 1 — ACCEPT (seller fills the bid):
  Require: CARDID transfer in same tx (seller transfers ownership to bidder's key)
           AND payment(bid_amount → seller_addr)
  
Spend path 2 — EXPIRY (bidder reclaims after expiry_block):
  Require: signature(bidder_pubkey) AND currentHeight >= expiry_block
```

## Indexing

### What the indexer tracks

- Active asks: `ask_txid → { cardimg_txid, price, operator_pubkey, block, status }`
- Active bids: `bid_txid → { cardimg_txid, amount, bidder_pubkey, expiry_block, block, status }`
- Trades: `trade_txid → { cardimg_txid, price, buyer, seller, ref_txid, block }`
- Cancelled asks: `cancel_txid → { ask_txid, block }`

### Status values

- `active` — covenant UTXO unspent, bid not expired
- `filled` — covenant spent via match/accept path
- `expired` — bid past expiry_block, UTXO reclaimable
- `reclaimed` — bid UTXO reclaimed by bidder after expiry
- `cancelled` — ask cancelled by operator
