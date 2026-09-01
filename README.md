# Monorail Growth Engine

The agent-skills growth desk behind Monorail's growth function, running on Claude.

I run growth at [Monorail](https://monorail.xyz), the swap aggregator on Monad, as a one-person function. These are the Claude agent skills that make that possible. Each one packages a real workflow I ran manually first, with every correction from live runs written back into the skill.

This is not prompt-collection theater. Each skill encodes a tested pipeline: the traps, the workarounds, the quality gates, and an iteration log where the playbook updates after every run.

## The skills

| Skill | What it does |
|---|---|
| [`charubak-growth-engine`](skills/charubak-growth-engine/) | The full first-week growth engagement for any Web3 project: content guidelines, competitor recon, creator plan, narrative digest, funnel teardown, user signal map, outreach email. Battle-tested on a live prediction market during the 2026 World Cup. |
| [`monad-daily-digest`](skills/monad-daily-digest/) | Daily ecosystem monitoring: 38 X/Twitter queries, narrative classification into six buckets, bridge-opportunity flags, integration-target scanning with mandatory URL verification, one Telegram message by 8am. |
| [`dune-whale-dm-pack`](skills/dune-whale-dm-pack/) | Token pair in, DM-ready whale list out. Dune query through a logged-in browser (no API key), wallet extraction, identity mapping (ENS, X, Telegram), bot flagging, xlsx outreach pack. |

A few more run privately: brand-voice and campaign-writing skills for Monorail, and an outreach engine for Signal. Those encode my employer's playbooks, so they stay off the public internet.

## How they work

Install any skill folder into Claude (Cowork or Claude Code) as a skill. Trigger it in plain language: "run the engagement for X", "run the wallet intel", "run the daily digest". Skills with `{{PLACEHOLDER}}` values need your own tokens and IDs filled in before use.

The design principles are consistent across all of them: everything sourced and dated or it does not ship, estimates labeled as estimates, no invented numbers, verification steps built in, and honest tiers for what AI delivers versus what stays human (negotiation, taste, final approval).

## Why publish this

Most marketers talk about AI leverage. I would rather show the actual machinery. If you are building a growth function on agent skills, steal the structure: the iteration logs, the verification gates, and the changelog discipline matter more than any individual prompt.

More at [web3growthlab.com](https://web3growthlab.com) · [@thedeludedbull](https://x.com/thedeludedbull)

License: all rights reserved. The skills are published for reference and learning; ask before commercial reuse.
