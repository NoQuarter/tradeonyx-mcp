# TradeOnyx MCP Server

Connect [TradeOnyx](https://tradeonyx.io) to Claude, ChatGPT, Gemini or any
other MCP-capable assistant. Log trades by voice, have the assistant read your
journal back to you, and let it ground its answers in your actual trading
history instead of guessing.

TradeOnyx is a trading journal with pattern detection: it imports trades from
40 brokers and runs 28 detectors over your own history to surface what repeats
in your trading. This repository documents its MCP server. It contains no
product code.

**48 tools · JSON-RPC 2.0 over HTTP · hosted in Germany**

---

## Endpoint

```
https://tradeonyx.io/mcp
```

Transport is HTTP with JSON-RPC 2.0. Authentication is a Bearer token in the
`Authorization` header. The server implements `initialize`, `tools/list` and
`tools/call`, and speaks protocol version `2025-06-18`.

An unauthenticated call answers honestly rather than hanging:

```bash
curl -s -X POST https://tradeonyx.io/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

# {"jsonrpc":"2.0","id":null,"error":{"code":-32001,"message":"Unauthorized"}}
```

## Getting a token

1. Create an account at [tradeonyx.io](https://tradeonyx.io). The account is
   free and asks for no payment details.
2. Open **Settings → Integrations** and start the 7-day connector trial. A free
   account can start it once. After it runs out the connector needs Pro or
   Pro Plus; the journal, the broker imports and the tax export stay free
   either way. A call made without an active trial or plan is refused with a
   message saying so, never with a silent empty result.
3. On the same screen, create an MCP token.
4. Copy it immediately. The raw token is shown once; only a SHA-256 hash is
   stored, so a lost token cannot be recovered, only replaced.
5. Revoke any token from the same screen at any time.

Tokens carry 256 bits of entropy. Treat one like a password: it grants the same
access to your journal that you have.

## Connecting an assistant

Any client that speaks MCP over HTTP works. For a client configured through a
JSON file, the shape is:

```json
{
  "mcpServers": {
    "tradeonyx": {
      "url": "https://tradeonyx.io/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN_HERE"
      }
    }
  }
}
```

Assistants that support remote MCP connectors directly (Claude's Connectors,
for example) can add the endpoint through their own UI, including an OAuth
flow, so no token needs to be pasted by hand.

## What you can do with it

A few things people actually use it for:

- **Log a trade in a sentence.** "I bought 0.5 lots EURUSD at 1.0840, stop at
  1.0810, because the London range broke upward." The assistant fills the
  fields and writes the reasoning into the journal next to the trade.
- **Write the evening review by talking.** The assistant reads back the day's
  trades, asks what you were thinking, and files the answers as a journal entry.
- **Ask what actually happened.** "Which of my setups lost money this month,
  and what did those trades have in common?" — grounded in your own rows, not
  in a guess.
- **Prepare the session.** Pull the daily briefing and the planned playbooks
  before the open.

## Tools

48 tools. 45 need an active connector trial, Pro or Pro Plus; the 3 bulk
aggregations need Pro Plus. The **Plan** column below says which.

Daily tool-call caps, counted per UTC day and shown in the app:

| Plan | Calls per day |
|---|---|
| 7-day trial | 30 |
| Pro | 150 |
| Pro Plus | 2000 |

A call over the cap comes back as a normal tool result with `isError: true`,
naming the cap and when it resets, so an assistant can say what happened
instead of failing silently. It is HTTP 200: a cap is a business rule, not a
malformed request, and the specification's own worked example of a tool
execution error is an upstream rate limit.

### How errors arrive

Two levels, and which one you get tells you whether the fix is in your
arguments or in the data:

* **Wrong arguments** come back as a JSON-RPC error with code `-32602` and
  `data.code = "invalid_arguments"`. That covers an unknown tool, an argument
  name the tool does not accept, a value outside a declared `enum`, and a
  value past a declared `maxLength` / `maxItems` / `maximum`. Everything in
  that list is declared in the `inputSchema` you already hold, so a client
  can correct it without asking.
* **Everything else** comes back as a tool result with `isError: true` and a
  message in `content`: a row that does not exist, a value the database will
  not store, a plan that does not include the tool, the daily cap.

Both are HTTP 200, because the transport requires exactly one JSON object per
request. Values past a declared bound are **refused, not shortened**: a 5000
character note is rejected with the limit named rather than stored as its
first 2000 characters, so what you keep is what you sent.


### Accounts

| Tool | Plan | Description |
|---|---|---|
| `account_list` | Trial / Pro | List the user's trading accounts (id / name / type / is_active / FIFO-vs-LIFO). The is_active flag points at the dashboard's active account, which... |
| `settings_get` | Trial / Pro | Read the account-level limits the discipline checks and the weekly digest enforce: capital, risk per trade, daily and weekly loss limits, monthly goal, whether overnight positions are allowed, timezone. Percentages come back as percentages plus `risk_per_trade_amount` in the account currency. Read-only: these cannot be changed through the connector, and `risk_per_trade` on the strategy tools is a per-strategy annotation that does not move this budget. |

### Attachments

| Tool | Plan | Description |
|---|---|---|
| `attachment_upload_link_create` | Trial / Pro | Mint a single-use, 10-minute-TTL browser-upload link for a screenshot. Use this when the user shares an image blob you cannot encode or URL — commo... |

### Broker sync

| Tool | Plan | Description |
|---|---|---|
| `broker_sync_trigger` | Trial / Pro | Manually trigger a sync on an existing broker connection (Bybit, Alpaca, Binance, etc.). Pulls latest fills, FIFO-pairs Buy/Sell legs, persists rou... |

### Daily briefing

| Tool | Plan | Description |
|---|---|---|
| `briefing_get` | Trial / Pro | Read the cached daily AI briefing for a market category (gbpusd, eurusd, forex, general, crypto). Returns ``{briefing: null}`` if no briefing has b... |

### Economic events

| Tool | Plan | Description |
|---|---|---|
| `econ_analysis_get` | Trial / Pro | Read the cached AI analysis for one economic-calendar event by event_id. Returns ``{analysis: null}`` when no AI run exists for the event. Includes... |
| `econ_analysis_list` | Trial / Pro | List recent cached economic-event AI analyses, newest first. Optional ``symbol`` filters by case-insensitive substring on pairs_affected. Returns u... |

### Journal

| Tool | Plan | Description |
|---|---|---|
| `journal_attachment_list` | Trial / Pro | List every screenshot attached to one of the user's journal entries. Returns id, public URL, caption, mime type and uploaded_at for each attachment. |
| `journal_attachment_upload` | Trial / Pro | Attach a screenshot to one of the user's journal entries. PREFERRED PATHS (in priority order): 1. If the user shared an HTTPS URL of the image (Tra... |
| `journal_entry_get` | Trial / Pro | Read the journal entry for a specific date (or null if none). |
| `journal_entry_list` | Pro Plus | List journal entries within a date range. Pro Plus only. |
| `journal_entry_search` | Trial / Pro | Search the user's own journal entries by words or phrases. Multiple words are ANDed; wrap a phrase in double quotes to match it in order. Searches... |
| `journal_entry_upsert` | Trial / Pro | Create or update the journal entry for a date. Required: date (YYYY-MM-DD). Pass any subset of the six markdown sections — others are left unchange... |
| `journal_planned_playbook_clear` | Trial / Pro | Drop every planned-playbook for a calendar day. Idempotent — returns ``removed_count=0`` if nothing was planned. Returns ``journal_id=null`` if the... |
| `journal_planned_playbook_set` | Trial / Pro | Set the morning-plan playbooks for a calendar day. Replaces the existing set atomically — idempotent on re-call. Creates the JournalEntry if one do... |

### Market data

| Tool | Plan | Description |
|---|---|---|
| `market_heat_get` | Trial / Pro | Read the cached AI market-heat read (hot / neutral / cold) for a category and date. Companion to briefing_get — heat reads the calendar pressure (h... |

### News

| Tool | Plan | Description |
|---|---|---|
| `news_analysis_get` | Trial / Pro | Read the cached AI analysis for one news headline by news_id. The analysis cache is global (shared across users), so you may see analyses written b... |
| `news_analysis_list` | Trial / Pro | List recent cached news analyses, newest first. Optional ``symbol`` filters by case-insensitive substring on the pairs_affected column (e.g. 'EURUS... |

### Patterns

| Tool | Plan | Description |
|---|---|---|
| `pattern_narrative_get` | Trial / Pro | Read the weekly cross-pattern Onyx-Engine AI narrative for the caller. One artefact per (user, ISO-week); the cache rotates automatically on Monday... |
| `pattern_recos_get` | Trial / Pro | Read the weekly cross-detector action-recos for the caller. Sister of pattern_narrative_get — same cache rotation, but the output is a strict list... |

### Playbooks

| Tool | Plan | Description |
|---|---|---|
| `playbook_create` | Trial / Pro | Create a new strategy playbook with entry/exit criteria. Required: name (max 80 chars). description / rules / entry_criteria / exit_criteria are ma... |
| `playbook_delete` | Trial / Pro | Soft-delete a playbook (sets active=false). Historical trade attribution stays intact — the playbook just hides from the active picker. Idempotent. |
| `playbook_list` | Trial / Pro | List the user's strategy playbooks. optional account_id narrows to playbooks visible on that account (global ones + that account's). Omit to get ev... |
| `playbook_update` | Trial / Pro | Modify an existing playbook. Required: playbook_id. Any field omitted is left unchanged. Pass active=false to archive without losing history (same... |

### Statistics

| Tool | Plan | Description |
|---|---|---|
| `stats_summary` | Pro Plus | Aggregate performance stats over a recent period (default 30 days): win rate, expectancy, totals. Pro Plus only. |

### Strategies

| Tool | Plan | Description |
|---|---|---|
| `strategy_create` | Trial / Pro | Create a new named Strategy. The first Strategy a user creates auto-becomes the default (assigned to new trades + imports). All payload fields are... |
| `strategy_delete` | Trial / Pro | Hard-delete a Strategy. Trades tagged with it have their strategy_id reset to NULL (history preserved). If the deleted row was the default, the nex... |
| `strategy_list` | Trial / Pro | List the user's named trading strategies (e.g. 'Forex-Swing', 'Crypto-Scalp'). Each row carries narrative / edge / setups / focus symbols / rules /... |
| `strategy_set_default` | Trial / Pro | Mark a Strategy as the user's default. New trades + imports get auto-tagged with the default. Flips every other Strategy's is_default off. |
| `strategy_update` | Trial / Pro | Modify a named Strategy. Pass strategy_id + any subset of the payload fields. A supplied list (setups, symbols_focus, rules, trading_hours) replaces the stored one, so send every entry you want to keep; omit the field to leave it untouched. This is why the tool is annotated `destructiveHint: true` while `strategy_create` is not. Editing any field invalidates the AI-context caches that referenced t... |

### Tags

| Tool | Plan | Description |
|---|---|---|
| `tag_create` | Trial / Pro | Create a new tag in the user's library. Required: name (max 40 chars). Idempotent on name — returns existing row with created=false if the tag is a... |
| `tag_delete` | Trial / Pro | Hard-delete a tag. Refuses if the tag is still attached to any trades; the error message states how many attachments block the delete. |
| `tag_list` | Trial / Pro | List the user's tags (setup / mistake / psychology / custom). |
| `tag_update` | Trial / Pro | Rename a tag or change its category / color. Required: tag_id. Returns updated_fields so the caller can confirm which keys persisted. Errors with '... |

### Trades

| Tool | Plan | Description |
|---|---|---|
| `trade_ai_analysis_get` | Trial / Pro | Read the cached coach-style AI analysis for one of the user's trades. The analysis is one-shot per trade — once written via the dashboard, every re... |
| `trade_attachment_list` | Trial / Pro | List every screenshot attached to one of the user's trades. Returns id, public URL, caption, mime type and uploaded_at for each attachment. |
| `trade_attachment_upload` | Trial / Pro | Attach a screenshot to one of the user's trades. PREFERRED PATHS (in priority order): 1. If the user shared an HTTPS URL of the image (TradingView... |
| `trade_create` | Trial / Pro | Manually insert a trade (open or closed). Required: symbol, trade_type, volume, open_price. trade_type must be 'BUY' or 'SELL' (uppercase). open_ti... |
| `trade_delete` | Trial / Pro | Delete a trade that was logged manually or through this connector, together with its review, tags, attachments and AI analysis. Broker-imported trades are refused: the next sync would reconcile them against the statement again. Use this to withdraw a trade entered with wrong values, since gross_pl cannot be edited afterwards. |
| `trade_get` | Trial / Pro | Get full details of one trade owned by the user. |
| `trade_list` | Pro Plus | List the user's trades. Filter by date range, symbol, or open/closed status. Returns up to `limit` rows (default 100, max 500) sorted by open_time... |
| `trade_review_update` | Trial / Pro | Upsert the post-trade review for a trade. Required: trade_id. Common stumbles: mood_before/mood_after are INTEGERS 1-5 (NOT enum strings like 'calm... |
| `trade_update` | Trial / Pro | Edit user-mutable fields on an existing trade: sl, tp, comment, external_url, playbook_id, strategy_id, target_rr, mfe_price, mae_price. Also closes an open position: send close_price and close_time together on a trade that is still open and was not broker-imported, and gross_pl is derived from the two prices when the instrument is quoted in the account's currency. Broker-truth fields (gr... |

### Trading rules

| Tool | Plan | Description |
|---|---|---|
| `trading_rule_create` | Trial / Pro | Create a new discipline / trading rule. Required: text (the rule label, max 200 chars). optional account_id confines the rule to one account; omit... |
| `trading_rule_delete` | Trial / Pro | Delete one of the user's trading rules. Also removes the record of every time the rule was broken, because the journal's rule-violation entries cascade with it. To stop following a rule but keep that history, set active=false with trading_rule_update instead. |
| `trading_rule_list` | Trial / Pro | List the user's discipline / trading rules. optional account_id narrows to rules visible on that account (global + that-account). |
| `trading_rule_update` | Trial / Pro | Modify or deactivate a trading rule. Required: rule_id. Pass active=false to soft-delete (rule disappears from picker, history preserved). account_... |

### Weekly digest

| Tool | Plan | Description |
|---|---|---|
| `weekly_digest_get` | Trial / Pro | Read one weekly AI digest for the caller. The digest is the editorial Sunday-evening 600-800 word brief. ``week`` accepts the ISO Monday date (e.g... |

## Plans

| | Free | Free, trial running | Pro | Pro Plus |
|---|---|---|---|---|
| Connector access | no | yes, 7 days, once per account | yes | yes |
| Daily tool calls | — | 30 | 150 | 2000 |
| Logging + journal tools | — | yes | yes | yes |
| Bulk aggregations (`trade_list`, `journal_entry_list`, `stats_summary`) | — | no | no | yes |
| Journal, broker imports and tax export in the web app | yes | yes | yes | yes |

A free account starts the connector trial once, under **Settings →
Integrations**. Before it is started, and after it has run out, the connector
answers every call with an explicit refusal naming what to do; it does not
return empty results. The web app itself stays free throughout.

Prices and current limits are on the [pricing page](https://tradeonyx.io/pricing).

## Privacy and security

- Servers are in Germany and your journal and trades stay in the EU. The AI
  features send the text they analyse to a processor in the United States;
  every processor is named in the [privacy policy](https://tradeonyx.io/datenschutz),
  which is the authoritative list.
- Tokens are stored as SHA-256 hashes; the raw value exists only in your
  clipboard.
- Every tool resolves data through the authenticated user. A token reaches
  exactly the account it belongs to.
- Deleting your TradeOnyx account erases the associated data, including the
  access log for your requests.

Found a security issue? Please report it privately to the address in our
[security.txt](https://tradeonyx.io/.well-known/security.txt)
([hello@tradeonyx.io](mailto:hello@tradeonyx.io)) rather than opening a public
issue.

## Links

- [TradeOnyx](https://tradeonyx.io)
- [Supported brokers](https://tradeonyx.io/broker-support)
- [Comparison with other trading journals](https://tradeonyx.io/trading-journal-comparison)
- [Model Context Protocol](https://modelcontextprotocol.io)

## Disclaimer

TradeOnyx is a documentation and analysis tool. It does not provide investment
advice or investment brokerage, and it does not generate trading signals.
Trading financial instruments can lead to the total loss of the capital
invested.

## Licence

The documentation in this repository is MIT-licensed. TradeOnyx itself is a
commercial product and is not open source.
