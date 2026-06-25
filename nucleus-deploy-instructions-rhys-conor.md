# Nucleus — pull & deploy instructions (Rhys & Conor)

**Branch:** `feature/entra-sso` · **Target:** the Nucleus server environment · **Author:** Stu

This is **not a routine deploy.** It is the first one carrying a database schema change *and* a change to
how login resolves identity. Two things below (§A and §B) can break the environment if handled like a normal
code bump. Read those first, then follow the steps. If either is unclear, stop and check with Stu before
deploying — a half-applied schema or a broken login is much costlier than a paused deploy.

> Confirm with Stu the exact commit range / HEAD you are deploying before you start, and that the SSO rewire
> is included in it.

---

## What this version contains (since the last deploy)

| Change | Type | Migration? | Watch for |
|---|---|---|---|
| DB-1 (Charles) | **Schema** — additive fields + column renames + a DEFAULT | **Yes** (039–044 + mirrors 31–36) | §A — the renames/DEFAULT need care on a data-bearing DB |
| OCR-3 Wave 1 | API-only categorisation/extraction fixes | No | No user-facing change; safe |
| DB-2 | Two storage fields (`supplier_name`, `vat_rate`) | **No** (schemaless) | Nothing to do — fields appear on insert |
| Entra SSO rewire | **Auth-core** — login now resolves via `user_data.bname` | No | §B — server `user_data` must be populated, or SSO fails |

---

## §A — DB-1 is a schema change with renames and a DEFAULT (the destructive-step risk)

Migrations 039–044 auto-apply when the new API boots (or on `pnpm db:seed`). They are mostly additive, **but
they include column renames** (on `mileage_rate` and `mileage_priv`) **and a field with a DEFAULT**
(`mileage_rate.default_vat_code DEFAULT 'V0'`). The DEFAULT only takes effect on the *insert* path, and the
seeders use `UPSERT … CONTENT`, so applying these **over existing data is not reliable** — the proven-clean
path is a **fresh data-dir re-seed** (the insert path).

That creates a decision that depends on what the target environment holds, and **only Stu can confirm which
it is:**

- **If the target is a demo / sandbox with disposable data** → the clean path is a fresh re-seed (same as the
  local run book): stop SurrealDB, clear the data-dir **contents**, start, `pnpm db:seed`. Data is rebuilt
  from seed. Simple and reliable.
- **If the target holds data that must be preserved** → **do NOT fresh-re-seed**, and do **not** assume the
  renames apply cleanly on boot. This needs a considered migration (the run book does not cover preserving
  data through the renames). **Stop and agree the approach with Stu/Charles before deploying** — this is the
  step most likely to corrupt the environment if rushed.

**SurrealDB gotchas (apply to any re-seed) — these have bitten before:**
- Clear the **contents** of the data dir (`manifest`, `sstables`, `vlog`, `wal`, `LOCK`), **leaving the
  folder itself**. Deleting the folder, or a *partial* wipe, leaves SurrealKV unable to open the store →
  3.0.5 crash-loops.
- **Never** use “Refresh DB.”
- Fail-fast, don’t roll back: if a migration throws, the seed aborts and names the failing file — capture it
  and stop, don’t attempt a partial rollback.

Expected seed tail on a clean re-seed: migrations 039–044 applied; `mileage_rate` 17, `mileage_priv` 330,
`user_data` 67 (counts may differ on the server if it carries more than the seed set — confirm with Stu what
“correct” looks like there).

---

## §B — SSO now resolves via `user_data.bname` (the broken-login risk)

The login lookup no longer reads `employee.username`. It now matches the SAML NameID (the 4×4, e.g.
`MORR5286`) against **`user_data.bname`** and reads `pernr` off that same row. There is **no fallback** to the
old path — so:

> **If the server’s `user_data` table is not populated with `bname` + `pernr` for real users, SSO will not
> resolve and no one can log in via Entra.**

This is the single most important pre-deploy check. **Before relying on SSO on the server, confirm
`user_data` is populated** (this is what the SAP-dev feed provides). If it isn’t yet populated for the people
who need to log in, either the feed must run first, or the deploy of the auth change should wait — check with
Stu. Everything downstream of the match (roles, permissions, the minted token) is keyed on `pernr` and is
unchanged, so once `user_data` is populated the rest behaves exactly as before.

Two related notes (neither is a regression):
- The full SAML round-trip was **only verifiable on a live server** (it needs the real Entra IdP — it can’t
  be exercised locally). So the **post-deploy live SSO test below is the real proof** this change works.
- The diagnostic/debug SSO page now shows `bname` → PERNR instead of the old name/username fields. Expected,
  not a fault.

---

## Steps

### 1. Pre-deploy checks
- Confirm the HEAD/commit range to deploy with Stu (must include the SSO rewire).
- Confirm **§A**: is the target disposable-data (fresh re-seed) or data-bearing (agree the approach first)?
- Confirm **§B**: is the server’s `user_data` populated with `bname`+`pernr` for the users who must log in?
- Confirm the server’s SAML config (`.env` / Entra app settings, ACS URL, certificate) is in place — the
  rewire didn’t change assertion validation, but a deploy still needs the IdP config present on the server.

### 2. Pull
- `git fetch` and bring the target to the agreed HEAD on `feature/entra-sso` (clean fast-forward; if it is
  not a clean fast-forward, stop and check with Stu).
- `pnpm install` only if the lockfile moved.

### 3. Build
- **API:** `tsc` → `dist` (confirm clean, exit 0).
- **Flutter web:** build for the server in **SAML mode**, not the local bypass mode. (Local dev runs an
  auth-bypass; the deployed web build must have real SAML auth enabled — this is the SAML web-build caveat.
  Confirm the build flag/config you use for the server build with Stu if unsure.)
- Use your existing artifact/bundle + jump-box transfer process to get the build to the server — that
  mechanism is yours; this doc doesn’t prescribe it.

### 4. Database (per §A)
- Disposable-data target: stop SurrealDB → clear data-dir **contents** (leave the folder) → start → `cd
  apps/api && pnpm db:seed`. Watch for the 039–044 lines and the expected counts.
- Data-bearing target: **do not re-seed** — follow the approach agreed with Stu/Charles in step 1.

### 5. Bring up the API and apply
- Start the new API. On boot it will report 039–044 as applied (or “already applied, skipping” on a second
  boot). If a migration throws, capture the failing file and stop.

### 6. Post-deploy verification — do not declare done until these pass
- **DB schema (DB-1):** `INFO FOR TABLE` shows the additive fields and renames —
  `expense_claim` has `departure_datetime/arrival_datetime/location/text_id/vat_code`;
  `employee` has `hr_valid_from/hr_valid_to`;
  `mileage_rate` has `default_vat_code`/`period_parameter` and the new natural-key index (old
  `special_code`/`row_index` gone);
  `mileage_priv` has `vehicle_class/period_parameter/valid_from/valid_to` (old `pkwkl/pekez/begda/endda`
  gone);
  `user_data` has `auth_approver`/`auth_claimer` (and retains `auth_expense`).
- **SSO live round-trip (the real test for §B):** log in via Entra with a real account whose `user_data` row
  exists → confirm it resolves to the correct person with their roles/permissions, and the app loads. This is
  the proof the rewire works on the live IdP — it could not be tested locally.
- **Smoke test:** create a mileage claim for a private-vehicle employee (exercises the renamed `mileage_priv`
  columns); create a simple expense claim and confirm OCR categorisation still works (OCR-3 Wave 1).

### 7. Known gaps (not regressions — don’t chase these as bugs)
- A few test users may have **no `user_data` row yet** (awaiting the SAP-dev feed) — they won’t resolve via
  SSO until the feed adds them. They didn’t resolve under the old path either.
- `employee.username` is now **unused by any code** (migrations 035/036 are left in place). A separate later
  cleanup will remove the dead field — not part of this deploy, don’t touch it.

---

## If SSO fails after deploy
The most likely cause is §B — `user_data` not populated for the user testing it. Check that row exists with
`bname` + `pernr` before assuming the code is at fault. If `user_data` genuinely isn’t ready on the server,
the safe response is to hold the auth change with Stu rather than improvise a fallback. Capture the failing
state (the diagnostic page, the API log) and flag it; fail-fast over guesswork.

---

*Questions on any step → Stu. Don’t improvise on §A (data-bearing DB) or §B (empty `user_data`) — those are
the two that can take the environment down.*
