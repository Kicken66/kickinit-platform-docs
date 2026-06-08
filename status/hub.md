# Hub – status

## Nuläge
Hub-projektet skapat. Egen Lovable Cloud-instans provisionerad. Auth (email/password + Google) på plats. `user_roles` + `has_role` + `docs_sync_log` migrerade. Edge-funktionen `docs-sync` deployad. Admin-UI på `/super-admin/docs-sync`.

`hub.*`-schema live: `organizations`, `org_members`, `entitlements` med enums `org_type`, `member_role`, `app_kind`. RLS: medlemmar läser egen org/membership/entitlements; super_admin har full access; service-role bypassar (används av edge-funktioner). Schemat exponerat via PostgREST.

Edge-funktionen `org-info` deployad — verifierar nu Hub-utfärdad RS256-JWT (`iss=https://hub.kickinit.se`, `aud=tipspromenad|eventit`) och returnerar `{ user, org, role, entitlements, cached_until }` enligt `contracts/v1/org-info.md`. Felmodell: `missing_auth` (401), `invalid_token` (401), `no_org` (404).

Kontraktet `contracts/v1/org-info.md` bumpat till **v1.1.0** — addons-nycklarna `results_email`, `member_lookup`, `paper_print` exponeras nu även som top-level boolean-fält i `entitlements.tipspromenad` (additivt; `addons[]` behålls). Stänger Tipspromenads gap (migration `0005-hub-org-info-entitlements-gap.md`).

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

## Flöde B end-to-end ✅ verifierat 2026-06-08
Kod-utgivaren (`sso-issue`) fanns redan deployad från Flöde A-spåret — ingen ny endpoint behövdes. Full genuin kedja körd mot deployad miljö:

1. **`POST /sso-issue`** med Hub-användarens Supabase Auth-bearer (`{app:"tipspromenad"}`) → `200` med `sso_code` (256-bit, base64url), `expires_in: 60`, `redirect_url: https://app.kickinit.se/sso/callback?code=…`. SHA-256-hash av koden persisteras i `hub.sso_codes` med `hub_user_id`, `app`, `expires_at`.
2. **`POST /sso-exchange`** med `{token: b64u(payload).b64u(hmac_sha256(TIPSPROMENAD_EXCHANGE_SECRET, b64u(payload)))}` där `payload = {code, app:"tipspromenad", iat, exp, nonce}` → `200` med `hub_jwt` (RS256, `iss=https://hub.kickinit.se`, `aud=tipspromenad`, 1h TTL), `hub_user_id`, `email`, `org_id`, `role`. Koden konsumerades atomiskt (`consumed_at` satt), audit-rad skriven i `hub.sso_audit` med `flow='exchange'`, nonce, `request_id`.
3. **`GET /org-info?app=tipspromenad`** med `Authorization: Bearer <hub_jwt>` → `200` med `{user, org:{id, slug:"hbk", name:"Hedareds BK", type:"association"}, role:"super_admin", entitlements:{tipspromenad:{plan, addons, rounds_remaining, days_remaining}}, cached_until}`.

Hub-sidan av Flöde B är **produktionsklar**. Tipspromenad kan nu byta sin app-utfärdade SSO-payload mot en genuin Hub-JWT utan syntetiska koder.

## org-info v1.1.0 — boolean addon-keys ✅ 2026-06-08
Som svar på Tipspromenads `0005-hub-org-info-entitlements-gap.md`: `org-info` returnerar nu `results_email`, `member_lookup` och `paper_print` som top-level boolean-fält på `entitlements.tipspromenad` utöver `addons[]`-arrayen. Gäller både explicit-rader (booleanen härleds från `addons[]`-innehåll) och plan-implicit-fallback för `association` (alla tre = `true`) / `private` (alla tre = `false`). Bakåtkompatibelt — `addons[]` består. Kontraktet bumpat till v1.1.0 i kickinit-platform-docs.


## Backlog
- Edge-funktioner: `provision-org`, `apply-purchase` (Stripe-webhook).
- UI: organisationsväljare, fakturor, medlemshantering, token-revokering (audit-tabell finns).
- Admin-UI för att skapa orgs/medlemmar/entitlements manuellt.
- Launchpad pekar om provisioning till Hub.


## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS (`/api/public/jwks.json`), hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe) — TBD.
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
