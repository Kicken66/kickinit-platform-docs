# ADR 0001 — Docs-arkitektur för KickinIT-plattformen

**Status:** Accepted
**Datum:** 2026-06-05

## Kontext

KickinIT består av tre Lovable-projekt (Tipspromenad, Hub, EventIT) plus en
extern Launchpad/CRM. De delar versionerade API-kontrakt och designtokens
som inte får drifta isär. Initialt försökte vi använda git submodules under
`docs-platform/` i varje projekt, men Lovable-sandboxen blockerar
stateful git-kommandon (`git submodule add`, redigering av `.gitmodules`),
och varje Lovable-projekts GitHub-koppling är 1:1 — submodule från ett
externt repo går inte att lägga till från Lovables sida.

## Beslut

Hybrid:

- **Delade artefakter** (API-kontrakt, ADR, designtokens, JSON-scheman) lever
  i ett separat publikt repo: `Kicken66/kickinit-platform-docs`.
  - Läses on-demand via `raw.githubusercontent.com/...` av agenter och appar.
  - Uppdateras via edge-funktionen `docs-sync` i varje Lovable-projekt
    (kräver `super_admin` + `GITHUB_DOCS_PAT` som secret). Funktionen
    använder GitHub REST API (`PUT /repos/.../contents/<path>`) och loggar
    varje push i `docs_sync_log`.
- **Per-app-artefakter** (sign-offs, smoke-checklistor, migrationsnoter,
  `docs/status/<app>.md`) lever lokalt i respektive Lovable-projekt och
  versioneras via projektets egna GitHub-repo.

## Konsekvenser

- Ingen automatisk lokal cache av docs-repot i Lovable-projekten →
  ingen drift-risk.
- Skrivflöde går via edge-funktion → revisorbart i `docs_sync_log`.
- Hub och EventIT återanvänder samma `docs-sync`-funktion + samma PAT när
  de skapas.
- Submodule-ansatsen är **förbjuden** framöver — se denna ADR innan
  någon föreslår den igen.
