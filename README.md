# oss-contribution-harness

A regression-safe pipeline for getting AI-drafted pull requests merged upstream
instead of filed as noise. Implements the credibility signals that maintainers
look for now that AI agents have changed the contributor pipeline.

> "Talk and code is cheap. Show me you really care."
> — Roger Wang, MLSys 2026

Implemented as a [Claude Code](https://claude.com/claude-code)
skill, but the workflow is tool-agnostic — the gates are the point.

## Why this exists

AI coding agents changed open source contribution. Weekly PR volume to major
AI infrastructure repos spiked visibly around agent releases in 2025–2026.
Generating plausible code is no longer the bottleneck — the bottleneck is
reviewer trust. Maintainers now need to evaluate whether the contributor
understands the system, whether the change solves the right problem, and
whether the person will stay involved after the PR is opened.

LLM-driven contribution at scale fails for predictable reasons:

1. PRs that *look* like fixes but don't reproduce on the target.
2. PRs that follow `CONTRIBUTING.md` but ignore the *revealed* norm in merge history.
3. PRs that "fix" load-bearing intentional behavior.
4. Drafts in generic LLM-PR voice that maintainers spot instantly.
5. Duplicates of open or recently-closed work.
6. Bots that pierce CLA / merge / approval clicks they have no authority to make.
7. Security-class findings pushed to public PR diffs instead of disclosure channels.
8. PRs orphaned at "open" — never followed up after CI / reviews / bots.
9. Velocity that platforms read as abuse regardless of per-item quality.
10. Hardcoded branches that can't factor per-repo context the design stage missed.

This harness encodes ten rules as **hard phase gates** with explicit judgment
points, not pre-declared logic. Each rule corresponds to a phase in the pipeline
and writes its evidence to per-candidate state under `~/.omc/ghcontrib/`.

## The 12 rules

| # | Rule | Where it lives in the harness |
|---|------|------------------------------|
| 1 | **Reproduction is the merge gate.** No repro → candidate dies. | Phase 5 — gate on `repro_verdict`. |
| 2 | **Imitate merge history, not CONTRIBUTING.md.** | Phase 3 — fingerprint last 10 merged PRs. |
| 3 | **Respect load-bearing behavior.** Some "bugs" are intentional. | Phase 2 — blame + linked issues + design discussion. |
| 4 | **Few-shot with your own past PRs.** | Phase 4 — pull `gh search prs --author $ME --merged`. |
| 5 | **Dedupe before drafting.** Open + closed issues + PRs. | Phase 1 — `gh search` then read every result. |
| 6 | **Wrap attestation; never pierce it.** | Phase 9 — explicit `y/N` from a human; never auto-y. |
| 7 | **Route security-sensitive findings off the public PR path.** | Phase 6 — heuristic classifier → `security-queue/`. |
| 8 | **Plan the follow-up in the pipeline.** | Phase 11 — T+24h / T+72h / T+7d schedule. |
| 9 | **Throttle on calendar time, not just per-item quality.** | Phase 8 — append-only ledger, ≤5 PRs/7d/identity. |
| 10 | **Let the agent branch at decision points.** | Recorded in `state.json.decisions[]` with rationale. |
| 11 | **Demonstrate system understanding.** Plausible code is not a credibility signal. | Phase 2 — `understanding.md` proves subsystem comprehension. |
| 12 | **Make design decisions reviewable, not just the diff.** | Phase 7 — "Design context" section in the PR body. |

## Pipeline

```
scout → candidate → dedupe → archaeology → reveal → fewshot → reproduce
      → draft → security-gate → attest → open → followup → close|merged|dead
```

Forward-only. Each phase writes evidence to
`~/.omc/ghcontrib/candidates/<owner>__<repo>__<id>/`.

## Install

Drop `SKILL.md` into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/omc-learned/ghcontrib
curl -sLo ~/.claude/skills/omc-learned/ghcontrib/SKILL.md \
  https://raw.githubusercontent.com/yurekami/oss-contribution-harness/main/SKILL.md
```

Prerequisites:

- `gh` (authed: `gh auth login`)
- `git`
- `jq` (Windows: `winget install jqlang.jq`; macOS: `brew install jq`)

State directories are created on first run.

## Invocation

```
/ghcontrib <owner/repo> --scout
/ghcontrib <owner/repo> --candidate <issue-url>
/ghcontrib --resume <id>
/ghcontrib --followup        # sweep all open PRs and compute next actions
```

## Real example

[`examples/deepseek-aider/`](./examples/deepseek-aider/) contains the full
state of one live run that produced
[deepseek-ai/awesome-deepseek-integration#582](https://github.com/deepseek-ai/awesome-deepseek-integration/pull/582)
— Aider added to the Applications list, +5 LOC matching the median of recent
merges, drafted in the user's voice, with the closed precedent issue
referenced and the maintainer's explicit invitation cited.

`state.json` shows every gate decision and rationale; `draft.md` is the PR
body that went up; `reveal-summary.json` is the merge-history fingerprint
that drove sizing and tone.

## Attestation

This harness **never** clicks CLA, signs DCO, approves a review, merges a PR,
or pushes as anyone other than the identity `gh` is authed as. Every
network-visible action requires a fresh in-session human `y`. If the source
project requires a CLA, the human signs it in their browser. The bot
automates *up to* the click, never *through* it.

## Throttle

Default budget: ≤5 PRs / 7 days / identity, ≤2 PRs / repo / 14 days.
Edit the constants in Phase 8 of `SKILL.md` to suit your tolerance for
platform abuse heuristics. Velocity is a semantic signal regardless of
per-item quality.

## Security

If a candidate touches auth, crypto, sandbox, deserialization, SQL, shell
exec, path traversal, SSRF, TLS — or if the diff would *itself* reveal a
vulnerability — the harness redirects to `~/.omc/ghcontrib/security-queue/`
and surfaces the project's `SECURITY.md` disclosure channel. It does not
push.

## Attribution

Rules 11–12 and the credibility principle were informed by Roger Wang's talk
"Rethinking Open Source Contribution in the Age of AI Agents" at MLSys 2026.
Roger is a core maintainer of [vLLM](https://github.com/vllm-project/vllm)
and co-founder of Inferact. The talk drew on direct experience maintaining
AI infrastructure as agent-generated PRs became a significant share of
contributor activity.

## License

MIT — see [LICENSE](./LICENSE).
