# Hub – status

## Nuläge
Hub-projektet skapat. Egen Lovable Cloud-instans provisionerad. Auth (email/password + Google) på plats. `user_roles` + `has_role` + `docs_sync_log` migrerade. Edge-funktionen `docs-sync` deployad. Admin-UI på `/super-admin/docs-sync`.

`hub.*`-schema live: `organizations`, `org_members`, `entitlements` med enums `org_type`, `member_role`, `app_kind`. RLS: medlemmar läser egen org/membership/entitlements; super_admin har full access; service-role bypassar (används av edge-funktioner). Schemat exponerat via PostgREST.

Edge-funktionen `org-info` deployad — verifierar nu Hub-utfärdad RS256-JWT (`iss=https://hub.kickinit.se`, `aud=tipspromenad|eventit`) och returnerar `{ user, org, role, entitlements, cached_until }` enligt `contracts/v1/org-info.md`. Felmodell: `missing_auth` (401), `invalid_token` (401), `no_org` (404).

Kontraktet `contracts/v1/org-info.md` fryst till **v1.0.0** (DRAFT-flagga borttagen).

Publik JWKS-endpoint på `/api/public/jwks.json` (proxar Supabase Auth JWKS via stabil Hub-URL).

## Klart
- Kontrakt `org-info` v1.0.0 fruset och live.
- Edge-funktion `org-info` deployad med Hub-JWT-verifiering.
- Kontrakt `sso.md` v1.0.0 fryst.
- Flöde B (`sso-claim`) deployad, RS256 + JWKS verifierad, audit-tabell `hub.sso_audit` skriver per claim.
- `public.profiles` + sync-trigger från `auth.users` (email citext).
- Members-import v1 (migration 0002): `hub.org_members_registry` + auto-claim-triggers (`tg_profiles_claim_registry` på `public.profiles`, `tg_registry_claim_existing` på `hub.org_members_registry`). Edge-funktion `members-registry` (super_admin-only: list_orgs / list / add / import_csv / remove). Admin-UI på `/super-admin/members`. Seed: kicken@kickinit.se som org_admin i hbk — auto-claimad av triggern (verifierat).
- 0001c entitlement-seed (plan-implicit i stället för seedade rader): `org-info` synthesizar `entitlements.tipspromenad` när ingen explicit `hub.entitlements`-rad finns för orgen. `association` → `{ plan: "club", addons: [results_email, member_lookup, paper_print, sponsor_control], implicit: true }`. `private` → `{ plan: "standard", addons: [], implicit: true }` tills `apply-purchase` skapar en rad. Speglar `contracts/v1/entitlement_keys.md` v1.0.0 (88c1dda) och `migration/0001c-hub-entitlement-seed.md` (37ee35a). EventIT TBD.
- **Flöde A (`org-info`) verifierat end-to-end från Tipspromenad** — statusdokument `0003-org-info-flode-a-live.md` pushat till kickinit-platform-docs (commit `7500816`). Tipspromenad bekräftar att Hub-utfärdad RS256-JWT valideras, `org-info` returnerar korrekt `{ user, org, role, entitlements }`, och hela flödet är produktionsklart.
- **`sso-exchange` v1.0.0 deployad** enligt `contracts/v1/sso-exchange.md` (commit `cb5afd8`). Implementationen migrerad från header-HMAC till token-i-body-format (`b64u(payload).b64u(hmac_sha256(secret, b64u(payload)))`). Payload-schema: `{ code, app, iat, exp, nonce }`, TTL ≤ 60s, nonce uuid v4. Returnerar `{ hub_jwt, expires_at, hub_user_id, email, org_id, role }`. Felkoder enligt v1.0-tabellen: `invalid_token` / `app_mismatch` / `token_expired` / `replay_detected` / `code_not_found` / `code_expired` / `code_consumed` / `internal_error`. `X-Correlation-Id` ekas som `request_id` i felresponse. Smoke-test mot deployad funktion: `{token:"abc.def"}` → 400 `invalid_token` / `payload_decode_failed` ✓.
- **`hub.sso_audit` utökad för v1.0**: nya kolumner `flow` (`"claim" | "exchange"`), `nonce`, `code_hash`, `error_code`, `request_id`, `created_at`, `id` (surrogat-PK). `jti`/`hub_user_id`/`role`/`expires_at` nu nullable så att replay-reservationer och fel-rader kan loggas. Unikt index `sso_audit_exchange_nonce_uidx` på `(app, nonce) WHERE flow='exchange'` ger replay-skydd via INSERT-konflikt (`23505` → `replay_detected`).

## Pågående arbete
- Flöde B (`sso-exchange`) end-to-end: `TIPSPROMENAD_EXCHANGE_SECRET` konfigurerat och synkat. SSO E2E-test (`sso-e2e-test`) deployat och testat lokalt. Health-check ✓ (smoke-test mot deployad `/sso-exchange` med `{token:"abc.def"}` → 400 `invalid_token` / `payload_decode_failed` med korrekt felenvelopp inkl. `request_id`). `org-info` och `sso-claim` svarar `401 missing_auth` utan auth-header (förväntat). **Väntar på Tipspromenad** att utföra ett riktigt `sso-initiate`-anrop till Hub `/sso-exchange` med signerad payload (`b64u(payload).b64u(hmac_sha256(secret, b64u(payload)))`) för att verifiera flödet end-to-end. Hub-sidan blockerad tills dess.


## Backlog
- Edge-funktioner: `provision-org`, `apply-purchase` (Stripe-webhook).
- UI: organisationsväljare, fakturor, medlemshantering, token-revokering (audit-tabell finns).
- Admin-UI för att skapa orgs/medlemmar/entitlements manuellt.
- Launchpad pekar om provisioning till Hub.


## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS (`/api/public/jwks.json`), hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe) — TBD.
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
