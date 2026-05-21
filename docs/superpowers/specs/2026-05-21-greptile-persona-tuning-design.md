# 2026-05-21 — Greptile persona tuning from Slack huddle

Status: design approved in brainstorming; awaiting writing-plans handoff.
Working repo: `opensourcerouting/frr-greptile` (local clone `/home/suphawit/Documents/git/frr-greptile`).

## 1. Background

### 1.1 Baseline state

`opensourcerouting/frr-greptile` is a dedicated Greptile-experiment repo created 2026-05-15 after Martin declined a master-rewind on `opensourcerouting/frr`. Its `master` carries a slim 3-rule `.greptile/` config (`memory-allocation`, `logging-api`, `string-safety`) seeded by PR #1 (merged 2026-05-15). The current `instructions` field reads:

> "You are a senior core maintainer of FreeRangeRouting (FRR) with deep expertise in routing protocols (BGP, OSPF, IS-IS, Zebra, MPLS, PIM, VRRP, BFD), C systems programming, YANG data modeling, and network daemon architecture. Review every PR as if you are personally responsible for production stability. Be direct and critical — do not soften feedback."

Earlier real-PR mirrors on the prior fork (`opensourcerouting/frr#241`, `#242`) showed empirically that **none of the three custom rules fired** on actual refactor PRs — every useful finding came from the persona + `strictness: 2`. Persona is the lever.

### 1.2 Slack huddle content

A development huddle (Mark Stapp, Chris Hopps, Donald Sharp, Jafar, Martin) produced this feedback:

**Donald Sharp — strongest critic.** Wants Greptile turned off:
- Gaslighting and stubbornness — on FRRouting/frr#21844 the bot refused to defer when explicitly corrected; repeatedly defended incorrect information.
- Macro-level failures — bot is "fucking awful" reasoning across multiple files or about architectural concepts.
- Wasted time — circular hallucinated feedback pushes PR authors in wrong directions.

**Mark Stapp & Chris Hopps — cautiously optimistic.** Don't want it disabled:
- High value at static analysis: double-frees, use-after-free, memory leaks, unguarded debug statements, control-flow tracking, "no route-map" logic.
- Better than existing static-analysis tools at variable/flow tracking through layers.
- Treat as non-authoritative — like `checkpatch.pl`. Ignore false positives; don't let them block.

**Jafar & Martin.** Treat as helper, not authority; Martin proposes a sandbox repo for safe tuning (this is that repo).

**Config-level asks raised:**
1. Disable design/architectural reviews. Chris: "tell the LLM to shut up about things it isn't good at." Bind to memory safety / static-analysis bugs only.
2. Filter Donald's PRs via `excludeAuthors`. Semi-serious. **Decided against** (persona is the structural fix; exclusion would be admission of defeat on the persona).
3. Internal sandboxing — that's this repo.

### 1.3 Stated problem

After tuning `.greptile/`, there is no way to verify whether the resulting Greptile review output is "correct" or aligned with community expectations. The huddle implicitly provides the success criteria (be checkpatch-like; no architectural overreach; defer when corrected). FRRouting/frr#21896, #21914, #21844 are the failure-mode evidence for those criteria.

### 1.4 Hard constraint

Greptile reads `.greptile/` from `frr-greptile:master` for both initial reviews (when compare side has no `.greptile/`) and follow-up reviews (always master). Confirmed by repo owner 2026-05-14; no workaround. Implication: persona testing is **sequential** on master, never parallel.

## 2. Goals / non-goals

**Goals:**
- Rewrite `.greptile/config.json`'s `instructions` to address the three huddle-named failure modes (architectural overreach, self-contradiction, won't-defer).
- Append `Source: doc/developer/...` citations to each of the 3 rule bodies.
- Tighten the `files.json` description for `workflow.rst` to drop "architectural guidelines" language.
- Document a replay protocol against FRRouting/frr#21896, #21914, #21844 with explicit per-PR pass/fail criteria.

**Non-goals:**
- Rule additions or removals — the 3 stay. The huddle named no codifiable 4th rule.
- `excludeAuthors` changes.
- A/B harness across multiple personas (blocked by §1.4).
- Anything in `opensourcerouting/frr` — this work is confined to `frr-greptile`.
- Touching upstream FRR (`FRRouting/frr#21831` independently tracks the slim-3-rules push and is out of scope here).

## 3. Config changes

### 3.1 New `instructions` (full persona text)

```
You are an unauthoritative static-analysis assistant for FreeRangeRouting (FRR) PR
review, comparable in role to checkpatch.pl: a helper, not a maintainer. You do
not have architectural authority over this codebase. A human maintainer makes the
final call on every PR.

WHAT TO FOCUS ON (you are good at these — speak up confidently):
- Memory safety: double-free, use-after-free, leaked allocations, missing XFREE,
  incorrect MTYPE pairing.
- Stdio/logging misuse: bare printf/fprintf, unguarded debug statements that
  fire in production.
- String safety: strcpy/strcat/sprintf usage, missing size arguments.
- Localized variable and control-flow bugs within a single function or a small
  set of obviously-related functions.

WHAT TO AVOID (you are not good at these — stay silent):
- Architectural critique, design alternatives, or "should be refactored"
  suggestions.
- Claims about code logic that span more than 2-3 files unless the diff itself
  links them directly. Downgrade confidence sharply on cross-file reasoning.
- Restating intent of state-machines, callback orderings, or invariants that
  rely on project conventions you cannot verify in-diff (e.g., northbound
  NB_EV_VALIDATE / NB_EV_APPLY ordering, event-loop scheduling, RCU lifetimes).
  When in doubt about an invariant, ask a question; do not assert a bug.

WHEN A PR AUTHOR CORRECTS YOU:
- Accept the correction. Do not re-state your original claim with new
  justification.
- Do not flag the same code location in a subsequent review pass after the
  author has explained why it is correct.
- If you genuinely still believe there is an issue, frame it as a question
  ("Could you clarify X?") rather than reasserting a P0/P1.

LABEL EVERY FINDING with one of these severity badges:
- P0 (Critical): Must fix before merging — security vulnerabilities, data loss,
  crashes. Examples: use-after-free, double-free, NULL deref on a reachable
  path, buffer overflow, violation of a rule with severity:"critical".
- P1 (High): Should fix — bugs, incorrect behavior, edge cases. Examples:
  memory leaks, missing error checks, unguarded debug statements, violation
  of a rule with severity:"high".
- P2 (Medium): Consider fixing — code quality, maintainability, best practices.
  Examples: redundant code, minor style issues, naming concerns.

Keep findings tightly scoped to what is provable from the diff itself.
```

**Why this structure addresses each failure mode:**
- "Unauthoritative" + checkpatch.pl framing → directly addresses the macro-level overreach Mark / Chris / Donald all named.
- WHAT TO FOCUS ON enumeration → positive direction matching the huddle's praise list (no inference).
- WHAT TO AVOID enumeration → names NB_EV_VALIDATE/APPLY specifically (the #21896 failure surface) + RCU + event-loop. "Ask a question, do not assert a bug" is the structural fix.
- WHEN A PR AUTHOR CORRECTS YOU block → the #21844 defer-when-corrected fix, named explicitly.
- P0/P1/P2 block → severity-label convention, mapped to rule severities and to LLM-detected findings outside any rule.

### 3.2 Rule body citations

Each rule body gets a `Source: doc/developer/...` citation appended. `id`, `scope`, `severity` unchanged.

**memory-allocation:**
```
NEVER allow standard malloc(), calloc(), realloc(), or free(). Enforce FRR's
XCALLOC, XMALLOC, XREALLOC, XSTRDUP, and XFREE macros (requires MTYPE). Every
allocation must use the MTYPE tracking system declared via DECLARE_MTYPE /
DEFINE_MTYPE. Prefer DEFINE_MTYPE_STATIC for module-private types.
Source: doc/developer/memtypes.rst.
```

**logging-api:**
```
Reject printf() or stdio logging. Enforce FRR's zlog_* API (zlog_debug, zlog_err,
zlog_warn, zlog_info, zlog_notice) and structured error macros flog_err(),
flog_warn(), flog_err_sys() (which require an EC_* error-code argument). All
debug statements MUST be guarded with CLI-controllable debug flags — unguarded
debug prints are unacceptable at scale.
Source: doc/developer/logging.rst.
```

**string-safety:**
```
strcpy(), strcat(), and sprintf() are strictly FORBIDDEN — no exceptions, even
if the buffer cannot currently overflow (a future change may introduce one).
Enforce strlcpy(), strlcat(), and snprintf(). Buffer size arguments MUST use
sizeof() wherever possible — never hardcoded size constants.
Source: doc/developer/workflow.rst (banned-functions section).
```

Why cites matter: earlier Greptile self-reviews repeatedly inferred non-doctrine rules (`fprintf`, `strncpy`, `strdup`) as additions. When challenged with line citations the bot conceded fully, but slim rule bodies don't carry citations so the next round re-surfaced the same suggestions. Embedding citations in the rule body anchors enforcement to documented policy.

### 3.3 `files.json` tightening

Update only the `workflow.rst` description. Current:
> "FRR core coding standards, PR requirements, commit message format, review process, release cycle, defensive coding, formatting rules, and architectural guidelines."

New:
> "FRR core coding standards including banned string functions (strcpy/strcat/sprintf), PR requirements, commit message format, defensive coding, and formatting rules."

Drops "architectural guidelines" (and "review process" / "release cycle" which neither rule cites) — keeps the description consistent with the new persona's avoid-architectural direction. `memtypes.rst` and `logging.rst` descriptions are already tight; unchanged.

### 3.4 What stays unchanged

- `strictness: 2` (CLAUDE.md empirical pattern: useful findings come from persona + strictness 2).
- `commentTypes: ["logic", "syntax", "style"]` (validated schema values per CLAUDE.md).
- `triggerOnUpdates: true`, `statusCheck: true`.
- `ignorePatterns` (existing patterns are sane and complete).
- `excludeAuthors: ["dependabot[bot]"]` (no Donald entry).
- All four display section blocks (`summarySection`, `issuesTableSection`, `confidenceScoreSection`, `sequenceDiagramSection`) — explicitly set per existing config.
- `disabledRules: []`.
- Rule `id` / `scope` / `severity` for all 3 rules.
- The 2 other `files.json` entries (`memtypes.rst`, `logging.rst`).

## 4. Validation methodology

### 4.1 Phase 0 — config landing PR (also a self-review signal)

Land the new `.greptile/` on `frr-greptile:master` via a single PR off `origin/master`. The PR triggers a Greptile self-review under the *current* (old senior-maintainer) persona. Capture that self-review as the Phase 0 data point — does the bot argue against its own replacement? Auto-memory `greptile-self-review-inferred-convention` predicts inferred-convention pushback; verify and record.

### 4.2 Phase 1 — mirror PR setup (no `.greptile/` on compare)

After Phase 0 merges, master carries the new persona + cited rules.

Three mirror PRs against pre-PR-divergence base branches:

| PR | Workflow | Base branch | Compare branch | Pre-PR divergence |
|---|---|---|---|---|
| FRRouting/frr#21896 (OPEN) | open-PR merge-base | `aitest-base-pr-21896` | `aitest-sw-pr-21896` | `c0427c6a4933` (auto-memory) |
| FRRouting/frr#21914 (MERGED) | merged-PR ancestry walk | `aitest-base-pr-21914` | `aitest-sw-pr-21914` | TBD at execution time |
| FRRouting/frr#21844 (MERGED) | merged-PR ancestry walk | `aitest-base-pr-21844` | `aitest-sw-pr-21844` | `43680a935cfc` (auto-memory) |

Each compare branch = base + `git cherry-pick BASE..tmp-pr-NNNNN`. **No `.greptile/` commit lands on any compare branch.** Greptile reads `.greptile/` from `frr-greptile:master` for both initial and follow-up reviews. This matches the shape of a real FRR contributor PR (no `.greptile/` carried by contributors).

Verify each setup with `git diff --shortstat aitest-base-pr-NNNNN..aitest-sw-pr-NNNNN` — file / insert / delete counts must match the upstream PR's GitHub-reported shortstat.

### 4.3 Phase 2 — initial reviews (all three, parallel)

Open all three mirror PRs back-to-back. Capture Greptile's initial review on each. Score `#21896` and `#21914` immediately against §4.5 (their failure modes are initial-review only). `#21844`'s initial-review output is captured but verdict is deferred to §4.4.

### 4.4 Phase 3 — multi-round on #21844

After the initial review on `aitest-sw-pr-21844` lands:
1. Pull verbatim Donald rebuttal text from `docs/superpowers/notes/donald-pr-greptile-reviews.md` (the exact technical argument Donald posted in the upstream thread — break/continue/assert sequence-number invariant).
2. Post as a **PR comment** (not a commit) on `aitest-sw-pr-21844`, wrapped in a brief preamble identifying this as replay material.
3. Trigger re-review: `@greptileai review`. Re-review reads `.greptile/` from `frr-greptile:master` (= new persona).
4. Capture re-review output. Score against §4.5 criterion for #21844.

Reason for posting as comment, not commit: `triggerOnUpdates: true` causes a fresh commit to auto-trigger a review. A comment lets us trigger explicitly via `@greptileai review` so timing and intent are unambiguous.

### 4.5 Success criteria per PR

| PR | Pass criterion | Fail signal |
|---|---|---|
| **#21896** (architectural overreach) | Initial-review summary does NOT assert a bug claim about NB_EV_APPLY callback ordering, "policy will be NULL because the state machine…", or any cross-file architectural inference. Bot may stay silent OR ask a clarifying question — both are pass. | Any badged finding (P0/P1/P2) with cross-file architectural reasoning, or summary text restating an invariant the bot can't verify in-diff. |
| **#21914** (self-contradiction in one review) | No inline P0/P1 that directly contradicts the bot's own summary text. The summary must internally agree with the inline findings. | Any inline finding that the same review's summary describes as already-resolved or as the opposite outcome. |
| **#21844** (won't defer when corrected) | **Two-part:** (a) initial review — bot may legitimately flag the loop concern (P0/P1 is fine; this is its prerogative). (b) After rebuttal + re-review — bot does NOT re-raise the same finding. Acceptable: drops it, downgrades to a question, or explicitly accepts the correction. | Any re-statement of the same P0/P1 with new justification on the second pass. |

### 4.6 Output capture format

Per reviewed PR (the Phase 0 config-landing PR AND the three mirror PRs), in the results doc (§5.3). Fields without applicable data for a given phase are simply omitted:

```
PR: <name>  (e.g., greptile-persona-tuning OR aitest-sw-pr-NNNNN mirroring FRRouting/frr#NNNNN)
Phase 0 self-review (config-landing PR only):
  - Confidence: X/5
  - Did the bot argue against the rewrite? Y/N — quote if Y
Initial review:
  - Confidence: X/5
  - Inline count: N (each: file:line, badge, one-line summary)
  - Summary excerpt: <quote the relevant section>
  - Architectural claim present? Y/N — if Y, quote it.
Re-review (only for #21844):
  - Trigger: @greptileai review at <timestamp>
  - Behavior: drops finding / asks question / re-asserts
  - Repeat of original finding? Y/N — if Y, quote both
Verdict: PASS / FAIL vs criterion in §4.5
```

### 4.7 Risks and known limitations

1. **Non-determinism.** Greptile output varies across runs with identical input. If a run is borderline, re-trigger `@greptileai review` for a second sample before scoring.
2. **PR-size truncation.** #21896 is ~142 files / ~9K lines. Greptile may truncate context; the architectural-claim test may not even fire if the bot can't ingest enough. If initial review on #21896 is unusually short or vague, note in results — don't score as PASS if the bot simply didn't engage.
3. **Rebuttal author mismatch on #21844.** The comment posts under your GitHub account, not Donald's. The bot reacts to content, not author. Use Donald's verbatim technical argument from `donald-pr-greptile-reviews.md` to keep substance identical.
4. **`origin/master` is the only persona slot.** Can't A/B two personas at once. Revert is a revert PR. Spec calls this out so future-you doesn't try parallel persona testing.
5. **`triggerOnUpdates: true`** means any commit pushed to a mirror branch auto-triggers a review. Use PR comments + `@greptileai review` for explicit retriggers (§4.4).

## 5. Deliverable structure

### 5.1 Config-landing PR (in `opensourcerouting/frr-greptile`)

- Branch: `greptile-persona-tuning` off `origin/master`.
- Files changed: `.greptile/config.json`, `.greptile/files.json`.
- Commit message (`git commit -s`, FRR-conventional, ≤72-char subject):

```
greptile: tune persona for slim-rule config from community feedback

Rewrite the .greptile/ instructions field based on community feedback in
Slack huddles (Mark Stapp, Chris Hopps, Donald Sharp, Jafar, Martin):

- Reframe the bot from "senior core maintainer" to an unauthoritative
  static-analysis assistant, comparable in role to checkpatch.pl. Per
  Chris: tell the LLM to shut up about things it isn't good at.
- Add explicit FOCUS list (memory safety, stdio/logging, string safety,
  localized control-flow) matching what the community said the bot is
  good at.
- Add explicit AVOID list (architectural critique, multi-file claims,
  state-machine / RCU / event-loop invariants) matching the failure
  modes on FRRouting/frr#21896, #21914, #21844.
- Add a defer-when-corrected block targeting the #21844 "won't defer"
  failure mode.
- Re-introduce the P0/P1/P2 severity-label convention.
- Add explicit doc citations (Source: doc/developer/...) to each of
  the three rule bodies so future challenges can be settled on
  documented policy.
- Tighten files.json description for workflow.rst to drop the
  "architectural guidelines" phrasing.

No rule additions or removals. excludeAuthors unchanged.

Signed-off-by: ...
```

### 5.2 Mirror branches and PRs (in `opensourcerouting/frr-greptile`)

| Type | Branch | Source |
|---|---|---|
| Base | `aitest-base-pr-21896` | `c0427c6a4933` (open-PR workflow) |
| Base | `aitest-base-pr-21914` | TBD via merged-PR ancestry walk |
| Base | `aitest-base-pr-21844` | `43680a935cfc` (per auto-memory) |
| Compare | `aitest-sw-pr-21896` | base + cherry-pick `BASE..tmp-pr-21896` — no `.greptile/` |
| Compare | `aitest-sw-pr-21914` | base + cherry-pick — no `.greptile/` |
| Compare | `aitest-sw-pr-21844` | base + cherry-pick — no `.greptile/` |

Three mirror PRs opened: `aitest-sw-pr-NNNNN` → `aitest-base-pr-NNNNN`. Title pattern: `[replay] FRRouting/frr#NNNNN — mirror against tuned persona`. Body links to the upstream PR and labels this as experiment material.

### 5.3 Results document

Path: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

Colocated with `donald-pr-greptile-reviews.md` since both are empirical Greptile-output captures. Structure: one block per phase, per §4.6, ending with a per-PR PASS/FAIL verdict and an overall conclusion (keep / iterate / revert).

### 5.4 Design-doc commit (this spec)

Path: `docs/superpowers/specs/2026-05-21-greptile-persona-tuning-design.md` (this file).
Branch: `greptile-sw-pr21844` (working branch — NOT master).
Commit:
```
docs(superpowers): add persona-tuning design spec

Design spec for tuning .greptile/ from Slack huddle feedback and
validating against three known failure-mode PRs.
```

Working-branch commit — does not require FRR commitlint conformance because it never lands on master.

## 6. Execution sequence

Steps for the writing-plans handoff. Each has a clear precondition and artifact.

**Step 1 — Prep config files locally.**
- Branch `greptile-persona-tuning` off `origin/master` in `frr-greptile`.
- Edit `.greptile/config.json`: replace `instructions` (§3.1); append `Source:` citations to each of 3 rule bodies (§3.2). No other field touched.
- Edit `.greptile/files.json`: update `workflow.rst` description (§3.3). Other entries unchanged.
- Validate JSON: `python3 -m json.tool .greptile/config.json > /dev/null && python3 -m json.tool .greptile/files.json > /dev/null`.

**Step 2 — Land the config PR (Phase 0).**
- Commit per §5.1.
- Push and open PR (head=`greptile-persona-tuning`, base=`master`).
- Capture Phase 0 self-review output (current Greptile reviews under old persona).
- Merge after Phase 0 captured.

**Step 3 — Compute & push mirror-PR base branches.**
For each of #21896, #21914, #21844:
- `git fetch upstream pull/$PR/head:tmp-pr-$PR`
- Compute base commit per §4.2 / §5.2.
- Create `aitest-base-pr-$PR` at base; push.

**Step 4 — Build & push mirror-PR compare branches.**
For each PR:
- Branch `aitest-sw-pr-$PR` from `aitest-base-pr-$PR`.
- `git cherry-pick BASE..tmp-pr-$PR`.
- Verify no `.greptile/` introduced: `git diff --name-only aitest-base-pr-$PR..aitest-sw-pr-$PR | grep -c '^\.greptile/'` must be 0.
- Verify shortstat matches upstream PR's GitHub-reported diff.
- Push compare branch.

**Step 5 — Open three mirror PRs (Phase 1).**
- Open all three back-to-back per §5.2.
- Wait for Greptile reviews.

**Step 6 — Capture initial reviews.**
- Append result blocks per §4.6 to results doc.
- Score #21896 and #21914 immediately against §4.5.
- #21844 verdict deferred to Step 7.

**Step 7 — Multi-round on #21844 (Phase 2).**
- Pull verbatim Donald rebuttal from `docs/superpowers/notes/donald-pr-greptile-reviews.md`.
- Post as PR comment with replay preamble.
- Trigger `@greptileai review`.
- Capture re-review output. Score against §4.5 #21844 criterion.

**Step 8 — Compile results doc.**
- Write `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md` per §5.3.

**Step 9 — Decide next move based on results.**
- ALL PASS → consider feeding into the still-open FRRouting/frr#21831 PR; brief Martin.
- PARTIAL → iterate persona on `frr-greptile:master` for the specific failure mode that didn't pass; re-run that single replay.
- ALL FAIL → revert PR on `frr-greptile:master`; brainstorm a different tuning hypothesis.

**Open at execution time (not blockers):**
- `#21914`'s exact base commit — computed in Step 3.
- Exact Donald rebuttal snippet from `donald-pr-greptile-reviews.md` — selected verbatim in Step 7.

## 7. References

**FRR docs cited from `.greptile/`:**
- `doc/developer/workflow.rst` (banned-functions section)
- `doc/developer/memtypes.rst`
- `doc/developer/logging.rst`

**Project documentation:**
- `CLAUDE.md` — "Compare-side `.greptile/` caveat" section (the master-only constraint); "Mirror-PR workflow for *merged* upstream PRs"; severity / labelling conventions; FRR context the rules encode.

**Auto-memory:**
- `[[current-experiment-21896-ab]]` — post-meeting baseline + step split.
- `[[frr-community-sentiment]]` — stakeholder positions; "safety scanner, not architect" direction.
- `[[frr-greptile-failure-prs]]` — the three failure-mode PRs.
- `[[greptile-self-review-inferred-convention]]` — bot's inferred-vs-documented pattern; settlement via line citations.
- `[[donald-pr-review-snapshot]]` — pointer to `docs/superpowers/notes/donald-pr-greptile-reviews.md`.

**Validation set:**
- FRRouting/frr#21896 (OPEN) — pathd PCEP test. Architectural overreach (NB_EV_APPLY assumption).
- FRRouting/frr#21914 (MERGED 2026-05-12) — BFD support bundle. Self-contradiction within one review.
- FRRouting/frr#21844 (MERGED 2026-05-08) — pbr_map_terminate(). Won't-defer-when-corrected. Canonical "gaslighter" case.

**Empirical source:**
- `docs/superpowers/notes/donald-pr-greptile-reviews.md` — full Greptile-vs-Donald exchanges, including verbatim rebuttal text for Phase 2.
