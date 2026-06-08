# Migration 0003 — EventIT-integration (Hub-sidan klar)

**Status:** Hub-sidan produktionsklar (2026-06-08). Väntar på EventIT-projektets implementation.
**Scope:** Kontrakt + infrastruktur som EventIT behöver konsumera från Hub. Inga nya tabeller i Hub — allt återanvänder befintliga `hub.*`-objekt och edge-funktioner.

## Mål
EventIT är en separat Lovable Cloud-instans med egen Supabase Auth. Användaren loggar in i EventIT lokalt; EventIT byter sin lokala session mot en Hub-utfärdad RS256-JWT och anropar därefter `org-info` / `apply-purchase` mot Hub. EventIT lagrar **inte** org/medlems/entitlement-data — Hub är master.

Mönstret är identiskt med Tipspromenad-integrationen som frystes 2026-06-07/08.

## Hub-sidan: vad finns redan

| Komponent | Status | Referens |
|---|---|---|
| `appConfig("eventit")` i `_shared/hub-jwt.ts` | ✓ läser `EVENTIT_CALLBACK_URL`, `EVENTIT_EXCHANGE_SECRET`, `EVENTIT_AUTH_URL`, `EVENTIT_ANON_KEY` | `supabase/functions/_shared/hub-jwt.ts` |
| `sso-claim` (flöde B) | ✓ accepterar `app:"eventit"` så snart `EVENTIT_AUTH_URL` + `EVENTIT_ANON_KEY` är satta | `contracts/v1/sso.md` v1.0.0 |
| `sso-issue` (flöde A) | ✓ accepterar `app:"eventit"`, kräver `EVENTIT_CALLBACK_URL` | `contracts/v1/sso.md` v1.0.0 |
| `sso-exchange` | ✓ accepterar `app:"eventit"`, HMAC mot `EVENTIT_EXCHANGE_SECRET` (b64u-token-format) | `contracts/v1/sso-exchange.md` v1.0.0 |
| `org-info` | ✓ verifierar `aud=eventit`, returnerar `entitlements.eventit` (plan-implicit) | `contracts/v1/org-info.md` v1.1.0 |
| `apply-purchase` | ✓ tar `app:"eventit"` i payload | `contracts/v1/apply-purchase.md` |
| `members-registry` (auto-claim) | ✓ org-medlemskap som EventIT läser via `org-info` | `migration/0002-members-registry.md` |
| Hub-UI "Öppna EventIT" | ✓ kort på `/apps` anropar `sso-issue` med `app:"eventit"` | `src/routes/apps.tsx` |
| Publik JWKS | ✓ `/api/public/jwks.json` (stabil URL) | — |

## Hub-sidan: vad EventIT-projektet måste göra

1. **Mint EventIT-secrets på Hub:**
   - `EVENTIT_AUTH_URL` — `https://<eventit-project-ref>.supabase.co`
   - `EVENTIT_ANON_KEY` — EventIT Supabase anon-key (krävs för `sso-claim` HTTP-introspection)
   - `EVENTIT_CALLBACK_URL` — t.ex. `https://event.kickinit.se/sso/callback`
   - `EVENTIT_EXCHANGE_SECRET` — delad HMAC-secret med EventIT (b64u-token-format)

2. **EventIT-sidans implementation** (i EventIT-projektet, inte Hub):
   - **SSO-callback** `/sso/callback?code=…&org=…`: POST `/sso-exchange` med `{token}` (b64u-payload+HMAC). Resultat: `{ hub_jwt, hub_user_id, email, org_id, role }`. Använd `supabase.auth.admin.generateLink({type:"magiclink",email})` för att etablera lokal session, behåll `hub_jwt` i sessionStorage.
   - **SSO-claim** (när användaren redan är inloggad i EventIT): POST `/sso-claim` med `Authorization: Bearer <eventit-session-jwt>` + `{app:"eventit"}` → får tillbaka `hub_jwt`.
   - **JWT-verifiering**: validera Hub-JWT mot `https://hub.kickinit.se/api/public/jwks.json` (cache 5–10 min). `iss=https://hub.kickinit.se`, `aud=eventit`, RS256.
   - **org-info-klient**: `GET {HUB}/org-info?app=eventit` med `Authorization: Bearer <hub_jwt>`. Cache enligt `cached_until` i svaret.
   - **Refresh**: claim ny token när <5 min kvar av TTL.

## Entitlement-modell för EventIT

Plan-implicit fallback i `org-info` (samma princip som Tipspromenad enligt `0001c-hub-entitlement-seed.md`):
- `org_type='association'` utan explicit `hub.entitlements`-rad → `entitlements.eventit = { plan: "club", addons: […], implicit: true }`
- `org_type='private'` utan explicit rad → `entitlements.eventit = { plan: "standard", addons: [], implicit: true }`

**TBD:** Konkret EventIT addon-katalog och boolean-keys. Skapas i `contracts/v1/entitlement_keys.md` när EventIT-projektet specat vilka features som ska gateas. Implementeras analogt med Tipspromenads `results_email` / `member_lookup` / `paper_print` (v1.1.0). När addon-listan landar skapas `migration/0004-eventit-entitlements.md` och `org-info` uppdateras.

## Verifieringskriterier (gate för "EventIT-integration live")

1. EventIT POSTar `sso-exchange` med `app:"eventit"` → får `hub_jwt` med `aud=eventit` + audit-rad i `hub.sso_audit` med `flow='exchange'`.
2. EventIT POSTar `sso-claim` med EventIT-session-JWT → får `hub_jwt`.
3. EventIT GETar `org-info?app=eventit` med Hub-JWT → får `{user, org, role, entitlements:{eventit:{…}}}`.
4. Replay-skydd: andra `sso-exchange`-anrop med samma `nonce` → `409 replay_detected`.
5. `apply-purchase` med `app:"eventit"` skapar/uppdaterar `hub.entitlements`-rad och `org-info` reflekterar det vid nästa anrop.

## Öppna punkter

- [ ] EventIT Lovable-projekt skapas (utanför Hub-scopet).
- [ ] EventIT-secrets sätts på Hub (`EVENTIT_*`).
- [ ] EventIT addon-katalog spec:as → `entitlement_keys.md` bumpas → `org-info` v1.2.0.
- [ ] Verifieringskriterierna ovan körs end-to-end mot live miljö.
- [ ] EventIT JWKS-URL whitelistas i `sso.md` (rad "eventit: (kommer)" uppdateras).
