# Dune browser playbook

Everything about driving Dune through a logged-in Chrome tab with the Claude-in-Chrome MCP. Read start to finish before touching the editor. Load the Chrome tools in ONE ToolSearch call: `select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__read_network_requests`.

## Contents
- Why the browser (not the API)
- The editor-injection trap
- The packed-row query pattern
- Running and saving
- Exporting data out of the page
- The 50 K truncation ceiling

## Why the browser, not the API

There is no Dune MCP connector. The Dune API needs a key, and keys are credentials that must never be handled in plain text. The user is already logged into Dune in Chrome, so drive that session. Browsers are granted at "read" tier for computer-use (clicks blocked), but the Claude-in-Chrome MCP tools operate on the DOM directly and are not subject to that tier - use them, not pixel clicks.

## The editor-injection trap

Dune's query page has TWO text surfaces: an AI-prompt box (the "Wand", placeholder like "Find the largest token transfers...") and the real SQL editor (CodeMirror 6, `.cm-content`). If you click near the top and type, or use `computer` type actions, the text usually lands in the Wand box and gets interpreted as an AI request. Symptom: your SQL disappears or a one-line prompt bar shows it.

Fix: inject straight into CodeMirror via its DOM, never keyboard-type SQL. CM6 exposes no simple `.value`; use `execCommand('insertText')` on the focused `.cm-content`:

```js
const sql = "...";                       // your query as one line
const el = document.querySelector('.cm-content');
el.focus();
document.execCommand('selectAll', false, null);
document.execCommand('insertText', false, sql);
el.textContent.slice(-60);               // sanity: tail of what landed
```

`view.dispatch(...)` via `el.cmView.view` does NOT work here (`cmView` is undefined). `execCommand` is deprecated but works reliably in this context.

## The packed-row query pattern

Do not select raw rows and paginate the results grid - see the export section for why that stalls. Instead, have SQL pack all N wallets into a handful of long strings, 25 wallets per row, so the whole result is 2 pages of the grid at most.

Base query, then pack. `tx_from` is varbinary, so hex-encode it. Fields are `;`-joined within a wallet, wallets `#`-joined within a chunk:

```sql
with t as (
  select tx_from, amount_usd, block_time, project
  from dex.trades
  where blockchain = 'monad'
    and block_time > now() - interval '90' day
    and ((token_bought_symbol in ('WMON','MON') and token_sold_symbol = 'USDC')
      or (token_sold_symbol in ('WMON','MON') and token_bought_symbol = 'USDC'))
    and amount_usd is not null
),
base as (
  select tx_from as wallet,
         sum(amount_usd) as volume_usd,
         count(*) as swaps,
         min(block_time) as first_trade,
         max(block_time) as last_trade,
         count(distinct date_trunc('day', block_time)) as active_days,
         array_join(array_distinct(array_agg(project)), '|') as platforms
  from t group by 1 order by 2 desc limit 1000
),
ranked as (select *, row_number() over (order by volume_usd desc) as rn from base)
select cast(floor((rn-1)/25) as integer) as chunk,
       array_join(array_agg(
         '0x' || lower(to_hex(wallet)) || ';' ||
         cast(round(volume_usd,2) as varchar) || ';' ||
         cast(swaps as varchar) || ';' ||
         cast(first_trade as varchar) || ';' ||
         cast(last_trade as varchar) || ';' ||
         cast(active_days as varchar) || ';' || platforms
         order by rn), '#') as data
from ranked group by 1 order by 1;
```

Swap `blockchain`, the symbol lists, the interval, and the `limit` for other parameters. Keep `tx_from` (real end-user) rather than `taker`, which can be a router contract. For DuneSQL, `to_hex` on varbinary + `lower` gives a clean `0x...` address; concatenating varbinary directly throws `concat: Unexpected parameters (varbinary, varchar)`.

## Running and saving

Click Run (or "Save and run"). First run of an unsaved query prompts to save - name it descriptively (e.g. "Top 1000 MON-USDC swappers - Monad - 90d"). After saving, the URL becomes dune.com/queries/NNNNNNN; record it for the deliverable. Waits: simple runs finish in 2-5 s but the tab may need 8-15 s of settle time; screenshot to confirm the results grid rendered with the expected column headers before scraping.

Never `navigate` away with unsaved changes - it throws a "Leave site?" block. Save first.

## Exporting data out of the page

Three channels, each with a ceiling:

1. `javascript_tool` return value truncates near 1 KB. Use it to DRIVE and to return small counts/samples, never to return the dataset.
2. `read_network_requests` can catch the results JSON, but Dune's payloads are chunked/encoded and unreliable to reassemble.
3. `get_page_text` reads `<main>`. Under ~25 K chars it returns inline; above that it writes a host file - but that file hard-truncates at exactly 50,000 characters. This is the workhorse, within its ceiling.

Why not paginate the normal results grid: rows load server-side per page (25/page). Pages past the first cost a 30-60 s roundtrip, and a naive next-button loop breaks when the button's DOM position shifts. Scraping 40 pages this way took ~1 hour and lost rows. The packed-row query sidesteps all of it: 1000 wallets = 40 chunk rows = 2 grid pages.

Reading the packed chunks: the two grid pages hold 40 chunk cells. Read page 1's cells into a JS map keyed by chunk index, click to page 2 (poll until the first cell changes, up to ~60 loops of 500 ms - page-2 load is slow), read those, then reconstruct: split each chunk on `#`, each row on `;`, comma-join.

## The 50 K truncation ceiling and how to stay under it

To emit rows for the assembler, build a delimited block and read it with get_page_text. Keep total `<main>` text under 50 K or the tail is silently cut:

```js
document.querySelectorAll('table').forEach(t => t.remove());   // drop the heavy grid
let pre = document.getElementById('__export');
if (!pre) { pre = document.createElement('pre'); pre.id = '__export'; }
document.querySelector('main').prepend(pre);                    // main, not body
pre.textContent = 'CSVSTART\n' + rows.slice(0, 330).join('\n') + '\nCSVEND';
pre.textContent.length;                                         // keep < ~48000
```

~330 CSV rows ~= 47 K chars, safely under. For 1000 rows, dump in 3 slices (0-334, 334-667, 667-1000), each its own get_page_text, each saved to a separate host file. If a slice still crosses 50 K, the CSVEND marker will be missing and the last ~6 rows lost - shrink the slice and re-dump only the missing range.

The saved host files live under a /var/folders/.../tool-results/ path. Subagent Bash cannot see these; only the Read/Grep/Write tools can. Extract with Grep, pattern `0x[0-9a-fA-F]{40},[^"\\]+`.
