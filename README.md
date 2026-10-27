# Crypto.com Exchange API field notes

Practical notes from running a production trading integration against the Crypto.com Exchange API (v1). These are the things that broke for us or surprised us with real orders in a real account, why, and what fixed them. None of them is spelled out clearly in the official docs, and the first one is the most common support question we get on this connection.

Maintained by the team behind [JorgAI](https://jorgai.com), an auto-trading platform that places trades inside members' own brokerage and exchange accounts. The trader-facing guide is [How to automate crypto trading on the Crypto.com Exchange](https://jorgai.com/blog/how-to-automate-crypto-trading-on-crypto-com-exchange). The Kraken equivalent of these notes is [kraken-spot-api-field-notes](https://github.com/Jorg-AI-JorgAI/kraken-spot-api-field-notes).

Nothing here is affiliated with or endorsed by Crypto.com. Verify against the current docs at [exchange-docs.crypto.com](https://exchange-docs.crypto.com/) before relying on any detail; the exchange changes things. Items below were verified October 15 to 21, 2026 against the live API and our own account.

## Quick reference

| Item | Value | Notes |
|------|-------|-------|
| Auth | API key + secret | JSON envelope `{id, method, api_key, params, nonce, sig}` on every private POST |
| Signature | hex(HMAC-SHA256(secret, method + id + api_key + flattened params + nonce)) | Params flattened with sorted keys, key+value concatenated with no separators, recursing into lists and dicts |
| Nonce | Epoch milliseconds | Must be within 60 seconds of Crypto.com's clock |
| Permissions a bot needs | Trading on, Withdrawal off | Set on the Exchange's API Management page, not in the App |
| IP whitelist | On the key form | Effectively required; see trap 4 |
| Instruments | `BTC_USD` style, case-sensitive, underscore | `public/get-instruments` gives `qty_tick_size` and `price_tick_size` |
| Market buy sizing | `notional` (quote amount) | A market SELL takes `quantity`; see trap 3 |
| Numeric fields | Strings | Send `"10"`, not `10` |
| Order types | MARKET, LIMIT, STOP_LOSS, STOP_LIMIT, TAKE_PROFIT, TAKE_PROFIT_LIMIT | Stops carry `ref_price`; `spot_margin` pinned to `SPOT` |
| Bracket / OCO | Not available | See trap 2 |
| Errors | JSON `code` field; `40101`, `40103`, `40104` are key or signature problems | HTTP 401/403 also appear for a rejected key |
| Rate limits | Order create/cancel 15 per 100 ms; order detail 30 per 100 ms; trade history 1 per second; other private calls 3 per 100 ms | A low-frequency REST bot stays far under these |
| Sandbox | None for retail | UAT is institutional and invitation only; we proved the integration with a tiny live order |

## Trap 1: the key sees the Exchange balance, not the App balance

Crypto.com has two products behind one login: the App wallet and the Exchange. An Exchange API key reads the Exchange balance only. The most common support question we get on this connection is "it shows zero balance", and the answer is always the same: the coins are in the App. The user transfers them to the Exchange (free, instant, inside the Crypto.com app) and the balance appears.

**What we do:** on connect, if the balance read succeeds but every balance is zero, say exactly that on the screen ("your key is working; your Exchange balance is empty; transfer from the App to the Exchange") rather than a generic "no funds".

## Trap 2: there is no bracket order, so every exit lives on your side

The Exchange has stop and take-profit order types but no one-cancels-the-other pair. If you place both a stop and a target as resting orders, both are live, and when one fills the other remains, now an unhedged order on a position you no longer hold.

**What we do:** place the entry, record the stop and target in our own system, and run the exits from a monitor that evaluates every open position on a short loop, around the clock, because the market does not close. We mark every Crypto.com entry `bracket_failed=True` internally so the rest of the platform knows the exchange holds no exits. The honest consequence: if your connection drops, your exits stop with it. Design the connection monitoring and the position sizing with that in mind.

## Trap 3: a dollar-sized market buy fills slightly under the dollar amount

A market buy takes `notional` (a quote-currency amount, `"10"` for ten dollars) and fills to the instrument's quantity increment, so a ten-dollar order can fill at $9.97 worth. A market sell takes `quantity`, not `notional`. Stops carry a `ref_price`. Every numeric field is a string; send a float where a string is expected and the error message does not say so.

**What we do:** round quantities down to `qty_tick_size` before an order leaves, treat the tick as the minimum order size, read `cumulative_quantity` from the order after the fill rather than trusting the requested amount, and leave a small cushion when sizing a sell from a prior buy's notional. See [`examples/cryptocom_sign_and_market_buy.py`](examples/cryptocom_sign_and_market_buy.py).

## Trap 4: the IP whitelist is not optional in practice

The key form offers an IP whitelist. Without the caller's address on it, private calls fail in ways that look like a bad key. If you run a hosted service, publish your outbound addresses and tell users to add them when they create the key; if you run from a laptop, your home IP changes and the key stops working one morning for no visible reason.

## Smaller things worth knowing

- **The signature covers a flattened params string**, not the JSON. Get the flattening wrong (unsorted keys, a separator, `None` rendered as anything but `null`) and every call returns a signature error that looks like a bad secret.
- **Every call needs a timeout.** Nothing on the trade path should be able to wait forever.
- **Reserved quantity** comes back on balances (`reserved_qty`); subtract it to get what is actually spendable.
- **Market data for scanning** does not have to come from the Exchange. We size market buys on its public ticker and scan on a separate feed.
- **Rotate keys** by creating the new key (with the whitelist) first, confirming a balance read, then deleting the old one, so open positions never sit without their exits.

## Who we are

JorgAI runs rules-based automated trading in members' own brokerage and exchange accounts (Schwab, Alpaca, Tradier, tastytrade, Kraken and the Crypto.com Exchange). Related notes: [schwab-trader-api-field-notes](https://github.com/Jorg-AI-JorgAI/schwab-trader-api-field-notes) and [kraken-spot-api-field-notes](https://github.com/Jorg-AI-JorgAI/kraken-spot-api-field-notes). Corrections welcome as issues or pull requests.

## License

MIT. See [LICENSE](LICENSE).
