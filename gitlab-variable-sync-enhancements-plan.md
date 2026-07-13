# GitLab Variable Sync — Enhancement Plan

> **Scope:** the `sync_github_variables_to_gitlab` step in `.github/workflows/provision-gitlab-mirror.yml`
> **Status:** proposal / working document (not committed)

---

## 1. Background

| Aspect | Current baseline |
|--------|------------------|
| Naming grammar | `GL_<flags>__<name>` |
| Flags | `M` masked · `P` protected · `E` expansion · `H` masked + hidden |
| Expansion default | **off** (`raw: true`) unless `E` is set |
| Reserved keys | published by the `sync_reserved_gitlab_variables` step, never pruned |
| Stale handling | project-level variables not in the desired set are pruned |

This plan resolves the ⚠️ / ❌ transitions surfaced in the use-case matrix (§2). Phases are ordered so that cheap, foundational, and safety work lands before the heavier hidden-variable work.

---

## 2. Use case matrix

How every value and flag transition behaves against the **current** implementation (before this plan is applied).

**Legend:** ✅ works cleanly · ⚠️ works but churns / degenerate · ❌ fails or needs enhancement.
Flags live in the GitHub variable name, so "editing flags" = renaming the GitHub variable; the resolved GitLab key decides update vs. new key.

### 2.1 Creation — key not yet in GitLab

| # | Scenario | Example (GitHub → GitLab key) | Current behavior | Status |
|----|----------|-------------------------------|------------------|:------:|
| C1 | Visible, no flags | `GL__DB_HOST` → `DB_HOST` | POST `masked:false, raw:true` | ✅ |
| C2 | Masked, valid value | `GL_M__TOKEN`=`Abcd1234` → `TOKEN` | POST `masked:true, raw:true` | ✅ |
| C3 | Masked, invalid value (<8 / space / multiline) | `GL_M__TOKEN`=`ab c` | POST → masking error | ❌ |
| C4 | Protected | `GL_P__X` | POST `protected:true` | ✅ |
| C5 | Expanded | `GL_E__X`=`$OTHER` | POST `raw:false` | ✅ |
| C6 | Hidden, valid value | `GL_H__TOKEN`=`Abcd1234` | POST `masked_and_hidden:true, raw:true` | ✅ |
| C7 | Masked **+** expanded | `GL_ME__X` | POST `masked:true, raw:false` → charset-restricted | ⚠️/❌ |
| C8 | Hidden **+** expanded | `GL_HE__X` | POST `masked_and_hidden:true, raw:false` → contradiction | ⚠️/❌ |

### 2.2 Value transitions — flags unchanged, key exists

| # | Scenario | Example | Current behavior | Status |
|----|----------|---------|------------------|:------:|
| V1 | Value unchanged (visible/masked) | `DB_HOST` same value | compares stored value → unchanged, no write | ✅ |
| V2 | Value changed (visible) | `DB_HOST`: `a`→`b` | `valueChanged` → PUT | ✅ |
| V3 | Value changed (masked, valid) | `TOKEN`: `Abcd1234`→`Wxyz9876` | PUT new value | ✅ |
| V4 | Value changed (masked, invalid new value) | `TOKEN`→`ab c` | PUT → masking error | ❌ |
| V5 | Value unchanged (**hidden**) | `TOKEN` (hidden) same value | `valueChanged=true` forced → PUT every run | ⚠️ |
| V6 | Value changed (**hidden**, rotation) | `TOKEN` (hidden) rotated | can't read old → PUT new value every run | ⚠️ |

### 2.3 Flag transitions — value unchanged, key exists

| # | Transition | Example (rename) | Current behavior | Status |
|----|-----------|------------------|------------------|:------:|
| F1 | add masked | `GL__X`→`GL_M__X` | PUT `masked:true`; ok if value valid | ✅/❌ |
| F2 | remove masked | `GL_M__X`→`GL__X` | PUT `masked:false` | ✅ |
| F3 | add protected | `GL__X`→`GL_P__X` | PUT `protected:true` | ✅ |
| F4 | remove protected | `GL_P__X`→`GL__X` | PUT `protected:false` | ✅ |
| F5 | enable expansion (plain) | `GL__X`→`GL_E__X` | PUT `raw:false` | ✅ |
| F6 | disable expansion (plain) | `GL_E__X`→`GL__X` | PUT `raw:true` | ✅ |
| F7 | add expansion to masked | `GL_M__X`→`GL_ME__X` | PUT `masked:true, raw:false` → charset-restricted | ⚠️/❌ |
| F8 | **add hidden** | `GL_M__X`→`GL_H__X` | `isHiddenNow≠hidden` → **throws**, fails every run | ❌ |
| F9 | **remove hidden** | `GL_H__X`→`GL_M__X` | throws (can't un-hide), fails every run | ❌ |
| F10 | change protected/expansion **on** a hidden var | `GL_H__X`→`GL_HP__X` | hidden unchanged → PUT `protected:true` (+ value refresh) | ⚠️ |
| F11 | multiple flags at once | `GL__X`→`GL_MP__X` | detects each change → single PUT | ✅ |

### 2.4 Name changes, collisions, removal

| # | Scenario | Example | Current behavior | Status |
|----|----------|---------|------------------|:------:|
| N1 | rename the **name** part | `GL__FOO`→`GL__BAR` | `FOO` pruned; `BAR` created | ✅ |
| N2 | rename name **and** flags | `GL__FOO`→`GL_M__BAR` | prune `FOO` + create masked `BAR` | ✅ |
| N3 | GitHub var deleted | remove `GL__FOO` | `FOO` stale → pruned (unless reserved) | ✅ |
| N4 | same key from **variable + secret** | `GL__FOO` (var) + `GL_M__FOO` (secret) | both processed, secret wins, flip-flop each run; hidden mismatch → persistent fail | ❌ |
| N5 | reserved key | `GH_PROJECT_REPO` | never pruned (reserved step output) | ✅ |
| N6 | group-level var, same key | inherited `FOO` | untouched (project endpoint only) | ✅ |

### 2.5 Items needing work

| Use cases | Addressed by |
|-----------|--------------|
| C7, C8, F7 | Phase 1 |
| N4 | Phase 2 |
| C3, V4, F1 | Phase 3 |
| F8, F9 | Phase 4 |
| V5, V6 | Phase 5 |
| multiline/space secrets, secrets→hidden default | Phase 6 (parked) |

Everything else already behaves correctly.

---

## 3. Phase overview

| Phase | Enhancement | Use cases | Effort | Risk | Depends on |
|:-----:|-------------|-----------|:------:|:----:|:----------:|
| **1** | Ignore `E` on masked/hidden variables | C7, C8, F7 | S | Low | — |
| **2** | Detect variable ⇄ secret key collisions | N4 | S | Low | — |
| **3** | Pre-validate masking rules + trim rules text | C3, V4, F1 | S | Low | — |
| **4** | Hidden state change → delete + recreate | F8, F9 | M | Med | Phase 3 |
| **5** | Hidden value rotation → hash sidecar | V5, V6 | M | Med | Phase 3, 4 |
| **6** | Parked / future decisions | — | — | — | — |

Effort: **S** ≈ small · **M** ≈ medium.

---

## 4. Phases

### Phase 1 — Ignore `E` on masked/hidden variables

**Use cases:** C7, C8, F7

**Problem**
`GL_ME__X` / `GL_HE__X` request masked/hidden *and* expansion. GitLab cannot expand masked/hidden variables, and with expansion on the value is charset-restricted (`$` disallowed) — so these fail or are functionally useless.

**Approach**
Derive effective expansion in the sync loop:

```js
const effectiveExpansion = expansion && !desiredMasked;   // desiredMasked = masked || hidden
const raw = !effectiveExpansion;
```

When `expansion` was requested but dropped, flag it on the result and surface a note:
> `E` (expansion) ignored — not compatible with masked/hidden.

| | |
|---|---|
| **Touch points** | loop `raw` derivation · per-row `result` · summary notes |
| **Risk** | Low |
| **Verify** | parse-sim: `GL_ME__`, `GL_HE__` (expansion dropped); `GL_E__` (plain still expands) |

---

### Phase 2 — Detect variable ⇄ secret key collisions

**Use cases:** N4

**Problem**
The same resolved GitLab key coming from *both* a GitHub Variable and a Secret produces two candidates → flip-flop churn every run (secret wins), or a persistent failure on hidden mismatch.

**Approach**
After building `toSync`, group by `glKey`. If a key resolves from more than one candidate:

- **Fail fast (recommended)** — mark the key failed with a clear reason, skip syncing it, surface in the summary:
  > Same GitLab key `FOO` produced by a GitHub variable and a secret — rename one.
- *Alternative* — defined precedence (e.g. secret over variable) and note the skipped duplicate.

| | |
|---|---|
| **Touch points** | post-collection dedupe pass · `failed` / `results` · summary |
| **Risk** | Low |
| **Verify** | sim: duplicate `glKey` from both sources |

---

### Phase 3 — Pre-validate masking rules + trim rules text

**Use cases:** C3, V4, F1 · *prerequisite for Phase 4*

**Problem**
Masked/hidden values that are multiline, contain spaces, or are shorter than 8 characters fail at the API — we currently only surface the raw 400. Also `MASKING_RULES` still lists the Base64 charset, which no longer applies now that expansion is off for masked/hidden.

**Approach**

- Add `maskingViolation(value)` → returns a reason string or `null`. Checks: single line (no `\n`), no spaces, length ≥ 8.
- For masked/hidden candidates, pre-check **before** create/update; on violation, fail that variable with the friendly reason (no API call).
- Trim `MASKING_RULES` to: single line · no spaces · ≥ 8 chars · not matching an existing variable name. Drop the charset clause (or scope it to "only if expansion is enabled", which we no longer allow for masked/hidden).

| | |
|---|---|
| **Touch points** | new helper · sync-loop guard · `MASKING_RULES` constant · convention block wording |
| **Risk** | Low — must mirror GitLab's rules exactly to avoid false rejects |
| **Verify** | sim values: `ab c`, `short`, multiline, and a valid value |

---

### Phase 4 — Hidden state change → delete + recreate

**Use cases:** F8, F9

**Problem**
Hidden is create-only and can't be removed. Adding or removing `H` on an existing key currently throws every run and never applies.

**Approach**
When `isHiddenNow !== hidden`:

1. **Pre-validate the target state first** (Phase 3). If it's hidden/masked and the value is invalid → do **not** delete; fail with a clear reason.
2. Otherwise `DELETE` the existing key, then `POST` it fresh in the target state (hidden uses `masked_and_hidden: true`). The value comes from GitHub, so nothing is lost.
3. Report a new status `recreated`; note the brief non-atomic window.

| | |
|---|---|
| **Touch points** | replace the throw branch · add `recreated` to `STATUS_META` · summary counts/labels |
| **Risk** | **Medium** — non-atomic delete→create; pre-validation prevents the "deleted but recreate failed" trap |
| **Verify** | state matrix: visible→hidden, hidden→visible, invalid value aborts *before* delete |

---

### Phase 5 — Hidden value rotation via hash sidecar

**Use cases:** V5, V6

**Problem**
Hidden values can't be read back, so change can't be detected → we re-push every run (churn), and it relies on the API allowing hidden value updates.

**Approach**

- For each hidden variable `FOO`, maintain a companion **visible** variable `FOO__SRC_SHA` = `sha256(value)` (a one-way hash reveals nothing sensitive).
- Each run, compute `sha256` of the current GitHub value and compare to the sidecar:

  | Comparison | Action |
  |------------|--------|
  | hashes equal | **unchanged** — skip (no churn) |
  | hashes differ | push new value (update, or delete + recreate on locked versions), then update the sidecar |

- Pruning must **whitelist** the `__SRC_SHA` suffix so sidecars aren't treated as stale.

| | |
|---|---|
| **Touch points** | hash helper (Node `crypto`) · hidden-branch change detection · prune whitelist · summary |
| **Risk** | **Medium** — one extra variable per hidden secret; sidecar and prune must stay in sync |
| **Verify** | sim: equal vs changed hash paths; confirm sidecar excluded from prune |

---

### Phase 6 — Parked / future

- **Secrets default to masked + hidden (`H`).** Opt-in, source-based default — more attractive now that hidden values accept most characters. Reverses the "flags come only from the name" rule.
- **File-type variables for multiline / space-containing secrets.** PEM keys etc. can never be masked; map to `variable_type: file` instead of failing.

---

## 5. Ordering rationale

1. **Phase 1** — foundational and cheap; unblocks correct masked/hidden behavior.
2. **Phase 2** — independent; stops churn/failures from collisions.
3. **Phase 3** — safety; prerequisite for Phase 4.
4. **Phase 4** — depends on Phase 3.
5. **Phase 5** — depends on Phases 3–4 for the recreate fallback.
6. **Phase 6** — product decisions; not blocking.

> All phases preserve the existing summary/report structure and **add** statuses and notes rather than replacing them.
