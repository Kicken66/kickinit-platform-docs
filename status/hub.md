# Hub – status

## Publicering
Hub är publicerad 2026-06-09 på **https://kickinit-core-hub.lovable.app**. Detta är den stabila URL som syskonprojekt (Tipspromenad, EventIT) ska använda som `HUB_BASE_URL`. Eget domännamn (t.ex. `hub.kickinit.se`) kan kopplas senare via Projektinställningar → Domains; tills dess pekar `iss`-claim i Hub-JWT fortfarande på den logiska identifieraren `https://hub.kickinit.se` (konfigurerbar via `HUB_ISS`).

## Nuläge
Hub-projektet skapat. Egen Lovable Cloud-instans provisionerad. Auth (email/password + Google) på plats. `user_roles` + `has_role` + `docs_sync_log` migrerade. Edge-funktionen `docs-sync` deployad. Admin-UI på `/super-admin/docs-sync`.

`hub.*`-schema live: `organizations`, `org_members`, `entitlements` med enums `org_type`, `member_role`, `app_kind`. RLS: medlemmar läser egen org/membership/entitlements; super_admin har full access; service-role bypassar (används av edge-funktioner). Schemat exponerat via PostgREST.

Edge-funktionen `org-info` deployad — verifierar nu Hub-utfärdad RS256-JWT (`iss=https://hub.kickinit.se`, `aud=tipspromenad|eventit`) och returnerar `{ user, org, role, entitlements, cached_until }` enligt `contracts/v1/org-info.md`. Felmodell: `missing_auth` (401), `invalid_token` (401), `no_org` (404).

Kontraktet `contracts/v1/org-info.md` bumpat till **v1.1.0** — addons-nycklarna `results_email`, `member_lookup`, `paper_print` exponeras nu även som top-level boolean-fält i `entitlements.tipspromenad` (additivt; `addons[]` behålls). Stänger Tipspromenads gap (migration `0005-hub-org-info-entitlements-gap.md`).

Publik JWKS-endpoint på `/api/public/jwks.json` (proxar Supabase Auth JWKS via stabil Hub-URL).

## EventIT-integration — Hub-sidan klar 2026-06-08, e2e-verifierad 2026-06-09
`migration/0003-eventit-integration.md` pushat till platform-docs.

**`EVENTIT_*`-secrets satta på Hub 2026-06-09** (alla fyra: `EVENTIT_AUTH_URL`, `EVENTIT_ANON_KEY`, `EVENTIT_CALLBACK_URL`, `EVENTIT_EXCHANGE_SECRET`).

**End-to-end-verifiering 2026-06-09** via `sso-e2e-test` med `{app:"eventit"}` → samtliga steg gröna: `sso_issue`, `sign_exchange_token`, `sso_exchange` (Hub-JWT, `aud=eventit`, `org_id=hbk`, `role=super_admin`), alla JWT-checks `true` (alg_RS256, has_kid `hub-2026-06-07`, iss_present, aud_matches_app, exp_future, sub_matches_caller, email_matches, has_jti), `org-info?app=eventit` → `entitlements.eventit = { plan:"trial", events_remaining:1, addons:[], results_email:false, member_lookup:false, paper_print:false }` (plan-implicit fallback; EventIT addon-katalog ännu inte spec:ad).

Hub-sidan av EventIT-integrationen är **produktionsklar**. Återstår på EventIT-projektets sida: implementera `/sso/callback` som POSTar till `sso-exchange`, etablera lokal session, validera Hub-JWT via JWKS (`https://kickinit-core-hub.lovable.app/api/public/jwks.json`), och anropa `org-info`. Plus: spec:a EventIT addon-katalog → `entitlement_keys.md` + `org-info` v1.2.0.

## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS (`/api/public/jwks.json`), hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe) — TBD.
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
