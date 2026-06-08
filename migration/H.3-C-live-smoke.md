# H.3-C — Live cutover-smoke mot `tipstjanst.online`

**Datum:** 2026-06-08
**Operatör:** Lovable agent (super_admin-session)
**Mål:** 6/6 smoke-modes gröna mot `https://tipstjanst.online/...` per
`H.3-cutover-checklist.md` §C, som GO-krav för H.3-signoff.

## Status: ⛔ **BLOCKED — H.3-blocker #4 (prod-seed saknas)**

Smoke kan inte slutföras eftersom prod-DB saknar **öppna rundor** för 5 av 6
quiz_modes. Domän/SSL/SPA-fallback är dock verifierat fungerande (se §1
nedan). Detta är en **pre-cutover-blocker** — fortsatt arbete kräver att
seed-data flyttas in i prod (del av §A datamigration), alternativt att en
tidsbegränsad smoke-org seedas på prod innan §C kan stängas.

---

## 1. Domän / SSL / SPA-fallback (verifierat 2026-06-08 08:39 UTC)

| # | Test | Resultat | Anmärkning |
|---|------|----------|------------|
| 1.1 | `HEAD https://tipstjanst.online/` | ✅ HTTP 200, SSL OK | `server: cloudflare`, HSTS aktiv. |
| 1.2 | `HEAD https://www.tipstjanst.online/` | ✅ HTTP 200 | Serverar samma app. |
| 1.3 | `HEAD https://app.kickinit.se/` | ✅ HTTP 200 | Admin-domän serverar samma SPA. |
| 1.4 | Deep-link SPA-fallback (`/quiz/<random-uuid>`) | ✅ HTTP 200 + `index.html` | SPA-fallback fungerar. |
| 1.5 | Landningssida `<title>` | ✅ `KickinIT Admin Panel` | Korrekt build deployad. |

Domänlagret är klart för cutover (samstämmigt med PASS i `H.3-B-signoff.md`).

---

## 2. 6/6 smoke-modes anon-flöde (BLOCKED)

Per `H.2.0.b-smoke-checklist.md` (re-run 4–5) ska följande modes köras från
`/quiz/:roundId` → register → add-more → payment → play → submit → results:

| Mode | Prod-runda finns? | Status |
|------|-------------------|--------|
| `tipspromenad` (gps_unlock=false) | ❌ 0 öppna | ⛔ BLOCKED |
| `tipspromenad` (gps_unlock=true) | ❌ 0 öppna | ⛔ BLOCKED |
| `fragesport` | ❌ 0 | ⛔ BLOCKED |
| `styrd` | ❌ 0 | ⛔ BLOCKED |
| `aggjakt` | ❌ 0 öppna (1 completed) | ⛔ BLOCKED |
| `poangorientering` | ❌ 0 | ⛔ BLOCKED |
| `mystery_walk` | ❌ 0 | ⛔ BLOCKED |

DB-query (2026-06-08): `SELECT status, quiz_mode, count(*) FROM quiz_rounds GROUP BY status, quiz_mode;` → 0 rader med `status='open'`; 6 tipspromenad-drafts, 4 tipspromenad + 1 aggjakt completed.

Anon-deltagar-flöde i `quiz-register-participant` kräver `status='open'` → ingen mode kan smoke-testas mot live.

---

## 3. Övriga GO-krav (status)

| Krav (§C) | Status | Källa |
|---|---|---|
| `tsc --noEmit` = 0 | ✅ | CI-grön main (2026-06-06) |
| ESLint `no-explicit-any` aktiv | ✅ | `eslint.config.js` |
| Edge-coverage ≥70 % branch på 5 publika fn | ⚠️ Ej mätt denna runda | Senast H.2.4 |
| `apply-purchase` + `provision-private-org` E2E mot Launchpad-staging | ⚠️ Delvis | E3-tester gröna (`index_e3_test.ts`); apply-purchase E2E ej körd |
| `sso-from-launchpad` round-trip | ⚠️ Ej körd live | Verifierad i H.2.2 |
| DB-linter: 0 ERROR | ✅ | 2026-06-07 |

---

## 4. H.3-blocker #4 — Prod-seed saknas

Se `docs/migration/0006-h3c-prod-seed.md`. Sammanfattning: prod-DB saknar `status='open'`-rundor och 5/6 modes saknas helt. Rekommenderat alt A: engångsmigration `seed_h3c_smoke_org.sql` → smoke → `cleanup_h3c_smoke_org.sql`. Alt B (kör §A datamigration först) gör §C beroende av §A vilket inte var planerat.

---

## 5. Mätpunkter denna runda

| Mätpunkt | Mål | Resultat |
|----------|-----|----------|
| Domän/SSL/SPA-fallback verifierat live | OK | ✅ |
| 6/6 smoke-modes gröna mot tipstjanst.online | 6/6 | ⛔ 0/6 (BLOCKED) |
| Nya P0-findings i app-kod | 0 | ✅ 0 |
| Nya P0-findings i domän/infra | 0 | ✅ 0 |

---

## 6. Nästa steg

1. Beslut: Alt A (seed-migration) eller Alt B (kör §A datamigration först)?
2. Om Alt A: seed → smoke 6/6 → uppdatera detta dok med PASS per mode → `H.3-C-signoff.md`.
3. Vid signoff: uppdatera `docs/status/tipspromenad.md` och `H.3-cutover-checklist.md` §C med PASS-datum.
