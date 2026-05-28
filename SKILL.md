---
name: ghcontrib
description: Regression-safe OSS contribution pipeline — dedupe, reproduce, style-match, attest, throttle, follow-up. Enforces the 10 rules as gated phases over per-candidate state.
triggers:
  - ghcontrib
  - contribute to upstream
  - oss contribution
  - upstream pr pipeline
  - deepseek contribution
argument-hint: "<owner/repo> [--scout | --candidate <issue-url> | --resume <id> | --followup]"
---

<Purpose>
A contribution harness for open-source GitHub repositories that enforces the rules below as hard phase gates, with per-candidate state persisted under `$HOME/.omc/ghcontrib/`. Judgment gates are explicit — the agent decides per-repo rather than running hardcoded branches.

The rules:
1. Reproduction is the merge gate. No repro → candidate dies.
2. Imitate merge history, not CONTRIBUTING.md. Revealed norm > stated norm.
3. Respect load-bearing behavior. Some "bugs" are intentional.
4. Few-shot with the user's own merged PRs. Style calibration.
5. Dedupe before drafting. Open + closed issues + PRs.
6. Wrap attestation; never pierce it. Human clicks CLA/merge.
7. Route security-sensitive findings off the public PR path.
8. Plan the follow-up (CI, reviewer comments, bot signals).
9. Throttle on calendar time. Velocity is a semantic signal.
10. Branch at decision points (Ouroboros). Judgment > pre-declared rules.
11. Demonstrate system understanding. Plausible code is not a credibility signal.
12. Make design decisions reviewable, not just the diff.
</Purpose>

<Credibility_Principle>
> "Talk and code is cheap. Show me you really care."
> — Roger Wang, MLSys 2026 ("Rethinking Open Source Contribution in the Age of AI Agents")

AI coding agents changed the contributor pipeline. Weekly PR volume to major AI infrastructure repos (vLLM, etc.) spiked visibly around agent releases in 2025–2026. More people can generate plausible code and open pull requests quickly. That shifts what maintainers look for.

The hard part of review is no longer "does the code work." It is:
- Does the contributor understand the system they are changing?
- Does the change solve the right problem at the right scale?
- Will the contributor stay involved after the PR is opened?

If the new on-ramp to open source is "prompt an agent," maintainers will see more contributors who have not deeply read the codebase. The harness must produce contributions that carry **credibility signals beyond the diff**:

| Signal | Where the harness enforces it |
|--------|-------------------------------|
| **System understanding** | Phase 2 — `understanding.md` must show the contributor read the relevant subsystem, not just the target lines |
| **Right problem, right scale** | Phase 0 — scout evaluates problem-fit, not just repo-fit; Phase 3 — reveal constrains scope to median |
| **Clear communication** | Phase 7 — design context section makes decisions reviewable, not just the code |
| **Ownership beyond the PR** | Phase 11 — follow-up is not optional; orphaned PRs are closed, not left rotting |
| **Contributor voice** | Phase 4 — few-shot calibration; cold-start contributors default to terser, more observation-driven drafts |

The bar for contribution is higher and clearer now. This harness exists to clear that bar, not to lower it.
</Credibility_Principle>

<Use_When>
- User wants to contribute code/docs to an upstream GitHub repository
- User says "contribute to", "upstream pr", "ghcontrib", "oss contribution"
- User names a target org/repo and wants a managed pipeline
- Resuming a prior candidate (`--resume <id>`) or sweeping follow-ups (`--followup`)
</Use_When>

<Do_Not_Use_When>
- User wants bulk automated PR spam across many repos (this is explicitly what rule 9 exists to prevent)
- Task is a private/internal repo (use normal git workflow)
- User wants to self-approve / auto-merge (rule 6 forbids piercing attestation)
</Do_Not_Use_When>

<Prerequisites>
Run once before first use:

```bash
command -v gh  >/dev/null && gh auth status 2>&1 | grep -q "Logged in" && echo "gh: OK" || echo "gh: NOT AUTHED"
command -v git >/dev/null && echo "git: OK" || echo "git: MISSING"
# jq is preferred. On Windows, python3 is an acceptable fallback (see $JQ_OR_PY below).
command -v jq  >/dev/null && echo "jq: OK" || {
  command -v python3 >/dev/null && echo "jq: MISSING — using python3 fallback" || echo "jq: MISSING and no python3 — install jq via 'winget install jqlang.jq'"
}
test -d "$HOME/.omc/ghcontrib" || mkdir -p "$HOME/.omc/ghcontrib/security-queue" "$HOME/.omc/ghcontrib/candidates"
```

**JSON helper contract.** Examples below use `jq`. If jq is unavailable, the skill falls back to python3. Define once at the top of any session:

```bash
if command -v jq >/dev/null; then
  jqf() { jq "$@"; }
else
  # minimal compat: reads stdin JSON, evaluates expression like jq -r would for simple paths
  jqf() { python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d, indent=2))"; }
fi
```

For complex filters (group_by, arithmetic on dates, etc.) jq is required — install via `winget install jqlang.jq` on Windows. The skill will hard-fail on those phases without jq and tell you exactly which command needs it.

Identity is read from `gh auth status` (authoritative). Never push or commit as any identity other than the one `gh` is authed as. Never push to `upstream` — only to the user's fork.
</Prerequisites>

<State_Model>
All state lives under `$HOME/.omc/ghcontrib/`:

```
$HOME/.omc/ghcontrib/
├── throttle.jsonl              append-only: {ts, action, repo, identity}
├── security-queue/             redirected findings; NEVER pushed
│   └── <iso-ts>-<owner>-<repo>-<slug>.md
├── candidates/
│   └── <owner>__<repo>__<id>/
│       ├── state.json          {phase, created_at, repo, issue_url, pr_url}
│       ├── dedupe.json         matches from open+closed issues/PRs
│       ├── reveal.md           last-10-merged-PRs style fingerprint
│       ├── archaeology.md      blame, linked issues, design discussion
│       ├── understanding.md    subsystem comprehension proof (rule 11)
│       ├── fewshot.md          user's merged PRs as style corpus
│       ├── repro/              clone + repro script + logs
│       ├── draft.md            draft PR body (never pushed without confirm)
│       ├── security.flag       if present, candidate is redirected, not merged
│       └── followup.json       {next_check_at, pr_number, ci_state, reviews}
```

Phases (state machine): `scout → candidate → dedupe → archaeology → reveal → fewshot → reproduce → draft → security-gate → attest → open → followup → close|merged|dead`

Moving backward is allowed. Skipping forward is not.
</State_Model>

<Workflow>

## Phase 0 — Scout (rule 10, judgment gate)

Given an `<owner/repo>`, determine whether this project accepts external contributions and in what form. Many research orgs (DeepSeek being the canonical example) release weights + paper + inference reference code and do not meaningfully merge external code PRs. Check before spending any budget.

```bash
OWNER="$1"; REPO="$2"  # or parse from owner/repo
gh api "repos/$OWNER/$REPO" --jq '{stars:.stargazers_count,issues:.open_issues_count,archived,fork,license:.license.spdx_id,default_branch,pushed_at}'
# Signal: merged PRs + authorship distribution (last 50; filter to 90d with jq)
gh pr list --repo "$OWNER/$REPO" --state merged --limit 50 --json author,mergedAt,title \
  | jq '[.[] | select(.mergedAt > (now - 86400*90 | todate))] | group_by(.author.login) | map({author:.[0].author.login, count:length}) | sort_by(-.count)'
```

**Judgment gate:** if >80% of recent merges are by ≤2 maintainer accounts and external contributors get `0-2` merges/quarter, treat this as a **research-release repo**. Contribution targets shrink to:
- `awesome-*` listing repos (doc/integration entries)
- Integration examples / adapters
- Obvious typos or broken links
- Reproducible numerical/correctness bugs (rare, high-value)

**Right problem, right scale.** Even in contribution-friendly repos, evaluate whether the candidate issue is the right *kind* of problem for an external contributor. Maintainers value contributors who pick problems that match their depth of system understanding. Signals that a problem is right-sized:
- The issue is self-contained — doesn't require touching 5 subsystems the contributor hasn't read
- The scope matches the reveal norm (Phase 3 will enforce this numerically)
- The issue is not "design the architecture" work that requires deep context only maintainers have
- For first-contact contributions: prefer concrete bugs, missing tests, documentation gaps, or clearly-scoped enhancements over refactors or new features

Record the scout decision in `candidates/<id>/state.json` as `scout_verdict`. If the verdict is "research-release with no viable surface," stop here. Do not proceed to dedupe.

## Phase 1 — Dedupe (rule 5)

**Merge gate for wasted author time.** Search before drafting.

```bash
Q="$1"  # short phrase describing the candidate
# NOTE: gh search --state only accepts open|closed. Omit it entirely to search both (results include .state for filtering).
gh search issues --repo "$OWNER/$REPO" "$Q" --limit 30 --json number,title,state,url,createdAt > "$CAND/dedupe-issues.json"
gh search prs    --repo "$OWNER/$REPO" "$Q" --limit 30 --json number,title,state,url,createdAt > "$CAND/dedupe-prs.json"
jq -s '{issues:.[0], prs:.[1], open_count:((.[0]+.[1])|map(select(.state=="open"))|length), recent_closed: ((.[0]+.[1])|map(select(.state!="open" and (.createdAt > (now - 86400*180 | todate)))))}' \
  "$CAND/dedupe-issues.json" "$CAND/dedupe-prs.json" > "$CAND/dedupe.json"
```

Read every result. If any open or recently-closed (≤180d) item covers the same ground, one of:
- Comment on the existing thread with reproduction evidence (do NOT open a new PR)
- Kill the candidate; record `dead: duplicate of <url>` in `state.json`
- Narrow scope so the new work is genuinely orthogonal

Advance phase only when dedupe.json has been read *by name* and recorded.

## Phase 2 — Archaeology (rule 3, load-bearing behavior)

Before assuming something is a bug, check whether it is load-bearing on purpose.

```bash
# blame the relevant lines
git -C "$CAND/repro" blame -L <start>,<end> <path>
# issue history referencing the line
gh search issues --repo "$OWNER/$REPO" "<symbol-or-filename>" --state all --limit 20 --json number,title,state,url
# design discussions
gh api "repos/$OWNER/$REPO/discussions?per_page=30" --paginate 2>/dev/null | jq '.[] | select(.title | test("<keyword>"; "i"))'
```

Write `archaeology.md` with: original-author, original-commit-rationale, any linked design doc, any prior attempts at the same change and why they closed. If any of those say "this is intentional," **stop** and convert the candidate to a question issue (not a PR) or kill it.

**System understanding artifact (rule 11).** Also write `understanding.md` — a brief document (5–15 lines) showing the contributor has read and understood the relevant subsystem, not just the target lines. This is the credibility signal that separates "I prompted an agent to change line 194" from "I understand how `formatLinterResults` composes output from the `NamedLinter` pipeline and why the disable hint belongs in the per-linter header, not per-warning."

Contents of `understanding.md`:
- What subsystem this change touches and how it fits into the larger architecture
- What other code paths depend on or are affected by this change
- Why the chosen approach is correct given the system's design, not just that it compiles
- Any constraints or invariants the contributor discovered by reading the code

This artifact is not included in the PR body (maintainers don't want to read your homework). It exists to force the contributor — or the agent acting on their behalf — to actually read the codebase before generating a diff. If `understanding.md` cannot be written convincingly, the contributor does not understand the system well enough to change it.

## Phase 3 — Reveal (rule 2, merge history over CONTRIBUTING.md)

CONTRIBUTING.md is the stated norm; merged PRs are the revealed norm. Extract the revealed norm.

```bash
# NOTE: use `gh pr list` (not `gh search prs`) — search does not expose mergedAt/additions/deletions/changedFiles.
gh pr list --repo "$OWNER/$REPO" --state merged --limit 10 \
  --json number,title,additions,deletions,changedFiles,body,author,mergedAt > "$CAND/reveal-prs.json"
# summary fingerprint: median LOC + files, distinct external authors, size distribution
jq '{median_loc: ([.[] | .additions + .deletions] | sort | .[length/2|floor]),
     median_files: ([.[] | .changedFiles] | sort | .[length/2|floor]),
     distinct_authors: ([.[] | .author.login] | unique | length),
     title_prefixes: ([.[] | .title | split(":") | .[0]] | group_by(.) | map({p:.[0], n:length}))}' \
  "$CAND/reveal-prs.json" > "$CAND/reveal-summary.json"
# sample the actual diffs (tone + structure)
for N in $(jq -r '.[].number' "$CAND/reveal-prs.json" | head -5); do
  gh pr view "$N" --repo "$OWNER/$REPO" --json title,body,additions,deletions,files,reviews
done > "$CAND/reveal-samples.json"
```

Synthesize `reveal.md`:
- **Size**: median changed LOC and file count of the last 10 merges
- **Commit structure**: squash vs. multi-commit; imperative vs. descriptive titles
- **Test expectation**: are tests added/modified in merged PRs? what framework?
- **Description tone**: terse bullets, prose explanation, linked issue required?
- **Review latency**: median days to first review, days to merge

This fingerprint drives PR sizing and description tone. If the candidate's scope blows past the median by >3×, split it or narrow.

## Phase 4 — Few-shot (rule 4, the user's own voice)

Maintainers recognize voice. Don't let the draft drift into generic LLM-PR register.

```bash
ME=$(gh api user --jq .login)
# gh search prs exposes: number,title,author,closedAt,updatedAt,url,repository,state,body,labels,isDraft,isPullRequest
# It does NOT expose mergedAt or body reliably — use --merged flag and pull body per-PR via gh pr view.
gh search prs --author "$ME" --merged --limit 10 --json url,title,repository,closedAt > "$CAND/fewshot-index.json"
# enrich with body per PR (search doesn't return body for external repos)
echo "[]" > "$CAND/fewshot.json"
while IFS=$'\t' read -r URL TITLE; do
  REPO=$(echo "$URL" | sed -E 's|https://github.com/([^/]+/[^/]+)/pull/.*|\1|')
  NUM=$(basename "$URL")
  BODY=$(gh pr view "$NUM" --repo "$REPO" --json body --jq .body 2>/dev/null)
  jq --arg url "$URL" --arg title "$TITLE" --arg body "$BODY" \
    '. += [{url:$url, title:$title, body:$body}]' "$CAND/fewshot.json" > "$CAND/fewshot.json.tmp" \
    && mv "$CAND/fewshot.json.tmp" "$CAND/fewshot.json"
done < <(jq -r '.[] | [.url, .title] | @tsv' "$CAND/fewshot-index.json")
jq -r '.[] | "### \(.title)\n\(.body // "(no body)")\n\n---\n"' "$CAND/fewshot.json" > "$CAND/fewshot.md"
```

Include `fewshot.md` in the draft prompt as the style target. If `fewshot.md` is empty (no prior merged PRs under this identity), flag it in `state.json` as `fewshot_cold_start: true` and prefer terser drafts — a first-contact PR that sounds like GPT is a maintainer red flag.

## Phase 5 — Reproduce (rule 1, the merge gate)

**This is the single hardest gate. Everything upstream of it is cosmetic if skipped.**

```bash
cd "$CAND" && gh repo clone "$OWNER/$REPO" repro -- --depth 50
cd repro
# write repro.sh that exercises the claim end-to-end, from a clean checkout
cat > ../repro.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
# concrete commands that reproduce the bug on main
EOF
bash ../repro.sh 2>&1 | tee ../repro.log
```

Gate:
- Repro runs on `main` at `$(git rev-parse HEAD)` and observably fails (for a bug) or shows the gap (for a feature).
- Record `repro_head`, `repro_verdict` (`reproduces` | `does-not-reproduce` | `flaky`) in `state.json`.
- `does-not-reproduce` → candidate dies. Do not draft.
- `flaky` → not a merge gate. Open an issue first, do not draft a fix PR.

## Phase 6 — Security gate (rule 7)

Before drafting, classify the candidate. If it is security-relevant, **redirect off the public PR path**.

Heuristics that trigger security redirect:
- Changes touch auth, crypto, sandbox escapes, deserialization, SQL, shell exec, path traversal, SSRF, TLS
- Repro demonstrates data exfiltration, auth bypass, privilege escalation, RCE
- The fix would *itself* reveal the vulnerability in a public diff

If any trigger fires:
```bash
SLUG=$(date -u +%Y%m%dT%H%M%SZ)-$OWNER-$REPO-$(echo "$TITLE" | tr ' ' '-' | tr -cd 'A-Za-z0-9-')
cp "$CAND/draft.md" "$HOME/.omc/ghcontrib/security-queue/$SLUG.md" 2>/dev/null || true
touch "$CAND/security.flag"
# Find the project disclosure channel — do NOT push
gh api "repos/$OWNER/$REPO/contents/SECURITY.md" --jq '.content' 2>/dev/null | base64 -d
gh api "repos/$OWNER/$REPO" --jq '.security_and_analysis'
```

Report to the user: "security-class finding; redirected to `$HOME/.omc/ghcontrib/security-queue/$SLUG.md`; disclosure channel is `<email or advisory URL>`; I will not open a PR." Then stop. Rule 7 is non-negotiable.

## Phase 7 — Draft (compose, do not push)

Compose the PR body using `reveal.md` (shape) + `fewshot.md` (voice) + `archaeology.md` (context) + `repro.log` (proof).

```bash
cat > "$CAND/draft.md" <<EOF
<title matching revealed norm; imperative, ≤70 chars>

## What
<one paragraph or tight bullets matching the revealed tone>

## Why
<link to existing issue if one was found in dedupe.json; cite archaeology.md if relevant>

## Design context
<make the design decision reviewable, not just the code (rule 12):
 - what alternatives were considered and why this approach was chosen
 - what the change does NOT do and why (non-goals prevent scope creep reviews)
 - any constraints discovered during archaeology that shaped the approach
 Keep this to 2-4 sentences. Skip for trivial changes (typos, version bumps).>

## Reproduction
<verbatim from repro.log, trimmed>

## Test
<matches the test framework revealed in reveal.md>

## Scope / non-goals
<explicit list; avoid scope drift>
EOF
```

Write the patch. Keep diff size at or under the `reveal.md` median. Commit structure matches revealed norm (usually squash-friendly single commit with imperative subject).

**Design reviewability (rule 12).** Maintainers spend review time understanding *why* a change was made this way, not just *what* changed. The "Design context" section exists so the reviewer can evaluate the decision, not just the diff. This is especially important when AI agents draft the code — the maintainer needs to see that judgment was applied, not just code generation.

## Phase 8 — Throttle (rule 9, calendar-time budget)

Before opening anything on the network, check the ledger.

```bash
ME=$(gh api user --jq .login)
LEDGER="$HOME/.omc/ghcontrib/throttle.jsonl"
# count PR-opens in last 7 days across all repos, under this identity
COUNT_7D=$(jq -r --arg me "$ME" 'select(.action=="pr_open" and .identity==$me) | .ts' "$LEDGER" 2>/dev/null \
  | awk -v cutoff=$(date -u -d '7 days ago' +%s 2>/dev/null || gdate -u -d '7 days ago' +%s) \
    '{cmd="date -u -d \""$0"\" +%s"; cmd|getline t; close(cmd); if (t>cutoff) print}' \
  | wc -l)
# default budget: ≤5 PR-opens / 7 days / identity, ≤2 PRs / repo / 14 days
REPO_COUNT_14D=$(jq -r --arg me "$ME" --arg repo "$OWNER/$REPO" \
  'select(.action=="pr_open" and .identity==$me and .repo==$repo) | .ts' "$LEDGER" 2>/dev/null | wc -l)
echo "7d-total=$COUNT_7D  14d-this-repo=$REPO_COUNT_14D"
```

Gates:
- 7d-total ≥ 5 → **defer**. Record `state.json.deferred_until` = now + N days until count drops.
- 14d-this-repo ≥ 2 → **defer for this repo specifically**.
- Both under budget → proceed. Append to ledger only *after* the user confirms (phase 9).

**Do not bypass this to hit arbitrary velocity targets. Velocity is read as a semantic signal by both maintainers and platform abuse systems.**

## Phase 9 — Attest (rule 6, never pierce)

Before any network-visible action, stop and require explicit user confirmation *in this session*. The user clicks; the harness does not.

Checklist shown to the user:
- [ ] Scout verdict acceptable
- [ ] Dedupe read, no duplicate
- [ ] Archaeology does not say "intentional"
- [ ] Repro verdict = `reproduces`
- [ ] Draft matches revealed norm
- [ ] No security flag
- [ ] Throttle under budget
- [ ] Identity is correct: `<from gh auth>`
- [ ] Push target is the user's fork, not `upstream`

Ask: "Ready to open PR? (y/N)". If anything other than an explicit `y`, stop.

CLA: if the repo requires CLA, stop and tell the user to sign it in-browser. Do not attempt to click through or sign on behalf of the user. Same for merge, review approval, and Dependabot-style sign-off.

## Phase 10 — Open (attested action)

Only after phase 9 y:

```bash
git -C "$CAND/repro" checkout -b "ghcontrib/$(basename $CAND)"
git -C "$CAND/repro" add -p    # interactive; user reviews the hunks
git -C "$CAND/repro" commit -m "<subject from draft.md>"
# push to the user's fork, not upstream
gh repo fork "$OWNER/$REPO" --remote --clone=false 2>/dev/null || true
git -C "$CAND/repro" push -u origin HEAD
PR_URL=$(gh pr create --repo "$OWNER/$REPO" --title "$(head -1 $CAND/draft.md)" --body-file "$CAND/draft.md")
echo "{\"ts\":\"$(date -u +%FT%TZ)\",\"action\":\"pr_open\",\"repo\":\"$OWNER/$REPO\",\"identity\":\"$(gh api user --jq .login)\",\"pr\":\"$PR_URL\"}" >> "$HOME/.omc/ghcontrib/throttle.jsonl"
jq --arg pr "$PR_URL" '.phase="followup" | .pr_url=$pr' "$CAND/state.json" > "$CAND/state.json.tmp" && mv "$CAND/state.json.tmp" "$CAND/state.json"
```

## Phase 11 — Follow-up (rule 8, the pipeline doesn't end at open)

An unattended PR rots. Schedule checks at T+24h, T+72h, T+7d.

```bash
# resumable: scan all candidates where phase == "followup"
for C in $HOME/.omc/ghcontrib/candidates/*/; do
  PHASE=$(jq -r .phase "$C/state.json" 2>/dev/null)
  [ "$PHASE" = "followup" ] || continue
  PR=$(jq -r .pr_url "$C/state.json")
  PR_NUM=$(basename "$PR")
  OWNER_REPO=$(echo "$PR" | sed -E 's|https://github.com/([^/]+/[^/]+)/pull/.*|\1|')
  gh pr view "$PR_NUM" --repo "$OWNER_REPO" --json state,statusCheckRollup,reviews,comments,mergeable
done
```

For each PR, the harness computes the **next action** and records it in `followup.json`:

| Signal | Next action |
|---|---|
| CI red | Read failing job log, push fix commit after user confirm, or close with explanation |
| Requested changes | Draft response, cite specific commits, push fix after user confirm |
| Approved, merge queued | Wait, record `merged` when merged |
| Silent ≥14d, no review | Polite bump comment once; after second silence at 30d, close |
| Maintainer says "won't fix" | Close, record `dead: declined`, update reveal.md learnings |
| Bot says "signoff" / DCO / CLA | Stop, tell user to click |

Never orphan. If a candidate sits in `followup` for >60 days with no resolution, escalate to the user.

**Ownership beyond the PR.** A PR is not a fire-and-forget artifact. The contributor signals credibility by:
- Responding to reviewer feedback within 48h, not letting comments age
- Pushing fix commits that address the *spirit* of the review, not just the letter
- Closing the PR themselves if it becomes clear the approach is wrong, rather than waiting for the maintainer to do it
- Thanking reviewers for their time — reviewer bandwidth is the scarcest resource in OSS

If the harness is used by an AI agent, the agent must schedule follow-up checks and surface reviewer comments to the human. The human decides the response. An AI-generated "thanks for the feedback, I'll fix it" that is never followed up is worse than silence.

## Phase 12 — Close out

On merge/close, append a final event to the ledger and write a short post-mortem into `candidates/<id>/postmortem.md`. Record what the revealed-norm fingerprint got right and wrong — feed back into future drafts.

</Workflow>

<Judgment_Gates>
These are the decision points where the agent *should* branch on per-repo context, not hardcoded rules (rule 10):

1. **Scout verdict** — is this repo structurally contribution-friendly?
2. **Problem-fit** — is this the right problem at the right scale for this contributor's depth? (rule 11)
3. **Archaeology flags "intentional"** — downgrade to issue or kill?
4. **Understanding quality** — can the contributor articulate how the subsystem works, not just what to change? (rule 11)
5. **Dedupe partial match** — comment on existing thread, narrow, or kill?
6. **Repro flaky** — open issue first, do not draft fix
7. **Security classification** — redirect or proceed?
8. **Reveal shows tiny PRs, candidate is large** — split, narrow, or kill?
9. **Design reviewability** — does the draft explain the decision, not just the diff? (rule 12)
10. **Fewshot cold-start** — terser, more observation-driven draft
11. **Throttle near budget** — defer with an explicit date
12. **PR silent ≥14d** — bump once, then at 30d close; do not spam
13. **CI fails after push** — fix or close; do not leave a red PR sitting
14. **Reviewer responsiveness** — respond within 48h or escalate to user; never let feedback age

At each gate, record the decision and one-line rationale in `state.json` under `decisions: [{gate, choice, rationale, ts}]`. These become the dataset for tuning the harness.
</Judgment_Gates>

<DeepSeek_Note>
DeepSeek (`deepseek-ai/*`) is a research-release org. The high-star repos (`DeepSeek-V3`, `DeepSeek-R1`, `Janus`) are primarily weights + paper + inference reference; external code PRs are rarely merged. Before running this harness against DeepSeek, scout these surfaces specifically:

- `deepseek-ai/awesome-deepseek-integration` — the only repo that actively accepts community PRs (integration listings). Check `pulls?state=closed` to confirm recent merges by non-maintainers.
- `deepseek-ai/DeepSeek-Coder` — inference scripts; occasional small fixes merged.
- CUDA kernel repos (`FlashMLA`, `DeepGEMM`, `DeepEP`) — very high technical bar; numerical correctness and perf only, not stylistic.

If the scout verdict says "research-release, no viable surface," route the user's contribution energy elsewhere rather than forcing a low-probability PR.
</DeepSeek_Note>

<Invocation>
From a session:

```
/ghcontrib deepseek-ai/awesome-deepseek-integration --scout
/ghcontrib deepseek-ai/DeepSeek-Coder --candidate https://github.com/deepseek-ai/DeepSeek-Coder/issues/123
/ghcontrib --followup    # sweep all open PRs and compute next actions
/ghcontrib --resume <candidate-id>
```

The skill dispatches to the appropriate phase based on state.json.
</Invocation>

<Completion_Criteria>
A run of this skill is complete when every active candidate is in one of:
- `merged` — PR merged upstream
- `closed` — PR closed (by user or maintainer, with rationale recorded)
- `dead: duplicate` / `dead: intentional` / `dead: does-not-reproduce` / `dead: declined` — killed at a gate
- `deferred` — throttle budget exceeded, has a `deferred_until` date
- `security-redirected` — in `security-queue/`, disclosure channel identified

Never leave a candidate in `followup` without a scheduled next-check date.
</Completion_Criteria>
</content>
</invoke>