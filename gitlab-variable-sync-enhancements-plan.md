# GitLab Variable Sync — Enhancement Plan

Scope: `sync_github_variables_to_gitlab` step in `.github/workflows/provision-gitlab-mirror.yml`.
Baseline: `GL_<flags>__<name>` grammar; flags `M` masked, `P` protected, `E` expansion, `H` masked+hidden; expansion off by default (`raw:true`). Reserved keys published by the `sync_reserved_gitlab_variables` step. Stale project-level variables are pruned.

This plan addresses the ⚠️/❌ transitions found in the use-case analysis. Phases are ordered so cheap/foundational and safety work lands before the heavier hidden-variable work.

---

## Phase 1 — Ignore `E` on masked/hidden variables  (use cases C7, C8, F7)

**Problem.** `GL_ME__X` / `GL_HE__X` request masked/hidden *and* expansion. GitLab can't expand masked/hidden variables, and with expansion on the value is charset-restricted (`$` disallowed), so these fail or are functionally useless.

**Approach.** In the sync loop, derive effective expansion:
```js
const effectiveExpansion = expansion && !desiredMasked;   // desiredMasked = masked || hidden
const raw = !effectiveExpansion;
```
When `expansion` was requested but dropped, record it on the result (e.g. `noteExpansionIgnored = expansion && desiredMasked`) and surface a note in the summary / step log: *"E (expansion) ignored — not compatible with masked/hidden."*

**Touch points.** Loop `raw` derivation; per-row `result`; summary detail/notes.

**Risk.** Low. Verification: parse-sim for `GL_ME__`, `GL_HE__`, `GL_E__` (plain still expands).

---

## Phase 2 — Detect variable⇄secret key collisions  (use case N4)

**Problem.** The same resolved GitLab key from a GitHub Variable and a Secret produces two candidates → flip-flop churn each run (secret wins), or a persistent failure on hidden mismatch.

**Approach.** After building `toSync`, group by `glKey`. If a key resolves from more than one source/candidate:
- **Fail fast (recommended):** mark the key failed with a clear reason ("same GitLab key `FOO` produced by a GitHub variable and a secret — rename one"), skip syncing it, and surface in the summary.
- (Alternative: defined precedence, e.g. secret over variable, and note the skipped duplicate.)

**Touch points.** Post-collection dedupe pass; `failed`/`results`; summary.

**Risk.** Low. Verification: sim with duplicate `glKey` from both sources.

---

## Phase 3 — Pre-validate masking requirements + trim rules text  (use cases C3, V4, F1; prerequisite for Phase 4)

**Problem.** Masked/hidden values that are multiline, contain spaces, or are <8 chars fail at the API. We currently only surface the 400. Also `MASKING_RULES` still lists the Base64 charset, which no longer applies (expansion is off for masked/hidden after Phase 1).

**Approach.**
- Add `maskingViolation(value)` returning a reason or `null`: checks single line (no `\n`), no spaces, length ≥ 8.
- For masked/hidden candidates, pre-check before create/update; on violation, fail that variable with the friendly reason (no API call).
- Trim `MASKING_RULES` to: single line, no spaces, ≥8 chars, not matching an existing variable name. Drop the charset clause (or note it only applies if expansion is enabled, which we no longer allow for masked/hidden).

**Touch points.** New helper; sync-loop guard; `MASKING_RULES` constant; convention block wording.

**Risk.** Low. Must mirror GitLab's rules exactly to avoid false rejects. Verification: sim values (`ab c`, `short`, multiline, valid).

---

## Phase 4 — Hidden state change → delete + recreate  (use cases F8, F9)

**Problem.** Hidden is create-only and can't be removed. Adding/removing `H` on an existing key currently throws every run and never applies.

**Approach.** When `isHiddenNow !== hidden`:
1. **Pre-validate** the target state first (Phase 3). If the new state is hidden/masked and the value is invalid → do **not** delete; fail with a clear reason (protects against leaving no variable).
2. Otherwise: `DELETE` the existing key, then `POST` it fresh in the target state (hidden uses `masked_and_hidden:true`). We always have the value from GitHub, so no data is lost.
3. Report status `recreated` (new summary status), and note the brief non-atomic window.

**Touch points.** Replace the throw branch; add `recreated` to `STATUS_META`; summary counts/labels.

**Risk.** Medium — non-atomic delete→create; a concurrent pipeline could miss the variable briefly. Pre-validation prevents the "deleted but recreate failed" trap. Verification: sim state matrix (visible→hidden, hidden→visible, invalid value aborts before delete).

---

## Phase 5 — Hidden value rotation via hash sidecar  (use cases V5, V6)

**Problem.** Hidden values can't be read back, so we can't detect change → we re-push every run (churn), and it relies on the API allowing hidden value updates.

**Approach.**
- For each hidden variable `FOO`, maintain a companion **visible** variable `FOO__SRC_SHA` = `sha256(value)` (hash reveals nothing sensitive).
- Each run: compute `sha256` of the current GitHub value; compare to the sidecar.
  - equal → **unchanged**, skip (no churn);
  - differ → value rotated → push the new value (update, or delete+recreate if the GitLab version locks hidden values), then update the sidecar.
- Pruning must **whitelist** the `__SRC_SHA` suffix so sidecars aren't treated as stale.

**Touch points.** Hash helper (Node `crypto`); hidden-branch change detection; prune `desiredKeys`/whitelist; summary.

**Risk.** Medium — extra variable per hidden secret; must keep sidecar and prune in sync. Verification: sim equal/changed hash paths; confirm sidecar excluded from prune.

---

## Phase 6 — Parked / future (decide later)

- **Secrets default to masked+hidden (`H`).** Opt-in source-based default; more attractive now that hidden values accept most characters. Reverses "flags come only from the name."
- **File-type variables for multiline/space secrets.** PEM keys etc. can never be masked; map to `variable_type: file` instead of failing.

---

## Suggested order & rationale

1. Phase 1 (foundational, cheap, unblocks correct masked/hidden behavior).
2. Phase 2 (independent, stops churn/failures from collisions).
3. Phase 3 (safety; prerequisite for Phase 4).
4. Phase 4 (depends on Phase 3).
5. Phase 5 (depends on Phase 3/4 for the recreate fallback).
6. Phase 6 (product decisions; not blocking).

All phases keep the existing summary/report structure and add statuses/notes rather than replacing them.
