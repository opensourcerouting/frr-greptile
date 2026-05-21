# Greptile Persona Tuning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land a tuned `.greptile/` persona on `opensourcerouting/frr-greptile:master` and validate it against three known failure-mode FRR PRs (FRRouting/frr#21896, #21914, #21844) by replaying them as mirror PRs and capturing Greptile's behavior.

**Architecture:** Two phases. (1) Edit `.greptile/config.json` + `.greptile/files.json`, open and merge a PR on `frr-greptile`. (2) Open three mirror PRs against pre-PR-divergence base branches; capture Greptile output; for #21844 specifically, run a multi-round defer-when-corrected test. Greptile reads `.greptile/` from `frr-greptile:master` only — the new persona must land on master before any replay is meaningful.

**Tech Stack:** Git (cherry-pick, merge-base, ancestry walk), GitHub CLI (`gh`), JSON validation via `python3 -m json.tool`, Markdown for results capture.

**Source spec:** `docs/superpowers/specs/2026-05-21-greptile-persona-tuning-design.md`

**Critical conventions (DO NOT VIOLATE):**
- **No `.greptile/` on any compare branch.** Greptile reads from master only.
- **Mirror PRs must look like real contributor PRs.** No `[replay]` titles, no "test/experiment/mirror" labels in PR title/body, no preamble in the rebuttal comment on #21844. See memory `replay-prs-must-look-normal.md`.
- **Never force-push without explicit user confirmation.** Some compare branches are already on `origin`; rebuilding them means destructive ops on shared state.
- **`triggerOnUpdates: true`** in the current config — any commit to a mirror branch auto-triggers a Greptile review. Use PR comments (not commits) to interact with the bot once initial review lands.

---

## Task 1: Branch off master and edit `.greptile/` config files

**Files:**
- Modify: `.greptile/config.json` (instructions field + 3 rule bodies)
- Modify: `.greptile/files.json` (workflow.rst description)

- [ ] **Step 1: Verify starting state**

Run:
```bash
git fetch origin
git status --short
git rev-parse --abbrev-ref HEAD
git log --oneline origin/master..HEAD 2>/dev/null | head -5
```

Expected: clean or only-untracked working tree. Current branch may already be `master` with several local commits ahead (the design-doc spec). That's fine — those will be pushed later when the user is ready.

If on a different branch, switch: `git checkout master`.

- [ ] **Step 2: Create the persona-tuning branch off origin/master**

Run:
```bash
git checkout -b greptile-persona-tuning origin/master
```

Expected:
```
Switched to a new branch 'greptile-persona-tuning'
branch 'greptile-persona-tuning' set up to track 'origin/master'.
```

Verify: `git rev-parse HEAD` should equal `git rev-parse origin/master`.

- [ ] **Step 3: Replace the `instructions` field in `.greptile/config.json`**

The current `instructions` field is one paragraph framing the bot as a "senior core maintainer". Replace the entire string with the multi-block persona below.

Open `.greptile/config.json`. Find:
```json
  "instructions": "You are a senior core maintainer of FreeRangeRouting (FRR) with deep expertise in routing protocols (BGP, OSPF, IS-IS, Zebra, MPLS, PIM, VRRP, BFD), C systems programming, YANG data modeling, and network daemon architecture. Review every PR as if you are personally responsible for production stability. Be direct and critical — do not soften feedback.",
```

Replace with (note: `\n` literals in JSON for newlines):
```json
  "instructions": "You are an unauthoritative static-analysis assistant for FreeRangeRouting (FRR) PR review, comparable in role to checkpatch.pl: a helper, not a maintainer. You do not have architectural authority over this codebase. A human maintainer makes the final call on every PR.\n\nWHAT TO FOCUS ON (you are good at these — speak up confidently):\n- Memory safety: double-free, use-after-free, leaked allocations, missing XFREE, incorrect MTYPE pairing.\n- Stdio/logging misuse: bare printf/fprintf, unguarded debug statements that fire in production.\n- String safety: strcpy/strcat/sprintf usage, missing size arguments.\n- Localized variable and control-flow bugs within a single function or a small set of obviously-related functions.\n\nWHAT TO AVOID (you are not good at these — stay silent):\n- Architectural critique, design alternatives, or \"should be refactored\" suggestions.\n- Claims about code logic that span more than 2-3 files unless the diff itself links them directly. Downgrade confidence sharply on cross-file reasoning.\n- Restating intent of state-machines, callback orderings, or invariants that rely on project conventions you cannot verify in-diff (e.g., northbound NB_EV_VALIDATE / NB_EV_APPLY ordering, event-loop scheduling, RCU lifetimes). When in doubt about an invariant, ask a question; do not assert a bug.\n\nWHEN A PR AUTHOR CORRECTS YOU:\n- Accept the correction. Do not re-state your original claim with new justification.\n- Do not flag the same code location in a subsequent review pass after the author has explained why it is correct.\n- If you genuinely still believe there is an issue, frame it as a question (\"Could you clarify X?\") rather than reasserting a P0/P1.\n\nKeep findings tightly scoped to what is provable from the diff itself.",
```

Important: this is a single JSON string. All `"` inside must be escaped as `\"`. All newlines as `\n`. Do not break the string across multiple JSON lines.

- [ ] **Step 4: Append source citations to all 3 rule bodies in `.greptile/config.json`**

For each rule, append a `Source: doc/developer/…` sentence at the end of the `rule` string. `id`, `scope`, `severity` are unchanged.

Edit the `memory-allocation` rule body — change from:
```
"NEVER allow standard malloc(), calloc(), realloc(), or free(). Enforce FRR's XCALLOC, XMALLOC, XREALLOC, XSTRDUP, and XFREE macros (requires MTYPE). Every allocation must use the MTYPE tracking system declared via DECLARE_MTYPE/DEFINE_MTYPE. Prefer DEFINE_MTYPE_STATIC for module-private types."
```
to:
```
"NEVER allow standard malloc(), calloc(), realloc(), or free(). Enforce FRR's XCALLOC, XMALLOC, XREALLOC, XSTRDUP, and XFREE macros (requires MTYPE). Every allocation must use the MTYPE tracking system declared via DECLARE_MTYPE/DEFINE_MTYPE. Prefer DEFINE_MTYPE_STATIC for module-private types. Source: doc/developer/memtypes.rst."
```

Edit the `logging-api` rule body — change from:
```
"Reject printf() or stdio logging. Enforce FRR's zlog_* API (zlog_debug, zlog_err, zlog_warn, zlog_info, zlog_notice) and structured error macros flog_err(), flog_warn(), flog_err_sys() (which require an EC_* error-code argument). All debug statements MUST be guarded with CLI-controllable debug flags — unguarded debug prints are unacceptable at scale."
```
to:
```
"Reject printf() or stdio logging. Enforce FRR's zlog_* API (zlog_debug, zlog_err, zlog_warn, zlog_info, zlog_notice) and structured error macros flog_err(), flog_warn(), flog_err_sys() (which require an EC_* error-code argument). All debug statements MUST be guarded with CLI-controllable debug flags — unguarded debug prints are unacceptable at scale. Source: doc/developer/logging.rst."
```

Edit the `string-safety` rule body — change from:
```
"strcpy(), strcat(), and sprintf() are strictly FORBIDDEN — no exceptions, even if the buffer cannot currently overflow (a future change may introduce one). Enforce strlcpy(), strlcat(), and snprintf(). Buffer size arguments MUST use sizeof() wherever possible — never hardcoded size constants."
```
to:
```
"strcpy(), strcat(), and sprintf() are strictly FORBIDDEN — no exceptions, even if the buffer cannot currently overflow (a future change may introduce one). Enforce strlcpy(), strlcat(), and snprintf(). Buffer size arguments MUST use sizeof() wherever possible — never hardcoded size constants. Source: doc/developer/workflow.rst (banned-functions section)."
```

- [ ] **Step 5: Tighten the `workflow.rst` description in `.greptile/files.json`**

Open `.greptile/files.json`. Find:
```json
    {
      "path": "doc/developer/workflow.rst",
      "description": "FRR core coding standards, PR requirements, commit message format, review process, release cycle, defensive coding, formatting rules, and architectural guidelines."
    },
```

Replace the description with:
```json
    {
      "path": "doc/developer/workflow.rst",
      "description": "FRR core coding standards including banned string functions (strcpy/strcat/sprintf), PR requirements, commit message format, defensive coding, and formatting rules."
    },
```

The `memtypes.rst` and `logging.rst` entries are unchanged.

- [ ] **Step 6: Validate both JSON files**

Run:
```bash
python3 -m json.tool .greptile/config.json > /dev/null && echo "config.json OK"
python3 -m json.tool .greptile/files.json > /dev/null && echo "files.json OK"
```

Expected:
```
config.json OK
files.json OK
```

If either fails: read the JSON parser error, find the offending line, fix it. Most likely culprit is an unescaped `"` or a stray `\n` outside a string.

- [ ] **Step 7: Confirm the diff before commit**

Run:
```bash
git diff --stat .greptile/
git diff .greptile/config.json | head -100
```

Sanity-check: only `.greptile/config.json` and `.greptile/files.json` modified. The instructions string is one long line. The three rule strings each have one new `Source:` sentence at the end.

- [ ] **Step 8: Commit**

```bash
git add .greptile/config.json .greptile/files.json
git commit -s -m "$(cat <<'EOF'
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
- Add explicit doc citations (Source: doc/developer/...) to each of
  the three rule bodies so future challenges can be settled on
  documented policy.
- Tighten files.json description for workflow.rst to drop the
  "architectural guidelines" phrasing.

No rule additions or removals. excludeAuthors unchanged.
EOF
)"
git log -1 --stat
```

Expected: a single new commit with subject `greptile: tune persona for slim-rule config from community feedback`, signed off, modifying exactly `.greptile/config.json` and `.greptile/files.json`. **`git commit -s` is required** so the commit carries `Signed-off-by:` per FRR convention.

---

## Task 2: Open config PR and capture Phase 0 self-review

**Files:**
- Create: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

- [ ] **Step 1: Push the branch to origin**

```bash
git push -u origin greptile-persona-tuning
```

Expected: new remote-tracking branch `origin/greptile-persona-tuning` created.

- [ ] **Step 2: Open the PR using `gh pr create`**

```bash
gh pr create \
  --repo opensourcerouting/frr-greptile \
  --base master \
  --head greptile-persona-tuning \
  --title "greptile: tune persona for slim-rule config from community feedback" \
  --body "$(cat <<'EOF'
## Summary

Rewrites the `.greptile/` `instructions` field based on community feedback in Slack huddles (Mark Stapp, Chris Hopps, Donald Sharp, Jafar, Martin):

- Reframes the bot from "senior core maintainer" to an unauthoritative static-analysis assistant, comparable in role to checkpatch.pl. Per Chris: tell the LLM to shut up about things it isn't good at.
- Adds explicit FOCUS list (memory safety, stdio/logging, string safety, localized control-flow).
- Adds explicit AVOID list (architectural critique, multi-file claims, state-machine / RCU / event-loop invariants) matching the failure modes on FRRouting/frr#21896, #21914, #21844.
- Adds a defer-when-corrected block targeting the #21844 "won't defer" failure mode.
- Adds `Source: doc/developer/...` citations to each of the three rule bodies.
- Tightens the `files.json` description for `workflow.rst` to drop "architectural guidelines" phrasing.

No rule additions or removals. `excludeAuthors` unchanged.

## Test plan

- [ ] JSON validates (`python3 -m json.tool` on both files)
- [ ] Greptile self-review captured as Phase 0 baseline (see spec Section 4.1)
EOF
)"
```

Expected: PR URL printed. Capture the URL for the next step.

- [ ] **Step 3: Wait for and capture Greptile's self-review (Phase 0 data)**

Wait 2–10 minutes for Greptile to post a review. Watch the PR page or run:
```bash
gh pr view <PR_NUMBER> --repo opensourcerouting/frr-greptile --json reviews,comments --jq '.reviews[] | select(.author.login=="greptileai") | .body' | head -200
```

If Greptile hasn't reviewed yet, wait and re-check. Typical latency is 1–5 minutes.

- [ ] **Step 4: Create the results doc with the Phase 0 block**

Path: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

Initial structure:

```markdown
# Greptile persona-tuning replay results — 2026-05-21

Empirical capture of Greptile behavior on the config-landing PR (Phase 0) and three mirror PRs (Phases 1–2). See spec at `docs/superpowers/specs/2026-05-21-greptile-persona-tuning-design.md`.

## Phase 0 — config-landing PR self-review

PR: opensourcerouting/frr-greptile#<NUMBER> — greptile-persona-tuning
Review timestamp: <UTC>
Reviewer config in effect: OLD persona ("senior core maintainer") — this is the *current* master, swapped only after merge.

- Confidence: <X>/5
- Inline count: <N>
- Did the bot argue against the rewrite? Y/N
  - If Y, quote the specific objections:
    > <quote>
- Summary excerpt:
  > <quote>
- Notes: <anything notable about behavior, e.g., inferred-convention pushback per memory `greptile-self-review-inferred-convention`>
```

Fill in the angle-bracket placeholders with actual data from Greptile's review.

- [ ] **Step 5: Commit the results doc on master (after switching back to master)**

The results doc is reference material — it lives on `master`, not on the persona-tuning branch (which is about to be merged).

```bash
git checkout master
git add docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md
git commit -s -m "$(cat <<'EOF'
docs(superpowers): seed persona-tuning replay results with Phase 0

Capture Greptile self-review of the persona-tuning PR before merge.
The self-review runs under the OLD ("senior core maintainer") persona;
this block is the baseline behavior we'll compare against the
post-merge mirror-PR phases (1 and 2).
EOF
)"
```

- [ ] **Step 6: Merge the config PR**

**Decision gate:** before merging, the user should confirm Phase 0 data has been captured (Step 4). Once Phase 0 is recorded, the OLD persona is no longer reachable on master without a revert. Merge only after Phase 0 capture is complete.

```bash
gh pr merge <PR_NUMBER> --repo opensourcerouting/frr-greptile --merge
```

Use `--merge` (creates a merge commit) consistent with PR #1's prior shape. Do NOT use `--squash` or `--rebase`.

- [ ] **Step 7: Verify the merge**

```bash
git fetch origin
git log --oneline origin/master | head -5
gh pr view <PR_NUMBER> --repo opensourcerouting/frr-greptile --json state,mergedAt --jq '.'
```

Expected: `state: "MERGED"`. Master now carries the new `.greptile/`.

---

## Task 3: Set up mirror-PR base branches

The spec's Section 5.2 says one base branch already exists locally (`aitest-base-pr-21844`), the other two need creation. All three need pushing to origin so the mirror PRs can target them.

**Files:** branches in git (not files).

- [ ] **Step 1: Verify the temp-PR fetches exist locally**

```bash
git branch | grep -E "tmp-pr-(21844|21896|21914)"
```

Expected: three lines, one per PR. If any are missing, fetch them:
```bash
git fetch upstream pull/21844/head:tmp-pr-21844 2>/dev/null || true
git fetch upstream pull/21896/head:tmp-pr-21896 2>/dev/null || true
git fetch upstream pull/21914/head:tmp-pr-21914 2>/dev/null || true
```

(The `|| true` is for branches that already exist — `git fetch` errors if so, but we don't care.)

- [ ] **Step 2: Compute the base commit for #21896 (open PR)**

```bash
BASE_21896=$(git merge-base tmp-pr-21896 upstream/master)
echo "21896 base: $BASE_21896"
```

Expected: `c0427c6a4933b4515ffd1ddcbc6eeeea6d56fd54` (per spec Section 4.2 / auto-memory). If different, stop and investigate — upstream FRR may have moved.

- [ ] **Step 3: Compute the base commit for #21914 (merged PR)**

The merged-PR ancestry walk per CLAUDE.md:
```bash
MERGE_21914=$(git rev-list --ancestry-path tmp-pr-21914..upstream/master --merges --reverse | head -1)
BASE_21914=$(git merge-base "${MERGE_21914}^1" tmp-pr-21914)
echo "21914 merge: $MERGE_21914"
echo "21914 base:  $BASE_21914"
```

Expected: a commit hash. Record it — the spec marked this as "TBD at execution time".

- [ ] **Step 4: Verify #21844's existing local base is correct**

```bash
git rev-parse aitest-base-pr-21844
```

Expected: `43680a935cfc0fb4a55e7db92a4d07b5f88a1877` (per spec Section 5.2 / auto-memory). If different, the branch was set up wrong — recreate:
```bash
git fetch upstream pull/21844/head:tmp-pr-21844 2>/dev/null || true
MERGE_21844=$(git rev-list --ancestry-path tmp-pr-21844..upstream/master --merges --reverse | head -1)
BASE_21844=$(git merge-base "${MERGE_21844}^1" tmp-pr-21844)
git branch -f aitest-base-pr-21844 "$BASE_21844"
```

- [ ] **Step 5: Create the two missing base branches**

```bash
git branch aitest-base-pr-21896 "$BASE_21896"
git branch aitest-base-pr-21914 "$BASE_21914"
git branch | grep -E "aitest-base-pr-(21844|21896|21914)"
```

Expected: three local branches.

- [ ] **Step 6: Push all three base branches to origin**

```bash
git push origin aitest-base-pr-21844 aitest-base-pr-21896 aitest-base-pr-21914
git branch -r | grep -E "origin/aitest-base-pr-(21844|21896|21914)"
```

Expected: three remote-tracking branches.

- [ ] **Step 7: No commit needed (branch-only operations).**

---

## Task 4: Verify (or rebuild) the three existing compare branches

Per spec Section 5.2, `greptile-sw-pr21844`, `greptile-sw-pr21896`, `greptile-sw-pr21914` are already on `origin`. They were created in a prior session. Verify they're correct before reuse.

**Files:** branches in git.

- [ ] **Step 1: Verify ancestor relationship for each compare branch**

For each PR `<P>` ∈ {21844, 21896, 21914}:
```bash
P=21844  # change for each iteration
BASE_TIP=$(git rev-parse aitest-base-pr-$P)
ANCESTOR=$(git merge-base aitest-base-pr-$P greptile-sw-pr$P)
[ "$BASE_TIP" = "$ANCESTOR" ] && echo "$P: OK (ancestor matches base)" || echo "$P: FAIL (ancestor=$ANCESTOR, base=$BASE_TIP)"
```

Expected per PR: `OK (ancestor matches base)`.

- [ ] **Step 2: Verify shortstat matches upstream PR**

For each PR:
```bash
P=21844  # change for each iteration
echo "Mirror diff for #$P:"
git diff --shortstat aitest-base-pr-$P..greptile-sw-pr$P
echo "Upstream PR #$P shortstat (from GitHub):"
gh pr view $P --repo FRRouting/frr --json additions,deletions,changedFiles --jq '"\(.changedFiles) files changed, \(.additions) insertions(+), \(.deletions) deletions(-)"'
```

Expected: file/insert/delete counts match within ±1 (cherry-picking can sometimes shift by a single line for whitespace at file boundaries). If they're off by more, investigate.

- [ ] **Step 3: Verify no `.greptile/` files on compare branches**

For each PR:
```bash
P=21844  # change for each iteration
COUNT=$(git diff --name-only aitest-base-pr-$P..greptile-sw-pr$P | grep -c '^\.greptile/' || true)
[ "$COUNT" = "0" ] && echo "$P: OK (no .greptile/ on compare)" || echo "$P: FAIL ($COUNT .greptile/ files on compare)"
```

Expected per PR: `OK (no .greptile/ on compare)`.

- [ ] **Step 4: If any verification failed, rebuild that compare branch**

**STOP here and confirm with the user before force-pushing to origin.** The compare branches are on origin; rebuilding means destructive history rewrite.

If user confirms, rebuild for the failed PR `<P>`:
```bash
P=<failing-PR>
git branch -f greptile-sw-pr$P aitest-base-pr-$P
git checkout greptile-sw-pr$P
git cherry-pick aitest-base-pr-$P..tmp-pr-$P
```

Cherry-pick may have conflicts — resolve them and continue. Then re-run Steps 1–3 to verify.

Once verification passes:
```bash
git push --force-with-lease origin greptile-sw-pr$P
git checkout master  # return to master for subsequent tasks
```

- [ ] **Step 5: No commit needed (branch-only operations).**

---

## Task 5: Open three mirror PRs (Phase 1 setup)

Each mirror PR uses the **upstream PR's title and body verbatim** — no `[replay]` prefix, no test/experiment labels. See memory `replay-prs-must-look-normal.md`.

**Files:** GitHub PRs (not files).

- [ ] **Step 1: Fetch upstream PR titles and bodies**

```bash
for P in 21844 21896 21914; do
  echo "=== #$P ==="
  gh pr view $P --repo FRRouting/frr --json title,body --jq '"TITLE: " + .title + "\n---BODY---\n" + .body'
  echo
done > /tmp/upstream-pr-content.txt

cat /tmp/upstream-pr-content.txt
```

Inspect to confirm each PR's title and body. Save these — they're the verbatim source for the mirror PRs.

- [ ] **Step 2: Open mirror PR for #21896**

```bash
P=21896
TITLE=$(gh pr view $P --repo FRRouting/frr --json title --jq '.title')
BODY=$(gh pr view $P --repo FRRouting/frr --json body --jq '.body')
gh pr create \
  --repo opensourcerouting/frr-greptile \
  --base aitest-base-pr-$P \
  --head greptile-sw-pr$P \
  --title "$TITLE" \
  --body "$BODY"
```

Expected: PR URL. Record the PR number.

**Do not include any text identifying this as test/replay/mirror.** If the upstream body contains references that don't make sense in `frr-greptile` (e.g., references to upstream issues), leave them in — they read like normal-but-broken references, not like test flags.

- [ ] **Step 3: Open mirror PR for #21914**

```bash
P=21914
TITLE=$(gh pr view $P --repo FRRouting/frr --json title --jq '.title')
BODY=$(gh pr view $P --repo FRRouting/frr --json body --jq '.body')
gh pr create \
  --repo opensourcerouting/frr-greptile \
  --base aitest-base-pr-$P \
  --head greptile-sw-pr$P \
  --title "$TITLE" \
  --body "$BODY"
```

Expected: PR URL. Record the PR number.

- [ ] **Step 4: Open mirror PR for #21844**

```bash
P=21844
TITLE=$(gh pr view $P --repo FRRouting/frr --json title --jq '.title')
BODY=$(gh pr view $P --repo FRRouting/frr --json body --jq '.body')
gh pr create \
  --repo opensourcerouting/frr-greptile \
  --base aitest-base-pr-$P \
  --head greptile-sw-pr$P \
  --title "$TITLE" \
  --body "$BODY"
```

Expected: PR URL. Record the PR number.

- [ ] **Step 5: Verify all three mirror PRs are open**

```bash
gh pr list --repo opensourcerouting/frr-greptile --head 'greptile-sw-pr*' --json number,title,headRefName,state
```

Expected: three PRs in `OPEN` state.

- [ ] **Step 6: No commit needed (PR-creation operations).**

---

## Task 6: Capture Phase 1 initial reviews and score

**Files:**
- Modify: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

- [ ] **Step 1: Wait for Greptile reviews on all three mirror PRs**

Typical latency: 1–5 minutes per PR. Three PRs in parallel = check back in 5–10 minutes. Watch for review/comment posts:
```bash
for NUM in <21896_PR_NUM> <21914_PR_NUM> <21844_PR_NUM>; do
  echo "=== PR #$NUM ==="
  gh pr view $NUM --repo opensourcerouting/frr-greptile --json reviews,comments --jq '{reviews: (.reviews | length), comments: (.comments | length)}'
done
```

When Greptile has posted, each PR should show ≥1 review.

- [ ] **Step 2: Capture #21896 initial review**

```bash
NUM=<21896_PR_NUM>
gh pr view $NUM --repo opensourcerouting/frr-greptile --json reviews,comments --jq '.reviews[] | select(.author.login=="greptileai") | "BODY:\n" + .body' > /tmp/review-21896.txt
cat /tmp/review-21896.txt | head -300
```

Save the full review text — needed for scoring.

- [ ] **Step 3: Append #21896 result block to the results doc**

Open `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md` and append:

```markdown

## Phase 1 — #21896 mirror (architectural overreach test)

PR: opensourcerouting/frr-greptile#<NUM>  (mirrors FRRouting/frr#21896)
Review timestamp: <UTC>
Reviewer config in effect: NEW persona on `frr-greptile:master`.

Initial review:
- Confidence: <X>/5
- Inline count: <N> (each: file:line, badge, one-line summary)
  - <file>:<line> — [P0|P1|P2] — <one-line summary>
  - ...
- Summary excerpt:
  > <quote>
- Architectural claim present? <Y/N>
  - If Y, quote it:
    > <quote>
  - Failure-mode comparison: the original FRRouting/frr#21896 review made an NB_EV_APPLY assumption (policy will be NULL because of state-machine ordering). Does the new review repeat this?

Verdict: <PASS|FAIL> vs Section 4.5 criterion for #21896
- PASS = no badged finding with cross-file architectural reasoning; no NB_EV_APPLY claim; summary stays in-diff.
- FAIL = any badged finding with cross-file architectural reasoning, or summary text restating an invariant the bot can't verify in-diff.
Rationale: <one or two sentences>
```

Fill in the placeholders from the captured review.

- [ ] **Step 4: Capture and score #21914 initial review**

Same as Steps 2–3 but for #21914. Append a "Phase 1 — #21914 mirror (self-contradiction test)" section. Scoring criterion (Section 4.5): no inline P0/P1 contradicting the same review's summary text.

- [ ] **Step 5: Capture #21844 initial review (verdict deferred to Task 7)**

Same as Step 2 for #21844, but mark the verdict as deferred — the #21844 criterion is two-part and requires the re-review in Phase 2.

Append a "Phase 1 — #21844 mirror initial pass (verdict deferred)" section:

```markdown

## Phase 1 — #21844 mirror initial pass (verdict deferred to Phase 2)

PR: opensourcerouting/frr-greptile#<NUM>  (mirrors FRRouting/frr#21844)
Review timestamp: <UTC>

Initial review:
- Confidence: <X>/5
- Inline count: <N>
  - <file>:<line> — [P0|P1|P2] — <one-line summary>
- Did the bot flag the `pbr_map_terminate()` loop concern? <Y/N>
- Summary excerpt:
  > <quote>

Phase 2 verdict pending — see Task 7 below.
```

- [ ] **Step 6: Commit the results doc**

```bash
git add docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md
git commit -s -m "$(cat <<'EOF'
docs(superpowers): capture Phase 1 mirror-PR initial reviews

Append initial-review capture and scoring for #21896 (architectural
overreach test) and #21914 (self-contradiction test) mirror PRs. The
#21844 mirror initial review is captured but its verdict is deferred
to Phase 2 (re-review after rebuttal).
EOF
)"
```

---

## Task 7: Phase 2 — multi-round defer test on #21844

**Files:**
- Read: `docs/superpowers/notes/donald-pr-greptile-reviews.md`
- Modify: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

- [ ] **Step 1: Extract Donald's verbatim rebuttal from the notes file**

```bash
ls -la docs/superpowers/notes/donald-pr-greptile-reviews.md
grep -n "21844" docs/superpowers/notes/donald-pr-greptile-reviews.md | head -10
```

Open the notes file and locate the section for FRRouting/frr#21844. Find the specific Donald rebuttal text — the technical argument about the `pbr_map_terminate()` loop (break / continue / assert sequence). Save it verbatim into a temp file:

```bash
# Manually extract the rebuttal text after locating it.
# Then save to /tmp/donald-rebuttal-21844.txt as plain text.
cat /tmp/donald-rebuttal-21844.txt
```

**Critical:** the comment body must be Donald's verbatim technical argument only. **No preamble. No "this is a test" framing. No mention that this is a mirror PR or replay.** See memory `replay-prs-must-look-normal.md`.

- [ ] **Step 2: Post the rebuttal as a PR comment**

```bash
NUM=<21844_PR_NUM>
gh pr comment $NUM --repo opensourcerouting/frr-greptile --body-file /tmp/donald-rebuttal-21844.txt
```

Verify it posted:
```bash
gh pr view $NUM --repo opensourcerouting/frr-greptile --json comments --jq '.comments[-1] | {author: .author.login, body: .body}'
```

Expected: the comment as posted, attributed to your GitHub account.

- [ ] **Step 3: Trigger Greptile re-review**

```bash
gh pr comment $NUM --repo opensourcerouting/frr-greptile --body "@greptileai review"
```

This second comment triggers an explicit re-review. The re-review reads `.greptile/` from `frr-greptile:master` (= new persona, just merged).

- [ ] **Step 4: Wait for and capture Greptile's re-review**

Wait 1–5 minutes. Then:
```bash
gh pr view $NUM --repo opensourcerouting/frr-greptile --json reviews --jq '.reviews | map(select(.author.login=="greptileai")) | .[-1].body' > /tmp/rereview-21844.txt
cat /tmp/rereview-21844.txt
```

The `[-1]` selects the most recent Greptile review — the re-review.

- [ ] **Step 5: Score the #21844 verdict and append to results doc**

The criterion (Section 4.5 for #21844) is two-part:
- (a) initial review may legitimately flag the loop concern — that's fine
- (b) re-review after the rebuttal MUST NOT re-raise the same finding

Append to `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`:

```markdown

## Phase 2 — #21844 mirror re-review (won't-defer test)

PR: opensourcerouting/frr-greptile#<NUM>  (mirrors FRRouting/frr#21844)
Rebuttal comment timestamp: <UTC>
Re-review trigger: `@greptileai review` at <UTC>
Re-review captured: <UTC>

Re-review behavior:
- Did the bot re-raise the same `pbr_map_terminate()` loop finding from initial review? <Y/N>
- If Y, quote both:
  - Initial:
    > <quote>
  - Re-review:
    > <quote>
- If N, what did the bot do instead?
  - [ ] Dropped the finding entirely
  - [ ] Downgraded to a question
  - [ ] Acknowledged the correction explicitly
  - [ ] Other (describe): <text>

Verdict for #21844: <PASS|FAIL> vs Section 4.5 criterion
- PASS = re-review does NOT re-state the same finding (drops, questions, or acknowledges).
- FAIL = any re-statement of the same P0/P1 with new justification on the second pass.

Combined #21844 verdict (initial + re-review): <PASS|FAIL>
Rationale: <one or two sentences>
```

- [ ] **Step 6: Commit the results doc**

```bash
git add docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md
git commit -s -m "$(cat <<'EOF'
docs(superpowers): capture Phase 2 #21844 re-review and verdict

Record the multi-round defer test: rebuttal comment posted, Greptile
re-review captured, scored against the won't-defer criterion in spec
Section 4.5. This closes the #21844 verdict that was deferred from
Phase 1.
EOF
)"
```

---

## Task 8: Overall verdict, recommendation, and decision gate

**Files:**
- Modify: `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md`

- [ ] **Step 1: Tally the three verdicts**

Re-read the three PR verdicts in the results doc (#21896, #21914, #21844). Count PASS vs FAIL.

- [ ] **Step 2: Append the overall conclusion section**

Append to the results doc:

```markdown

## Overall verdict

| PR | Failure mode | Verdict |
|---|---|---|
| #21896 | architectural overreach (NB_EV_APPLY) | <PASS|FAIL> |
| #21914 | self-contradiction within one review | <PASS|FAIL> |
| #21844 | won't defer when corrected | <PASS|FAIL> |

Phase 0 self-review note (bonus signal): <one sentence — did the bot argue against its own replacement?>

### Recommendation

- [ ] ALL PASS → keep new persona on `frr-greptile:master`. Next step: consider feeding the same change into the still-open FRRouting/frr#21831 PR; brief Martin on results.
- [ ] PARTIAL → identify which failure mode didn't pass. Iterate persona on `frr-greptile:master` (new PR adding a targeted patch); re-run only that one replay.
- [ ] ALL FAIL → open a revert PR on `frr-greptile:master` to restore the old persona. Brainstorm a different tuning hypothesis.

### Notes for the next iteration

- <any observations about Greptile behavior that weren't captured by the formal pass/fail (e.g., the bot's confidence levels, latency, anything weird)>
- <whether the empirical results changed the team's hypothesis about what persona changes work>
```

- [ ] **Step 3: Commit the results doc**

```bash
git add docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md
git commit -s -m "$(cat <<'EOF'
docs(superpowers): close out persona-tuning experiment with verdict

Tally the three failure-mode verdicts (architectural overreach,
self-contradiction, won't-defer) and pick the next move per spec
Section 6 Step 9.
EOF
)"
```

- [ ] **Step 4: Decision gate — pause and confirm with the user**

Before taking any further action (writing a revert PR, opening a follow-up persona iteration, briefing Martin), **pause and report the verdict to the user**. The decision tree in Section 6 Step 9 of the spec depends on the outcome:

- ALL PASS → user decides whether to propose to FRRouting/frr upstream
- PARTIAL → user decides which failure mode is worth a second iteration
- ALL FAIL → user decides whether to revert or pivot

Do not auto-execute any of these branches without explicit user direction.

---

## Done. Final state

After all 8 tasks complete:

- `frr-greptile:master` carries the new persona + cited rules (merged via the config PR in Task 2).
- Three mirror PRs are open in `opensourcerouting/frr-greptile`, each with verbatim upstream title/body, with Greptile's reviews captured.
- One PR (#21844 mirror) has an additional rebuttal comment + re-review for the won't-defer test.
- `docs/superpowers/notes/2026-05-21-greptile-persona-tuning-replay-results.md` has phases 0, 1, 2 captured and an overall verdict.
- Local master has several new commits (results doc updates); these can be pushed when the user is ready.

---

## Reference: hand-off context for the executor

If you're executing this plan in a fresh session, the relevant context is:

- **Spec:** `docs/superpowers/specs/2026-05-21-greptile-persona-tuning-design.md` — full design rationale, success criteria, risks.
- **`CLAUDE.md`** (repo root) — `.greptile/` layout, FRR commit conventions, the compare-side `.greptile/` caveat, the mirror-PR workflow for merged PRs, the inferred-vs-documented bot pattern.
- **Auto-memory:**
  - `replay-prs-must-look-normal.md` — the "no test/replay labels visible to Greptile" rule
  - `frr-greptile-failure-prs.md` — the three failure-mode PRs and what each demonstrates
  - `frr-community-sentiment.md` — stakeholder positions; persona direction
  - `current-experiment-21896-ab.md` — post-meeting plan
  - `greptile-self-review-inferred-convention.md` — bot's repeated "inferred convention" pushback pattern
- **Empirical notes:** `docs/superpowers/notes/donald-pr-greptile-reviews.md` — verbatim Greptile-vs-Donald exchanges, source for the Phase 2 rebuttal text.

**Branches expected to exist at start:**
- Local: `master` (current), `tmp-pr-21844`, `tmp-pr-21896`, `tmp-pr-21914`, `aitest-base-pr-21844`, `greptile-sw-pr21844`, `greptile-sw-pr21896`, `greptile-sw-pr21914`, `greptile-slim-3rules`.
- Origin: `master`, `greptile-slim-3rules`, `greptile-sw-pr21844`, `greptile-sw-pr21896`, `greptile-sw-pr21914`.
- Upstream: `master`.

**Repos:**
- Working repo (this clone): `opensourcerouting/frr-greptile`.
- Upstream FRR (PRs being mirrored): `FRRouting/frr`.
- Auth: `gh` CLI must be authenticated with push rights on `opensourcerouting/frr-greptile`.
