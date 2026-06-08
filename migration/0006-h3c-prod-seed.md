# H.3-blocker #4 — Prod-seed saknas för H.3-C live-smoke

**Öppnad:** 2026-06-08
**Spår:** H.3-C (live-smoke mot `tipstjanst.online`)
**Severity:** P1 (blockerar GO-kravet i `H.3-cutover-checklist.md` §C, men
inte cutover-fönstret som helhet — kan lösas före §A datamigration).

## Symptom

`SELECT status, quiz_mode, count(*) FROM quiz_rounds GROUP BY status, quiz_mode;`
returnerar 0 rader med `status='open'` i prod-DB. 5 av 6 quiz_modes har
ingen rad alls (inte ens draft). Endast `tipspromenad` har drafts (6 st i
test-orgs Ericsson / Bollklubben) samt en handfull `completed`.

Konsekvens: anon-deltagar-flödet i `quiz-register-participant` kräver
`status='open'` → ingen mode kan smoke-testas live, vilket blockerar
PASS-kvitto för §C.

## Bakgrund

Smoke 6/6 har t.o.m. H.2.x körts mot **preview-DB** med två test-orgs
(`Sanity Test G1.5` + `Sanity Test G2`). Dessa orgs finns inte i prod
eftersom §A datamigration (memberhub → Quest) inte är körd. H.3-C-doc
antog implicit att seed skulle finnas — den noteringen saknas.

## Beslut som krävs

Välj alternativ:

**Alt A (rekommenderas — låg risk, ~30 min):**
Skapa engångsmigration `seed_h3c_smoke_org.sql` som lägger in:
- 1 smoke-org (`Smoke H.3-C 2026-06-08`, ej kopplad till någon kund).
- 1 `quiz_rounds` per mode (tipspromenad×2 m/u `gps_unlock_enabled`,
  fragesport, styrd, aggjakt, poangorientering, mystery_walk).
- Minimal qbank + 2 stationer per runda.
- Sätt `status='open'` under smoke, sätt `status='archived'` direkt
  efteråt (eller dropa hela orgen) i en `cleanup_h3c_smoke_org.sql`-migration.

Pro: oberoende av §A, kan köras nu, inga FK-flytt, kortvarig publik
exponering kan mitigeras med obfuskerad org-UUID.

Con: extra migrationspar (seed + cleanup) — minimalt overhead.

**Alt B — kör §A datamigration först:**
Flytta memberhub-snapshot till prod enligt H.3-cutover-checklist §A
(11 tabellgrupper, 8 export/import-steg). Då finns produktionsrundor att
smoke-testa mot.

Pro: ingen ad hoc-seed; smoke verifierar faktiskt migrerad data.

Con: gör §C beroende av §A, vilket inte var avsikten i checklistans
ordning. Större jobb, kräver lågtrafikfönster — även om §A i sig är
beslutat som ej-kommunicerat (C utgår), så är data-tunghet ett risk-steg
som hellre körs efter att §C-smoke har visat att stacken fungerar.

## Rekommendation

Alt A. Kör seed → smoke 6/6 → cleanup → uppdatera
`H.3-C-live-smoke.md` med PASS-rader → `H.3-C-signoff.md`. Lämna §A
intakt i sin egen tranche.

## Länkar

- `docs/migration/H.3-C-live-smoke.md` — BLOCKED-rapporten.
- `docs/migration/H.3-cutover-checklist.md` §C — GO-krav.
- `docs/migration/H.2.0.b-smoke-checklist.md` re-run 4–5 — preview-PASS
  per mode (referens för förväntat resultat).
