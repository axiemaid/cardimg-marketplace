# CARDIMG Marketplace — Architecture

> **Date:** 2026-10-15

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│                    cardimg-viewer                        │
│  ┌──────────┐  ┌──────────┐  ┌───────────────────────┐ │
│  │ Gallery  │  │ Card Set │  │   Marketplace UI     │ │
│  │ (existing)│  │ (existing)│  │ (orderbook, trade)  │ │
│  └─────┬────┘  └─────┬────┘  └──────────┬────────────┘ │
└────────┼─────────────┼──────────────────┼───────────────┘
         │             │                  │
         ▼             ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                   cardimg-api                           │
│  /img  /thumb  /card  /sets  /stats  +  /bids /asks    │
│                                          /orderbook     │
│                                          /trades        │
└────────┬────────────────────────────────┬──────────────┘
         │                                │
         ▼                                ▼
┌─────────────────┐        ┌──────────────────────────┐
│  cardimg-indexer│        │   mkt2-indexer (new)      │
│  (existing)     │        │  scans MKT2 OP_RETURNs    │
│  CARDIMG + CARDSET│      │  tracks bids/asks/trades  │
└────────┬────────┘        └──────────┬────────────────┘
         │                            │
         ▼                            ▼
┌─────────────────┐        ┌──────────────────────────┐
│  cardimg-data   │        │   mkt2-data (new)         │
│  ledger.json    │        │  asks.json                │
│  mempool.json   │        │  bids.json                │
│  images/        │        │  trades.json              │
│  thumbs/        │        │  cancelled.json           │
└─────────────────┘        └──────────────────────────┘

         ┌──────────────────────────────────┐
         │     CARDID (ownership layer)     │
         │  cardid-indexer (new)            │
         │  scans CARDID OP_RETURNs         │
         │  builds provenance chains         │
         │  → /owner/:txid  /provenance     │
         └──────────────────────────────────┘
```

## Components

### New Components

| Component | Path | Description |
|---|---|---|
| mkt2-indexer | `~/.openclaw/cardimg-marketplace/` | Scans for MKT2 OP_RETURNs, tracks bid/ask/trade state |
| mkt2-api | (extends cardimg-api) | `/bids`, `/asks`, `/orderbook`, `/trades` endpoints |
| bid-rolling-agent | `~/.openclaw/cardimg-marketplace/agent/` | Automated agent that re-places expired bids |
| cardid-indexer | `~/.openclaw/cardimg-marketplace/cardid/` | Scans for CARDID OP_RETURNs, builds provenance chains |
| marketplace UI | (extends cardimg-viewer) | Orderbook view, buy/sell buttons, portfolio |

### Existing Components (Extended)

| Component | Change |
|---|---|
| cardimg-api | Add MKT2 + CARDID endpoints |
| cardimg-viewer | Add marketplace tab, orderbook view, trade buttons |
| bsv-wallet | Used for covenant creation and signing |

## Data Flow

### Placing a Bid
```
User → CLI/API → bsv-wallet → create bid covenant TX
  → broadcast to BSV network
  → mkt2-indexer catches it → bids.json
  → cardimg-api serves it → viewer shows in orderbook
```

### Accepting a Bid (Selling)
```
Seller → CLI/API → bsv-wallet → create trade TX
  ├── spends bid covenant (Accept path)
  ├── spends seller's CARDID UTXO (ownership transfer)
  ├── creates new CARDID OP_RETURN (to buyer's key)
  └── pays bid_amount to seller
  → broadcast → miners confirm
  → mkt2-indexer catches TRADE → trades.json
  → bid status → filled
  → cardid-indexer catches new CARDID → updates provenance
```

### Bid Expiry & Roll
```
Block height passes expiry_block
  → bid-rolling-agent detects expired bids
  → bidder (or agent) reclaims bid UTXO
  → agent re-places bid at adjusted price (if operator)
  → mkt2-indexer updates: old bid → reclaimed, new bid → active
```

## Technology Stack

- **Covenant contracts:** sCrypt (scrypt-ts) — TypeScript-based sScript
- **Indexer:** JungleBus subscription (like cardimg-indexer)
- **API:** Node.js Express (extends cardimg-api)
- **CLI:** CommonJS scripts (like other cardimg tools)
- **UI:** Vanilla JS (extends cardimg-viewer)
- **Wallet:** bsv-wallet skill (existing)
- **Automation:** OpenClaw cron agent for bid rolling

## Deployment

- All services via PM2 (like existing cardimg services)
- Indexer on its own port (TBD)
- API endpoints added to existing cardimg-api (port 3014)
- UI added to existing cardimg-viewer (port 3015, Vercel)
- Bid-rolling agent via OpenClaw cron
