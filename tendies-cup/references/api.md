# Tendies Cup public API

Base: `https://app.playtendiescup.fun`. JSON unless noted. Reads need no auth. Orders are
authenticated by a session token or a wallet signature. Chain: Robinhood Chain, id 4663.

## Constants

| thing | value |
|---|---|
| $CUP token | `0xD475211BEF5dCc03B4c8c864f90bde331AF80ba3` (18 decimals) |
| burn address | `0x000000000000000000000000000000000000dEaD` |
| virtual bankroll | 100,000 |
| fee | 10 bps per fill |
| position cap | 40% of book value after a buy |
| trading lock | last 15 minutes of a cup |

## GET /api/cups

```json
{ "cups": [ { "slug": "opening-cup", "name": "Opening Cup", "status": "upcoming",
  "starts_at": "2026-09-14T00:00:00.000Z", "ends_at": "2026-09-18T20:00:00.000Z",
  "entry_amount": "2500000000000000000000000", "entryHuman": "2,500,000", "entryUsd": 2.6,
  "prize_text": "5 SPY", "potUsd": 3808.1, "prize_wallet": "0x21d4…11eb", "entries_open": true,
  "entries": 15, "tickers": ["NVDA","AAPL", "..."], "maxTrades": 20, "blurb": "…" } ] }
```
Ordered: open cups first, then upcoming by start, then settled.

## GET /api/cup/{slug}

One cup, same shape, plus `token` and `burnAddress`.

## POST /api/enter

Body `{ "cup": "opening-cup", "txHash": "0x…" }`. Verifies the receipt contains a $CUP transfer to
the burn address of at least the cup's entry amount, made before the cup closes, and that entries
are open. Returns `{ "ok": true, "wallet", "txHash", "alreadyEntered" }` or `{ "error" }` (400).

## GET /api/book?cup={slug}&wallet={address}

```json
{ "marks": { "NVDA": { "ticker": "NVDA", "price": 223.48, "roundId": "184…", "updatedAt": "…", "paused": false, "frozen": false } },
  "entered": true, "maxTrades": 20, "tradesLeft": 17,
  "book": { "cash": 94493.5, "value": 99993.5, "trades": 3, "fees": 6.5, "returnPct": -0.0065,
            "positions": [ { "ticker": "NVDA", "qty": 22.37, "avg": 223.48, "value": 5000, "weight": 0.05, "pnlPct": 0 } ] },
  "orders": [ { "id": 5, "ticker": "NVDA", "side": "buy", "qty": 22.37, "price": 223.48, "round_id": "…", "notional": 5000, "fee": 5, "created_at": "…" } ] }
```
Without a wallet, `marks` only.

## POST /api/session

Body `{ "cup", "wallet", "nonce", "sig" }` where `sig` is an EIP-191 `personal_sign` over exactly:
```
Tendies Cup session
cup: opening-cup
wallet: 0x…   (lowercase)
nonce: 1789000000000
```
Lines joined with `\n`, no trailing newline. `nonce` is a ms timestamp within five minutes of server
time. The wallet must have entered the cup. Returns `{ "token", "expiresAt" }`; the token is valid
until the cup closes and replaces any earlier session for that wallet and cup. Smart-account
signatures (EIP-1271 / ERC-6492) are accepted.

## POST /api/order

With a session: `{ "cup", "side", "ticker", "amount", "wallet", "token" }`.

With a signature: `{ "cup", "side", "ticker", "amount", "nonce", "wallet", "sig" }`, `sig` over
```
Tendies Cup order
cup: opening-cup
side: buy
ticker: NVDA
amount: 5000
nonce: 1789000000000
```
`amount` is a plain decimal string: USDG notional for buys, token quantity for sells, and must match
the signed text byte for byte. A nonce may not be reused.

Success: `{ "ok": true, "fill": { "id", "ticker", "side", "qty", "price", "roundId", "notional", "fee" } }`.
Failure: `{ "error" }` with 400 (rule), 401 (auth), 409 (nonce reuse), 429 (rate limit).
Rate limits: 12 orders per minute per wallet, 30 per minute per IP.

## GET /api/leaderboard?cup={slug}

`{ "cup", "settled", "started", "asOf", "rows": [ { "rank", "wallet", "value", "returnPct", "trades", "top", "prize" } ] }`.
`prize` is null before the cup opens and for cups with no posted prize.

## GET /api/snapshot?cup={slug}

Plain-text leaderboard, ready to post.

## GET /api/health

`{ "ok": true|false, "problems": [] }`, 503 when the cron is stale or a feed is frozen.
