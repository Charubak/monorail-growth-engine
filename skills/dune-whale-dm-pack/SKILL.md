---
name: dune-whale-dm-pack
description: >-
  Build a whale/top-trader outreach list from onchain DEX data, end to end - Dune
  query through the user's logged-in Chrome (no API key), wallet extraction,
  wallet-to-identity mapping (ENS, X handles, Telegram), bot flagging, and a
  DM-ready xlsx pack with per-wallet profile links. Use this whenever Charubak asks
  for top swappers or traders of any token pair on any chain, a whale list, wallet
  segmentation, mapping wallets to profiles or socials, onchain acquisition targets,
  a DM or outreach pack from onchain data, or says things like "who trades X",
  "top N wallets", "find the whales", "map these wallets", or "run the wallet intel".
  Trigger even if he only names a pair and a chain. Defaults to MON/USDC on Monad,
  last 90 days, top 1000.
---

# Dune Whale DM Pack

Turn a token pair into a ranked, identity-enriched, DM-ready wallet list. This skill encodes a full working run from 2026-07-15 (MON/USDC on Monad), including every workaround discovered. Follow the sequence; the traps are real and the detours around them are tested.

## Parameters (confirm or default, do not block on them)

- PAIR: token symbols, e.g. WMON/MON vs USDC. Include both wrapped and native symbols for gas tokens.
- CHAIN: dune `blockchain` value, e.g. 'monad', 'ethereum', 'base'.
- WINDOW: lookback, default `interval '90' day`.
- N: wallet count, default 1000.

State the defaults you assumed in one line and keep moving - the user prefers fast execution with corrections over upfront questions.

## Architecture in one paragraph

Dune is driven through the user's logged-in Chrome tab (Claude-in-Chrome tools), because there is no Dune MCP and the API key must not be handled in plain text. Results leave the page through a marker-delimited `<pre>` block read by `get_page_text`, because every other channel truncates. Large dumps land in host-side files that only the Read/Grep/Write tools can see (subagent bash cannot). Identity enrichment runs as fetch loops inside a neutral browser tab (example.com), because the shell sandbox has no network to crypto APIs and Dune's CSP blocks in-page fetches. The final workbook is built by the bundled script and verified with the xlsx skill's recalc.

## Step 0 - Task list and tabs

Create a task list (query, extract, enrich, deliver, verify). Get browser context with `tabs_context_mcp` (createIfEmpty). Use one tab for Dune, create a second later for enrichment. Never navigate away from an unsaved Dune query - a "Leave site?" dialog will block you.

## Step 1 - Author and run the Dune query

Navigate to dune.com/queries. Read `references/dune-browser-playbook.md` NOW - it covers the editor-injection trap (typed text lands in the AI prompt box, not the SQL editor), the exact CodeMirror-6 injection snippet, and the packed-row SQL pattern. Do not improvise here.

Use the packed-row query from the playbook from the start. Do NOT run a plain `limit N` query and try to paginate the results table: pages beyond ~2 trigger a server roundtrip of 30-60s each and stall - a 40-page scrape takes an hour and breaks. The packed pattern returns N wallets in N/25 rows (2 pages max) with full untruncated cell text.

Save the query with a descriptive name (Save button, then name dialog). Note the query URL - it goes in the deliverable so the user can rerun it.

## Step 2 - Extract the rows out of the browser

Read the "Exporting data out of the page" section of the playbook. Summary of the hard limits you are working around:

- `javascript_tool` output truncates near 1 KB. Store data in `window` vars; return only counts.
- `get_page_text` returns inline below roughly 25 K chars; above that it saves to a host file; but the saved file itself truncates at exactly 50,000 chars. So every dump must keep TOTAL page text under 50 K: remove `<table>` elements from the DOM first, and cap each `<pre>` dump at ~330 CSV rows (~47 K chars).
- Wrap dumps in `CSVSTART` / `CSVEND` markers inside a `<pre id="__export">` PREPENDED INSIDE `<main>` (get_page_text reads `<main>`, not `<body>`).

Extract rows from the host dump files with the Grep tool (host filesystem), pattern `0x[0-9a-fA-F]{40},[^"\\]+` - the comma immediately after the address separates real CSV rows from page junk. Spawn ONE general-purpose subagent to do the file extraction and CSV assembly so the raw data never floods your context; give it the exact host paths, the regex, and the dual-mapped outputs folder path. Tell it explicitly that its Bash cannot see /var/folders host paths and it must use Read/Grep/Write for those.

Verify the count. If rows are missing, they are at the tail of a dump that crossed 50 K - re-dump just that slice smaller. Do not ship a short file silently.

Data cleaning the assembler must do: volumes arrive in scientific notation (`1.664074E4`) from `round()` + varchar cast - convert to plain 2-decimal numbers; timestamps carry a `.000 UTC` suffix - strip it.

## Step 3 - Identity enrichment (browser-side)

Read `references/enrichment-apis.md` for the live/dead API table and the CORS/CSP findings. Summary of what works as of 2026-07:

- ensdata.net: WORKS from a neutral tab. Returns ENS name plus twitter/telegram/github text records. This is the main identity source.
- Warpcast lookup API: DEAD for anonymous use (401). Skip unless a Neynar key is provided.
- DeBank internal API: signature-gated. Do not fight it. DeBank value comes as profile LINKS in the deliverable.
- Sandbox shell: no network to any of these. Everything runs in the browser.

Mechanics: create a tab, navigate to example.com (neutral origin, no CSP), inject the wallet list in 250-address chunks (input to javascript_tool can be large; output cannot), then start a fire-and-forget async loop (concurrency 6) writing hits to `window.__ens`, and poll a counter every ~20 s. ~4 min per 1000 wallets. Delegate this whole step to a subagent with the playbook's loop snippet if context is getting heavy; it needs the Chrome tools plus Bash.

Expect a 3-10% identity hit rate on young chains. That is normal - say so rather than overpromising. The unresolved majority is still reachable wallet-natively (DeBank Hi, Blockscan Chat links in the pack).

Compute locally (no network): swaps_per_active_day, avg_trade_usd, likely_bot = yes when swaps_per_active_day > 150 or avg_trade_usd > 100000. Present it as a heuristic, not a verdict.

## Step 4 - Build the deliverable

Read the xlsx skill's SKILL.md first (output-format skill, read only after data is in hand). Then run the bundled builder:

```bash
python3 scripts/build_dm_pack.py <enriched_csv> <output_xlsx> \
  --pair "MON/USDC" --chain monad --window "90 days" \
  --query-url "https://dune.com/queries/XXXXXXX" --top-n 1000
```

It produces four tabs: Read Me (methodology, caveats, live COUNTIF stats), All N, DM Shortlist (non-bot wallets, identity-resolved rows sorted to top and highlighted), Platform Stats (COUNTIF formulas, not hardcoded counts). Arial throughout, no post-2007 formulas.

Then ALWAYS recalculate and verify before presenting:

```bash
python3 <xlsx-skill-path>/scripts/recalc.py <output_xlsx> 60
```

Zero formula errors required. Spot-check the live stats against Python-computed counts.

Present both the xlsx and the enriched CSV with the file-presentation tool.

## Step 5 - Report honestly

The final message must include, in prose the user can forward to a founder:

- Row count and volume range; the Dune query URL for reruns.
- Platform distribution WITH the standing caveat: platforms show where the swap executed; aggregator-routed trades (including Monorail) appear under the AMM that filled them, so this is liquidity venue, not frontend choice.
- Identity yield stated plainly (e.g. "33 ENS, 3 X handles out of 1000") plus the wallet-native fallback channels.
- Bot count and the reframe: high-frequency wallets are not DM targets, they are API/BD targets - one bot operator routing through the aggregator is worth hundreds of retail conversions.
- Anything unconfirmed marked [UNVERIFIED] inline (e.g. Blockscan chat URL format). Never fabricate a datapoint; this user has explicitly corrected fabrication before.

## Failure modes worth knowing upfront

| Symptom | Cause | Fix |
|---|---|---|
| Typed SQL vanishes / appears in a one-line box | It went into Dune's AI prompt (Wand) box | Use the CM6 injection snippet, never keyboard-type SQL |
| `concat` error: varbinary, varchar | `tx_from` is varbinary | `'0x' \|\| lower(to_hex(tx_from))` |
| Results table stuck on same page | Server-side pagination stall | You should not be paginating - use packed rows |
| Dump file missing tail rows | 50 K char hard truncation | Smaller slices; re-dump the missing range only |
| All fetches "Failed to fetch" in Dune tab | Dune CSP | Run fetches from an example.com tab |
| Subagent "can't find" a /var/folders file | Sandbox bash can't see host paths | Tell it to use Read/Grep/Write tools |
| Warpcast 401 | API now requires auth | Skip; note it; suggest Neynar key |
