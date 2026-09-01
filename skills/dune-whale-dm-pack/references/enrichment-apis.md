# Wallet identity enrichment

How to map raw EVM addresses to people, given a browser and no paid API keys. Findings current as of 2026-07. EVM addresses are identical across chains, so a Monad wallet's identity is resolvable on Ethereum mainnet records.

## Where fetches can run

- Shell sandbox (`mcp__workspace__bash`): NO network to crypto APIs. Confirmed dead for ethereum RPC, warpcast, ensdata. Do not try to enrich from bash.
- Dune tab: fetches blocked by Dune's Content-Security-Policy ("Failed to fetch" on everything).
- A neutral tab on example.com: WORKS. No restrictive CSP, permissive CORS reachable. This is where enrichment runs.

Create the tab, `navigate` to https://example.com, then run fetch loops with `javascript_tool`.

## API status table

| Source | Status | Returns | Notes |
|---|---|---|---|
| api.ensdata.net/<addr> | WORKS | `ens`, `twitter`, `telegram`, `github`, `url` | Primary source. Reverse ENS + text records in one call. |
| api.ensideas.com/ens/resolve/<addr> | WORKS | `name` | ENS name only; backup for ensdata. |
| api.warpcast.com/v2/user-by-verification | DEAD (401) | - | Anonymous lookup removed. Needs auth. Use Neynar API if a key is supplied. |
| api.debank.com internal | GATED | - | Signature-required; not worth fighting. Ship DeBank profile LINKS instead. |
| Public ETH RPC (eth_blockNumber etc.) | Blocked from sandbox | - | Not needed for identity; skip. |

## The enrichment loop (run in the example.com tab)

Inject addresses first (input can be large, output cannot), 250 per call:

```js
window.__addrs = [];
window.__addrs.push('0x...','0x...', /* up to 250 */);   // repeat 4x for 1000
window.__addrs.length;                                    // verify 1000
```

Then fire-and-forget with a concurrency cap, poll a counter:

```js
window.__ens = {}; window.__done = 0; window.__ensDone = false;
(async () => {
  const addrs = window.__addrs, CONC = 6; let i = 0;
  async function worker() {
    while (i < addrs.length) {
      const a = addrs[i++];
      try {
        const r = await fetch('https://api.ensdata.net/' + a);
        if (r.ok) { const j = await r.json();
          if (j && j.ens) window.__ens[a] = [j.ens, j.twitter||'', j.telegram||'', j.github||'', j.url||'']; }
      } catch (e) {}
      window.__done++;
    }
  }
  await Promise.all(Array.from({length: CONC}, worker));
  window.__ensDone = true;
})();
'started';
```

Poll: `({done: window.__done, finished: window.__ensDone, hits: Object.keys(window.__ens).length})` every ~20 s. ~4 min for 1000. Retrieve hits in small chunks (8 entries per `JSON.stringify` slice) to dodge the 1 KB output cap, or write them into a `<pre>` and read via get_page_text like the wallet dump.

If context is getting heavy, hand this whole step to a general-purpose subagent: give it the Chrome tool names, the loop snippet, the CSV path, and instructions to write `ens_results.json` into the shared outputs folder (host path == sandbox `/sessions/.../mnt/outputs`).

## Realistic yield

Young chains resolve 3-10% of wallets to a name. That is expected - most serious onchain wallets have no public ENS. Report the real number; do not imply better coverage than exists. The unresolved wallets are still perfectly targetable through wallet-native messaging, which is why the deliverable carries a DeBank Hi link and a Blockscan Chat link for every wallet.

[UNVERIFIED] Confirm the Blockscan Chat URL format (`https://chat.blockscan.com/index?a=<addr>`) resolves for the target chain before batch outreach; it was not live-tested in the source run.
