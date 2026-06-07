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

## Pågående arbete
- Flöde A: edge-funktionerna `sso-issue` (Hub-session → engångskod, 60s TTL) och `sso-exchange` (HMAC från appen → Hub-JWT) deployade. Tabellen `hub.sso_codes` (hashad kod, service-role only) på plats. Hub-UI `/apps` ("Mina produkter") med knappar "Öppna Tipspromenad/EventIT" som anropar `sso-issue` och redirectar. Avvaktar konfiguration av `TIPSPROMENAD_CALLBACK_URL` + `TIPSPROMENAD_EXCHANGE_SECRET` (motsvarande för EventIT när appen finns) samt staging-test mot Tipspromenads `/sso-from-hub`.
- **Kontrakt `sso-exchange.md` v0.1 DRAFT** (Tipspromenad, commit `e4afad0`) granskat. Wire-formatet skiljer sig från Hubs nuvarande `sso-exchange`-implementation — kontraktet måste finaliseras innan Tipspromenad implementerar `/sso-from-hub`.

  **Diff mot nuvarande Hub-implementation:**
  | Aspekt | Hub idag | Kontrakt v0.1 |
  |---|---|---|
  | Auth | Headers `X-Hub-App` + `X-Hub-Timestamp` + `X-Hub-Signature` (hex HMAC över `${ts}.${rawBody}`) | Body `{ token: "b64u(payload).b64u(sig)" }`, ingen Authorization-header |
  | Payload-shape | `{ code }` i rå JSON-body | `{ code, app, iat, exp, nonce }` JSON inuti b64u-token |
  | HMAC-input | `${timestamp}.${rawBody}` → hex | `b64u(payload_json)` → b64u (ingen padding) |
  | TTL-skew | 5 min (`MAX_TIMESTAMP_SKEW_MS`) | 60s (`exp - iat ≤ 60`) |
  | Replay-skydd | Implicit via `code` engångskonsumtion | Explicit `nonce` lagras + verifieras + `code` engångskonsumtion |
  | Felkoder | `missing_auth` / `invalid_token` / `app_not_allowed` / `org_selection_required` | `invalid_token` / `app_mismatch` / `token_expired` / `replay_detected` / `code_not_found` / `code_expired` / `code_consumed` |
  | 200-response | `{ hub_jwt, expires_at, hub_user_id, email }` | + `org_id` + `role` |

  **Hub-feedback på öppna frågor i kontraktet:**
  1. **Wire-format** — Hub föreslår att behålla token-i-body (kontraktet som skissat). Headers fungerar men token-strängen är enklare att logga utan att läcka i nätverkspaneler.
  2. **`org_id` + `role` i 200-svaret** — JA, returnera dem. Hub har redan datat efter `resolveOrgForUser()`; sparar appen en `/org-info`-round-trip.
  3. **TTL 60s** — OK. Räcker för synkront edge→edge-anrop; matchar `sso_code`-TTL.
  4. **Audit** — återanvänd `hub.sso_audit` med kolumn `flow` (`"claim" | "exchange"`). Kräver ALTER TABLE i samband med migration av `sso-exchange`.
  5. **Secret-rotation** — hård rotation i koordinerat fönster räcker för v1. Zero-downtime kan adresseras i v1.1 om/när det behövs.

  **Migration som krävs på Hub-sidan när kontraktet låses till v1.0:**
  - Skriv om `supabase/functions/sso-exchange/index.ts` till token-i-body-formatet.
  - Lägg till nonce-replay-skydd (`hub.sso_exchange_nonces` med TTL-cleanup, eller `hub.sso_audit.nonce UNIQUE`).
  - `ALTER TABLE hub.sso_audit ADD COLUMN flow text NOT NULL DEFAULT 'claim'`.
  - Returnera `org_id` + `role` i 200-svaret.
  - Mappa felkoder enligt v1.0-tabellen.

## Backlog
- Edge-funktioner: `provision-org`, `apply-purchase` (Stripe-webhook).
- UI: organisationsväljare, fakturor, medlemshantering, token-revokering (audit-tabell finns).
- Admin-UI för att skapa orgs/medlemmar/entitlements manuellt.
- Launchpad pekar om provisioning till Hub.


## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS (`/api/public/jwks.json`), hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe) — TBD.
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
