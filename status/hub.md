# Hub – status

## Nuläge
Hub-projektet skapat. Egen Lovable Cloud-instans provisionerad. Auth (email/password + Google) på plats. `user_roles` + `has_role` + `docs_sync_log` migrerade. Edge-funktionen `docs-sync` deployad. Admin-UI på `/super-admin/docs-sync`.

`hub.*`-schema live: `organizations`, `org_members`, `entitlements` med enums `org_type`, `member_role`, `app_kind`. RLS: medlemmar läser egen org/membership/entitlements; super_admin har full access; service-role bypassar (används av edge-funktioner). Schemat exponerat via PostgREST.

Edge-funktionen `org-info` deployad och verifierad — returnerar `{ user, org, role, entitlements, cached_until }` exakt enligt `contracts/v1/org-info.md`. Felmodell: `missing_auth` (401), `invalid_token` (401), `no_org` (404).

Kontraktet `contracts/v1/org-info.md` fryst till **v1.0.0** (DRAFT-flagga borttagen).

Publik JWKS-endpoint på `/api/public/jwks.json` (proxar Supabase Auth JWKS via stabil Hub-URL).

## Klart
- Kontrakt `org-info` v1.0.0 fruset och live.
- Edge-funktion `org-info` deployad och verifierad.

## Pågående arbete
- Hub-JWT signering (RS256, kid `hub-2026-06-07`) + edge-funktionen `sso-claim` deployad mot Tipspromenads JWKS-whitelist. Publik nyckel exponeras via `/api/public/jwks.json` tillsammans med Supabase Auth-nycklarna. Audit-tabell `hub.sso_audit` på plats. Avvaktar end-to-end-test mot Tipspromenad i staging innan `contracts/v1/sso.md` fryses till v1.0.0.
- `public.profiles` + sync-trigger från `auth.users` (email citext) — används för email-baserad hub-user-lookup i `sso-claim`.

## Backlog
- Frys `contracts/v1/sso.md` till v1.0.0 efter lyckad staging-claim.
- Flöde A: `sso-issue` + `sso-exchange` + Hub-UI "Öppna Tipspromenad/EventIT".
- Migrera `org-info` till att verifiera Hub-JWT (idag accepterar den Supabase Auth-JWT direkt).
- Edge-funktioner: `provision-org`, `apply-purchase` (Stripe-webhook).
- UI: "Mitt KickinIT", organisationsväljare, fakturor, medlemshantering, "Mina produkter", token-revokering (audit-tabell finns).
- Admin-UI för att skapa orgs/medlemmar/entitlements manuellt.
- Launchpad pekar om provisioning till Hub.

## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS (`/api/public/jwks.json`), hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe) — TBD.
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
