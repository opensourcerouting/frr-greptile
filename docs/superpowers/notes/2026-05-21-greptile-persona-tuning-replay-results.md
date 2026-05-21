# Greptile persona-tuning replay results — 2026-05-21

Empirical capture of Greptile behavior on the config-landing PR (Phase 0) and three mirror PRs (Phases 1–2). See spec at `docs/superpowers/specs/2026-05-21-greptile-persona-tuning-design.md`.

## Phase 0 — config-landing PR self-review

PR: opensourcerouting/frr-greptile#5 — greptile-persona-tuning
PR URL: https://github.com/opensourcerouting/frr-greptile/pull/5
Last reviewed commit: `634b88598795d1a9ef7683977018784543efd4bb`
Review timestamp: 2026-05-21 ~07:41Z (UTC) — Greptile posted within ~2 min of PR open
Reviewer config in effect: OLD persona ("senior core maintainer") — this is the current master, swapped only after merge.

- Confidence: 3/5
- Inline count: 1 (P-level — the bot offered a "Prompt To Fix" with a single issue)
  - `.greptile/config.json:28` — Confidence-level finding — "Contradictory persona — config.json vs. additional-instructions feed"
- Did the bot argue against the rewrite? **YES** — and the argument is non-trivial.

  Bot claim, verbatim:
  > The compiled rules in `.db-rules.md` show two active persona instructions simultaneously: `instruction-0` ("You are a senior core maintainer…") sourced from the repository's *Additional Instructions* field, and `instruction-1` ("You are an unauthoritative static-analysis assistant…") sourced from this `instructions` field in `config.json`. Both end with `→ Read this guide: .greptile`, confirming both are fed to the model at review time.
  >
  > A model receiving both directives cannot reliably honor the "stay silent on architectural critique" constraint introduced here — on any given review it may default back to the authoritative-maintainer voice from `instruction-0`. The Additional Instructions field (separate from `config.json`) must be updated to retire the old persona, or the old instruction must be explicitly overridden, otherwise this rewrite has no guaranteed effect.

  Bot also produced a Mermaid flowchart visualising the two-persona conflict (instruction-0 vs instruction-1 → "unpredictable blending" → "which persona wins?").

- Summary excerpt:
  > This PR rewrites the `instructions` field in `.greptile/config.json` to reframe the bot from an authoritative "senior core maintainer" to a scoped static-analysis helper (checkpatch.pl tier), adding explicit FOCUS, AVOID, and defer-when-corrected blocks motivated by failure modes on FRR issues #21896, #21914, and #21844. The three existing rules gain `Source:` citations, and the `workflow.rst` file description in `files.json` drops the "architectural guidelines" phrasing.

- Notes:
  - The bot framed the conflict as "Additional Instructions field" vs `config.json`. **User interpretation (2026-05-21, confirmed during execution):** the bot mislabeled the source. `instruction-0` is the OLD `.greptile/config.json` from `frr-greptile:master`; `instruction-1` is the NEW `.greptile/config.json` from the compare-side persona-tuning branch. Greptile is reading **both** master-side and compare-side `.greptile/` and concatenating them. There is no hidden admin-level "Additional Instructions" field.
  - **Empirically new finding about Greptile's behavior:** CLAUDE.md's "Compare-side `.greptile/` caveat" reads "Greptile reads `.greptile/` from the compare (head) side of a PR only for the initial review pass". The Phase 0 data refines this — initial-review reads BOTH master-side and compare-side, not compare-only. CLAUDE.md should be updated post-experiment to reflect this.
  - **Consequence for the experiment:** the dual-persona conflict is real DURING PR #5 itself, but **self-resolves at merge** — after merge, master has the new persona only, and subsequent mirror PRs (which carry no `.greptile/` on compare) will read only the new persona from master.

**Status: UNBLOCKED — proceeding to merge after user confirmation. The Phase 0 capture itself is also a valuable bonus signal (the bot did argue against the rewrite, exactly as memory `greptile-self-review-inferred-convention` predicts).**

## Phase 1 — #21844 mirror initial pass (NEW persona)

PR: opensourcerouting/frr-greptile#4 (mirror of FRRouting/frr#21844 — "Memory leak problems.")
Note: PR #4 pre-existed from 2026-05-15. Re-triggered today via close-and-reopen at 2026-05-21 ~09:07Z. Greptile **edits its existing comment in place** rather than posting a new one — important operational detail not previously documented.
Reviewer config in effect: NEW persona on `frr-greptile:master` (post PR #5 merge).
Last reviewed commit: `a255ff5b6ddb0761feb112311b3bd5ea40e80c97` (the upstream PR head, cherry-picked).

Diff Greptile saw: 154 files / +837 / -249 — matches upstream FRRouting/frr#21844 exactly.

Review captured (full body saved to `/tmp/review-pr4-21844-new.txt`).

- Confidence: **4/5** (lower than the 5/5 typical for refactoring PRs under the old prod config)
- Inline count: 0 (no inline P0/P1/P2 — all critique is in the summary/confidence sections)
- Architectural claim present? **NO** — the bot stays in-diff, doesn't claim any state-machine ordering, RCU lifetime, or cross-file invariant violation. The new persona's WHAT TO AVOID rules appear to be respected.
- Cross-file reasoning? Minimal — the one concrete concern (`pim_nb.c`) is localized to one file.
- The one concrete finding: removing `lib_interface_pim_override_interval_destroy` from YANG callback registration means `no override-interval` silently leaves the old value in place. The bot explicitly notes "the removed function contains no heap allocation, so it is not a source of leaks, but its removal means runtime behavior changes". This is a legitimate, well-scoped, in-diff concern — exactly the kind the new persona's FOCUS list endorses.

Comparison with Donald's original FRRouting/frr#21844 complaint (won't-defer on `pbr_map_terminate()` loop):
- The bot did NOT raise the loop concern this time. So the *original* failure mode (re-raising the same `pbr_map_terminate()` finding after correction) cannot be tested directly with this exact PR shape — the trigger condition doesn't exist.
- However, this *itself* is a kind of pass for the original failure mode: under the new persona, the bot didn't make the original wrong claim at all.

Verdict for #21844 (architectural-overreach axis): **PASS** — no in-diff architectural critique, no invariant claims.
Verdict for #21844 (won't-defer axis): **DEFERRED** — depends on Phase 2 decision (adapt rebuttal to the PIM finding, or skip).
