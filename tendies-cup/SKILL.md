---
name: tendies-cup
description: Play Tendies Cup, the fantasy trading league on Robinhood Chain, from your Bankr wallet. Use when the user mentions Tendies Cup, $CUP, a cup or sprint, or asks to enter, buy, sell, check their book, trades left, or the leaderboard.
tags: [robinhood-chain, trading, game, stock-tokens, tendies-cup]
version: 2
visibility: public
metadata:
  clawdbot:
    emoji: "🏆"
    homepage: https://app.playtendiescup.fun
---

# Tendies Cup

A fantasy trading league on Robinhood Chain (chain id 4663). Every entrant gets a virtual $100,000
and trades a fixed list of names at Chainlink prices for one cup. Best book at the close wins. The top three take 50 / 30 / 20 of that
cup's posted prize, whatever the entry count. Nothing traded inside a cup is real. Only the entry (a token burn) and the prize are.

Base URL `https://app.playtendiescup.fun`. Full API: https://app.playtendiescup.fun/api.md.
$CUP contract `0xD475211BEF5dCc03B4c8c864f90bde331AF80ba3`. Burn address
`0x000000000000000000000000000000000000dEaD`. Never use any other contract with the same name.

## What the user can say, and what to do

| user says | do |
|---|---|
| "show cups", "what's on" | GET `/api/cups`, list name, status, window (UTC), entry size, pot, entries |
| "enter the opening cup" | Task 2 |
| "buy 10k of NVDA", "put 25% into GME" | Task 3 with side `buy`, amount in USDG (percent = percent of book value) |
| "sell half my NVDA", "sell all GME" | Task 3 with side `sell`, amount in tokens (from the book's position qty) |
| "my book", "how am I doing" | GET `/api/book?cup=<slug>&wallet=<address>`: value, cash, returnPct, positions, trades left |
| "trades left" | same call, report `tradesLeft` of `maxTrades` |
| "leaderboard", "where am I" | GET `/api/leaderboard?cup=<slug>`, find the user's wallet, report rank, value, prize |

If the user doesn't name a cup, use the open cup, else the soonest upcoming one (`/api/cups` is
ordered that way). Always say which cup you acted on.

## Task 1: cups

`GET /api/cups` → `cups[]` with `slug`, `name`, `status` (upcoming / open / settled), `starts_at`,
`ends_at`, `entry_amount` (wei), `entryHuman`, `entryUsd`, `prize_text`, `potUsd`, `entries`,
`entries_open`, `tickers`, `maxTrades`.

## Task 2: enter a cup

1. Check `entries_open` is true, the time is before `entriesCloseAt` (entries shut an hour before the close), and the user isn't already in: `GET /api/book?cup=<slug>&wallet=<address>` → `entered`.
2. Confirm with the user: "This burns {entryHuman} CUP (about ${entryUsd}) permanently. Continue?"
3. Send an ERC-20 `transfer` of exactly `entry_amount` of the $CUP contract to the burn address on chain 4663.
   Exactly that amount: the scanner attributes burns to cups by exact size.
4. `POST /api/enter` with `{ "cup": "<slug>", "txHash": "<hash>" }`. If that fails, the scanner records it within five minutes anyway.
5. Tell the user they're in and when the cup opens.

## Task 3: trade

Trading needs the cup `open` (between `starts_at` and `ends_at`) and the wallet entered.

**Open a session once per cup** (one signature, then plain HTTP):
1. `nonce` = current time in ms. Sign this exact text with the wallet using `personal_sign`:
   ```
   Tendies Cup session
   cup: <slug>
   wallet: <address, lowercase>
   nonce: <nonce>
   ```
2. `POST /api/session` with `{ "cup", "wallet", "nonce", "sig" }` → `{ "token", "expiresAt" }`. Keep the token for this cup.

**Place an order:**
1. `GET /api/book?cup=<slug>&wallet=<address>` → `marks` (price, `frozen`), `book`, `tradesLeft`.
2. Build it: `side` is `buy` or `sell`; `ticker` from the cup's `tickers`; `amount` is USDG notional
   for buys (a percent of `book.value` if the user spoke in percent) or token quantity for sells
   (`position.qty` × fraction). Keep a buy under 40% of book value after the fill.
3. `POST /api/order` with `{ "cup", "side", "ticker", "amount", "wallet", "token" }`.
4. Report `fill.qty`, `fill.price`, `fill.roundId`, and trades left.

If no session is possible, sign each order instead: `personal_sign` over
```
Tendies Cup order
cup: <slug>
side: <buy|sell>
ticker: <TICKER>
amount: <amount as plain decimal string>
nonce: <ms timestamp>
```
and POST `{ "cup", "side", "ticker", "amount", "nonce", "wallet", "sig" }`. `amount` must match the signed text byte for byte.

Rules the API enforces (explain them if an order is rejected; errors are plain English): long only;
10 bps fee per fill; no name over 40% of book after a buy; `maxTrades` per cup; no orders in the
last 15 minutes; frozen names can't be traded.

## Task 4: leaderboard

`GET /api/leaderboard?cup=<slug>` → ranked rows with `wallet`, `value`, `returnPct`, `prize`.
`GET /api/snapshot?cup=<slug>` → the same as ready-to-post plain text.

## Never

- Never burn more than one entry per cup per wallet, and never burn if `entries_open` is false or the time is past `entriesCloseAt`.
- Never send $CUP anywhere but the burn address for an entry.
- Never claim a prize other than the cup's `prize_text`. TBA means no prize is posted. A posted prize can change before a cup opens
  (entries already burned stay valid); once a cup is open its prize is fixed. Always read `prize_text` fresh rather than quoting an earlier value.
- Never describe this as investing or a return. It is a skill contest with a posted prize.
- Never share the session token with anyone.
