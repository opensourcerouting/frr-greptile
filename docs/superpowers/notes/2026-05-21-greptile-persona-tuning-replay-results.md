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
  - **Bot claims a hidden "Additional Instructions" admin-level configuration field outside `.greptile/config.json`.** This needs human verification — Greptile's claims about its own config have been historically unreliable (see CLAUDE.md "Reviewing Greptile-on-Greptile feedback" and memory `greptile-self-review-inferred-convention`). The `.db-rules.md` filename is internal Greptile state, not in our repo.
  - If the claim is REAL, Phase 1/2 reviews on the mirror PRs will be inconclusive: any failure-mode behavior could be explained as the conflicting `instruction-0` still being active, not as the new persona failing. Validation is blocked until the admin field is verified and (if needed) updated.
  - If the claim is HALLUCINATED, Phase 1/2 proceed as planned.

**Status: BLOCKED pending user verification of the "Additional Instructions" field claim.**
