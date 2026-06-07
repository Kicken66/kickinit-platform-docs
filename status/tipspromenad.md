# Tipspromenad — status

Per-app statusdokument. Här bor det som rör **detta** Lovable-projekt och
inte de andra (Hub, EventIT). Delade kontrakt: se `docs/contracts/README.md`.

## Aktuellt fokus

- H.2.13 / E3 (se `docs/migration/H.2.13.E3-plan.md`).
- StationPicker-fix enligt `.lovable/plan.md` (väntar verifiering).
- Docs-arkitektur etablerad (alt C hybrid) — kontrakt flyttade till
  `kickinit-platform-docs`, push via edge-funktionen `docs-sync`.
- **Arkitektur uppdaterad: instance-per-app** (ADR-0002 i docs-repot).
  Hub har egen Lovable Cloud-instans; Tipspromenad fortsätter med sin
  egen DB och kommer i en framtida H.x-tranche att läsa org +
  entitlements från Hubs `org-info`-endpoint istället för lokala
  `organizations`/`user_roles`.
- **`org-info` live och verifierad hos Hub** (testat i Hub-sessionen
  mot seedad data, Hedareds BK). Svarar
  `{ user, org, role, entitlements, cached_until }`. Kontraktet är
  fryst på v1.0.0 (`contracts/v1/org-info.md` i docs-repot).
- **Staging-integration kan börja nu.** Peka direkt på Hubs
  Supabase-URL:er tills Hub publiceras på stabil domän:
  - `org-info`:
    `https://yeyubjxotynuelcahwpy.supabase.co/functions/v1/org-info`
  - JWKS (Supabase Auth direkt):
    `https://yeyubjxotynuelcahwpy.supabase.co/auth/v1/.well-known/jwks.json`
  - Hub-proxyn `/api/public/jwks.json` ligger bakom Lovable-auth tills
    Hub publiceras — använd inte den i staging.
  När Hub publicerats byts båda URL:erna till stabil Hub-domän utan
  kontraktsändring. Tidigare "vänta på Hub"-notering är upphävd för
  staging-bruk.
- **Klient implementerad** (2026-06-06) bakom feature flag
  `VITE_USE_HUB_ORG_INFO` (default **av**):
  - `src/integrations/hub/config.ts` — staging-URL:er + flagg.
  - `src/integrations/hub/orgInfo.ts` — `fetchOrgInfo()` + typer som
    speglar kontrakt v1.0.0.
  - `src/hooks/useHubOrgInfo.ts` — React Query-wrapper med
    `staleTime: 60 s`, `gcTime: 5 min`, ingen retry på `invalid_token`/
    `no_org`/`missing_auth`.
  - `src/components/dev/HubOrgInfoDebug.tsx` — dev-panel monterad
    under `<details>` på `/super-admin/docs-sync` för att jämföra
    Hub-svar mot lokal `currentOrg`.
  - `OrganizationContext` är **orört** — läser fortsatt lokalt.
  - **Förväntat utfall i staging:** `invalid_token` (401) tills SSO-
    flöde via Hub utfärdar JWT som Hubs `org-info` accepterar.
    Tipspromenads Supabase-JWT signeras av annan instans. Klient-,
    cache- och error-paths verifierade.
  - **Aktivera lokalt:** `echo 'VITE_USE_HUB_ORG_INFO=true' >> .env.local`
    och starta om dev-servern.
- **SSO-flöde skissat** (2026-06-06): `contracts/v1/sso.md` v0.1 DRAFT
  (i `/mnt/documents/sso-draft-v0.1.md` — pusha via `/super-admin/docs-sync`).
  Flöde B (claim från redan inloggad app-användare) prioriterat för MVP;
  flöde A (Hub-initierad redirect) skissat men ej krävt.
  - Tipspromenad-leverabler implementerade:
    - `src/integrations/hub/session.ts` — claim/storage av Hub-JWT
      i sessionStorage (`kickinit.hubToken.v1`), refresh-margin 5 min.
    - `src/contexts/HubSessionContext.tsx` — `HubSessionProvider`
      auto-claim:ar vid inloggning/user-byte/utgång; monterad runt
      `OrganizationProvider` i `App.tsx`.
    - `src/hooks/useHubOrgInfo.ts` — använder nu Hub-JWT (inte
      Supabase-JWT) som `Authorization`.
    - Debug-panelen på `/super-admin/docs-sync` visar token-TTL,
      claim-fel och har "Reclaim Hub-token"-knapp.
  - **Förväntat utfall i staging:** `claim`-anrop fallerar tills Hub
    deployar `/sso-claim` med Tipspromenads JWKS i whitelist. Felet
    visas tydligt i debug-panelen.
  - Flöde A (`sso-from-hub` edge function + `/sso/callback`-route) =
    senare epic när Hub UI ska kunna djuplänka in användare.
- **Runtime-flagga införd** (2026-06-07): `HUB_ORG_INFO_ENABLED` läses
  nu i ordning `localStorage["kickinit.useHubOrgInfo"]` >
  `VITE_USE_HUB_ORG_INFO` > `false`. Toggle i debug-panelen på
  `/super-admin/docs-sync` (per-browser) eftersom Lovable-preview saknar
  `.env.local`. Motsvarande `kickinit.useHubEntitlementsAuthoritative`
  styr om Hub-entitlements ska vara primärkälla i `useOrgEntitlements`
  (default = shadow).
- **Hub-entitlements i shadow-läge** (2026-06-07): `useOrgEntitlements`
  läser parallellt från Hubs `org-info` när flaggan är på och loggar
  `console.warn("[hub-entitlements] mismatch", …)` per (org, diff-sig).
  Mappning lever i `src/integrations/hub/entitlementMapping.ts` —
  **TODO**: publicera `entitlement_keys.md` i docs-repot så båda apparna
  delar samma nyckeluppsättning.
- **SSO-claim-verifiering 2026-06-07 — BLOCKERAD på Hub-sidan.**
  Toggle på i preview, claim körs, Hub svarar `401 invalid_token`.
  Rotorsak: `session.access_token` som supabase-js skickar är ibland
  **HS256** (legacy symmetrisk, utan `kid`) efter token-refresh. Hub
  kan inte verifiera HS256 via JWKS — JWKS exponerar bara asymmetriska
  nycklar. Tidigare REST-anrop i samma session använde ES256 (kid
  `e683a410-…`); auth-loggen visar att en `grant_type=refresh_token`-
  call precis innan claim:en bytte aktiv signeringsnyckel till HS256.
  - **Tipspromenad kan inte åtgärda detta:** Lovable Cloud exponerar
    inte "JWT Signing Keys" i Auth-UI:t (se Cloud → Users → Auth
    Settings). Vi kan alltså inte tvinga bort HS256-läget från klient-
    eller projektsidan.
  - **Lösning flyttar till Hub:** byt verifieringsstrategi i
    `/sso-claim` från krypto-verifiering (JWKS) till HTTP-introspection
    mot Tipspromenads Auth-server:
    ```
    GET https://wkmtgihiveqwyvozrlid.supabase.co/auth/v1/user
    Authorization: Bearer <inkommande-jwt>
    apikey: <TIPSPROMENAD_ANON_KEY>
    ```
    200 → läs `email`/`id`, mappa mot Hubs `users.email`, utfärda
    Hub-JWT. 401 → returnera `invalid_token`. Tipspromenads Auth-server
    validerar JWT:n oavsett alg (HS256 eller ES256).
  - Nytt Hub-secret: `TIPSPROMENAD_ANON_KEY` =
    `sb_publishable__LhLQvhbY5f8nKiylbNrmQ_39CFNjbV` (publik).
    Generalisera till `verifyAppUser(appSlug, jwt)` med config-tabell
    `{auth_url, anon_key}` per app inför EventIT.
  - Klient-, cache-, error- och toggle-paths verifierade Tipspromenad-
    sidan (panel visar `invalid_token` rent, ingen retry-storm).
    Inget mer att göra här innan Hub deployat ny verifiering.
- **Migration 0001 — Hub org seed pushed**
  - Commit: [`b51817c`](https://github.com/Kicken66/kickinit-platform-docs/commit/b51817c2ca57be6a512c7e6370ff584140911719)
  - Fil: `migration/0001-hub-org-seed.md` (Ericsson, Fixhult AIK, Super Event)
  - Hedareds BK (slug-kollision) flyttad till separat migration **0001b**.
- **Migration 0001b — Hedareds slug-flip pushed** (2026-06-07)
  - Commit: [`c1798ee`](https://github.com/Kicken66/kickinit-platform-docs/commit/c1798ee41511c81b335ab8a14ae6818e056f57c2)
  - Fil: `migration/0001b-hedareds-slug-flip.md`
  - Variant **(b)** vald: Tipspromenad döper om `hedareds-bollklubb-000000`
    → `hbk` (matchar Hubs befintliga slug). Ingen FK rörs i någondera DB.
- **Migration 0001b — applicerad på Tipspromenad** (2026-06-07)
  - Hub har bekräftat att 0001 (Ericsson, Fixhult, Super Event) är seedad.
  - 0001b kördes lokalt: `quiz_organizations.id …0bb1` → `slug='hbk'`
    (sanity-checks: rad fanns, ingen `hbk`-kollision).
  - Klart för cross-app-test: `org-info` mot Hubs `hbk` bör nu matcha
    Tipspromenads `currentOrg.slug`.












## Senaste sign-offs

| Workstream | Sign-off |
|---|---|
| H.2.13 | `docs/migration/H.2.13-signoff.md` |
| H.2.12 | `docs/migration/H.2.12-signoff.md` |
| H.2.11 | `docs/migration/H.2.11-signoff.md` |
| H.2.10 | `docs/migration/H.2.10-signoff.md` |
| H.3 cutover | `docs/migration/H.3-cutover-checklist.md` |

## Externa beroenden

- **Launchpad** (kickinit.se) — anropar `provision-private-org`,
  `apply-purchase`, `sso-from-launchpad`, pollar `onboarding-status`.
- **CRM Suite** — äger Stripe. Slug är delad nyckel; se
  `contracts/v1/stripe-checkout-integration.md` i docs-repot.

## Backlog

Se `docs/backlog/feature-parity.md`.
