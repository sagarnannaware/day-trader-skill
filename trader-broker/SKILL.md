---
name: trader-broker
description: >-
  Order Executor. Sends approved orders to your broker, paper trading by
  default, never without your yes. Set up for moomoo's official MCP
  connector: shows each order as a plain ticket (a bracket shows its
  stop-loss, take-profit and the max loss at the stop) plus the exact moomoo
  calls it becomes (trading_order_place live, sim_trade_input_order on paper;
  moomoo has no bracket, so the stop goes in as its own STOP order the moment
  the entry fills and the target is watched by trader-alerts), records your
  yes word for word, then Claude's own Allow once / Deny card asks again.
  Builds the sell ticket that closes a day trade by your flat-by time,
  records fills, and logs each trade's max loss so trader-risk can hold the
  daily limit. Also works with Alpaca or Interactive Brokers. Use when the
  user says "place the order", "buy it", "execute", "send it to my broker",
  "close my position", "flatten", "get me out", "set up paper trading", or
  after /trader-risk passes.
argument-hint: "[order number]"
---

# trader-broker

The Order Executor is the only skill in this pack that can lead to a real
order, so it is the strictest. An order reaches the market only after three
separate gates, every time, for every order:

1. **trader-risk passed this exact order today** (matched by fingerprint).
2. **The user typed an explicit yes for this one order** in the chat.
3. **The user clicked Allow once** on Claude's own permission card, which
   appears because the order tool is set to "Needs approval".

Two yeses from the user, one PASS from the rules. No yes, no order.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/ticket.py" show                     # every ticket, READY or not
python3 "${CLAUDE_SKILL_DIR}/ticket.py" approve --index 1 --reply "yes"
python3 "${CLAUDE_SKILL_DIR}/ticket.py" result --index 1 --status placed --broker-id ID --filled-qty 8 --filled-price 364.20
python3 "${CLAUDE_SKILL_DIR}/ticket.py" fill --symbol TSLA --side sell --qty 8 --price 370.04
python3 "${CLAUDE_SKILL_DIR}/ticket.py" flatten --symbol TSLA --qty 8
python3 "${CLAUDE_SKILL_DIR}/ticket.py" log
```

`ticket.py` never talks to a broker. It renders tickets, checks that a reply
is a real yes for that order, builds the flatten ticket, and keeps the log in
`trader/state/broker.json`. `--now "2026-10-19 10:15"` pins the clock (New York time).

## What it reads and writes

| file | what |
| --- | --- |
| `trader/orders.json` | the proposed orders (trader-budget writes it; `flatten` replaces it) |
| `trader/config.json` | `"broker": "moomoo"` and `"paper"`; set them in Setup step 3 |
| `trader/state/risk.json` | trader-risk's verdicts; only a PASS from today is READY |
| `trader/state/broker.json` | every yes, no, placed, failed and filled, with each entry's max loss |
| `trader/portfolio.json` | cash and shares, updated by `fill` and by `result --filled-qty` |
| `trader/plan.json` | the flat-by time for `flatten` |

`trader/orders.json` keeps the pack's own order format (trader-budget and
trader-risk read and write it). `ticket.py` turns each order into moomoo's
calls; nothing in the format changes.

## Setup: moomoo (official MCP connector)

1. Add the moomoo connector to Claude (Customize > Connectors) and sign in to
   moomoo there, never in the chat. Each moomoo login is its own connector
   ("MooMoo – <name>"); use only the one for the account the user names, and
   say whose account it is on every ticket.
2. **Set the tool permissions** (Customize > Connectors > moomoo > tool permissions):
   - `trading_order_place`, `trading_order_replace`, `trading_order_cancel`,
     `trading_order_confirm`, `sim_trade_input_order`, `sim_trade_modify_order`,
     `sim_trade_cancel_order`: **Needs approval**. Never Always allow.
   - crypto order tools (`crypto_account_create_order`, `crypto_account_modify_order`,
     `crypto_account_cancel_order`): **Blocked** unless the user has a reason and says so.
   - read-only tools (`account_authorized_trd_accs`, `account_funds`, `account_positions`,
     `account_orders_active`, `account_order_fills_today`, `quote_stock_quote`,
     `sim_trade_account_list`, `sim_trade_position_list`): Always allow is fine.
3. In `trader/config.json` set `"broker": "moomoo"`. Keep `"paper": true` until
   the user has watched it work on moomoo's paper account. Going live
   (`"paper": false`) is the user's own decision; never switch it yourself.
4. Check it: ask for `account_authorized_trd_accs` (read only). The live
   account is the one whose `enable_market` includes 2 (US) and `acc_type` is
   margin; the paper account comes from `sim_trade_account_list` (US). Pass
   `acc_id` as a string, all digits. Never store it in a file.

In Claude Code instead of claude.ai, keep the order tools on ask in
`.claude/settings.json`, for example
`"permissions": {"ask": ["mcp__MooMoo_-_Sagar__trading_order_place"]}`, and
never put an order tool in `allow`.

**How a bracket goes to moomoo.** moomoo's connector has no bracket and no
OCO, and a resting take-profit sell would hold the shares the stop needs. So
the one approved bracket becomes:

| part | live (`trading_order_place`) | paper (`sim_trade_input_order`) |
| --- | --- | --- |
| entry | `MARKET` (or `LIMIT`) BUY, `DAY`, session `RTH`, remark `INTRADAY:<setup>` | `order_type` 3 market (1 limit), regular hours |
| stop-loss | as soon as the entry fills: `STOP` SELL the filled shares, `aux_price` = the stop, **`GTC`**, `RTH`, remark `INTRADAY:STOP` | not offered on paper (limit and market only): trader-alerts watches it |
| take-profit | watched by trader-alerts; when it rings, cancel the stop, then sell | watched by trader-alerts |

The stop is GTC so a missed flatten is still protected overnight. The user's
yes to the ticket covers this set for that one trade, and the ticket says so.
Any other order needs its own yes. Read every order back with
`account_orders_active` (a resting STOP shows `WAITING_SUBMIT`); a FAILED or
missing stop means the position is unprotected: say so first.

## Other brokers

`"broker": "alpaca"` or `"broker": "ibkr"` in the config gives the original
pack's tickets: Alpaca's `place_stock_order` with a real bracket (setup in
`claude_desktop_config.example.json`, paper keys in env vars, never in the
chat), or Interactive Brokers' connector, where the final yes happens in IBKR.

## Steps: the entry

1. `ticket.py show`. Only READY tickets go to the user. A ticket that is NOT
   READY or BLOCKED goes back to `/trader-risk`.
2. Show ONE ticket and ask: "Place this order? Reply yes or no." One order
   per question. Never offer "yes to all".
3. Run `ticket.py approve --index N --reply "<the user's reply, word for word>"`.
   If it prints "Not recorded", the reply was not an explicit yes for this
   order (for example "sure", "yes to all", or a yes that names a different
   stock or amount). Ask again. Never turn a reply into a yes yourself.
4. Only after "Recorded your yes": fill in `acc_id` (and `market` on paper)
   live from moomoo, then call the entry tool with exactly the arguments the
   ticket printed. Claude's card appears: Allow once or Deny. If the response
   has `need_order_confirm: true`, call `trading_order_confirm` with its
   `confirm_id` (moomoo's own check, not the user's yes).
5. When the entry fills (`account_order_fills_today` or `account_positions`),
   place the stop-loss call from the ticket for the shares that filled, and
   read it back.
6. Record what happened: `ticket.py result --index N --status placed
   --broker-id <moomoo order_id> --filled-qty Q --filled-price P`, or
   `--status denied`, or `--status failed` with `--note`. A failed order is
   not retried without a new check and a new yes. The log keeps the entry's
   max loss for trader-risk's daily limit.

## Steps: while it is open, and out by the close

- trader-alerts watches the stop, the target and the flat-by time
  (`alerts.py add --from-plan`). Live, the STOP at moomoo does the stopping;
  on paper the alert is the stop.
- When the stop fills, record it:
  `ticket.py fill --symbol TSLA --side sell --qty 8 --price 358.32`.
- When the target alert rings: show the user the sell (cancel the stop, then
  a `LIMIT` sell at the target or a `MARKET` sell), get their yes, place it,
  record the fill.
- By `flat_by` (15:55 New York), anything still open gets closed:
  1. Ask moomoo how many shares are held: `account_positions` (read only).
  2. `ticket.py flatten --symbol TSLA --qty 8` writes a market sell for those
     shares to `trader/orders.json`.
  3. `/trader-risk`. A sell that only closes what the user holds is never
     blocked by the daily limits or the entry window.
  4. `ticket.py show`, then the user's yes for that one order, as above.
  5. The INTRADAY:STOP still waits on those shares: find it with
     `account_orders_active` (US), `trading_order_cancel` it (`order_id`,
     `exchange`; Needs approval), then place the sell, then `result`.
- If the user is away at the close, the GTC stop still guards the shares
  overnight. Say so at the next session and let the user choose: sell at the
  open, or keep it under their own rules.

## Example: the reel

```
ORDER 1 of 1, LIVE ACCOUNT, REAL MONEY, moomoo
  BUY 8 TSLA, market, good for today
  stop-loss 358.32
  take-profit 370.04
  max loss about $46.88 if the stop fills at 358.32 (it can fill worse in a fast market)
  why: opening-range breakout on TSLA: 8 shares, bound by your $50 max loss per trade (8 x 5.86 = $46.88 at the stop)
  READY: trader-risk PASS, waiting for your yes
  entry: Claude will call trading_order_place with:
    {"acc_id": "<acc_id: the US margin account from account_authorized_trd_accs>", "code": "US.TSLA",
     "side": "BUY", "order_type": "MARKET", "qty": "8", "time_in_force": "DAY", "session": "RTH",
     "remark": "INTRADAY:orb"}
  stop-loss: Claude will call trading_order_place as soon as the entry fills, for the shares that filled with:
    {"acc_id": "<...>", "code": "US.TSLA", "side": "SELL", "order_type": "STOP", "qty": "8",
     "aux_price": "358.32", "time_in_force": "GTC", "session": "RTH", "remark": "INTRADAY:STOP"}
  moomoo has no bracket: the stop-loss goes in as its own STOP order, good until canceled, ...
  The take-profit 370.04 is watched by trader-alerts, not left at moomoo ...
```

The user replies "yes". Claude records it, calls `trading_order_place` for
the entry, the card reads "Claude wants to use trading_order_place", Allow
once sends it. When the 8 shares fill, the STOP goes in the same way and is
read back. The `why` and the plan stay in the file; they are never sent.

## What it never does

- Never places an order without all three gates, and never two trades on one yes.
- Never places orders from a scheduled or background task: nobody is there to say yes.
- Never asks for, reads, stores or repeats passwords, keys or trade PINs.
- Never switches `"paper"` to live. The user does that themselves.
- Never uses another person's moomoo connector for this account.
- Never calls crypto order tools or cancels orders it did not create on its own.
- Never promises a fill price: a market order and a stop fill at whatever the market gives.

## Safety

- Education and tooling, not financial advice. A PASS is a rules check, not a recommendation.
- Paper trading by default (`"paper": true`). Going live is the user's own decision, in their own config.
- Nothing in this pack places an order except trader-broker, and only when all three hold: trader-risk
  passed that exact order today, the user typed an explicit yes for that one order in the chat, and the
  user clicked Allow once on Claude's own card because the order tool is set to "Needs approval"
  (Customize > Connectors > tool permissions). Never set an order tool to "Always allow".
- No options, margin, short selling or leverage unless the config allows it and the user confirms.
- Broker logins live in the connector, never in the chat.
