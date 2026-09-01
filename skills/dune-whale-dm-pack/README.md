# dune-whale-dm-pack

Turn any token pair into a ranked, identity-enriched, DM-ready wallet outreach pack. End to end: Dune query authored and run through a logged-in Chrome tab (no API key handled), wallet extraction past every output-truncation trap, identity enrichment (ENS reverse records, X handles, Telegram) via in-browser fetch loops, bot flagging, and a final xlsx built by the bundled script with per-wallet profile links.

The skill encodes a full working run (MON/USDC on Monad, top 1000 wallets over 90 days) including the discovered traps: Dune's AI prompt box stealing typed SQL, CodeMirror 6 injection, packed-row output to survive page truncation, CSP-blocked fetches, and 1KB output caps on browser tool results.

Files: `SKILL.md` (the pipeline), `references/dune-browser-playbook.md` (the Dune editor traps and exact injection snippets), `references/enrichment-apis.md` (which identity APIs still work anonymously, tested), `scripts/build_dm_pack.py` (xlsx builder).

Defaults to MON/USDC on Monad; works for any pair on any chain Dune indexes.
