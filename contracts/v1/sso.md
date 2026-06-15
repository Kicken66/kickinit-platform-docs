Contract — SSO (Hub ⇄ produkt-appar)
Version: 1.1.0 (additiv, bakåtkompatibel)
Datum: 2026-06-15
Status: STABIL — v1.1.0 lägger till `is_super_admin`-claim. Audience, alg och alla befintliga fält oförändrade. Klienter som inte läser claimen påverkas inte.
Riktning: Tipspromenad / EventIT ⇄ Hub

Versionshistorik
Datum	Version	Ändring
2026-06-15	1.1.0	Additiv: `is_super_admin: boolean` läggs till i Hub-JWT-payload, alltid med (false om ej plattformsadmin). Mintas av både `/sso-claim` och `/sso-exchange`. Värdet hämtas från `public.has_role(user_id, 'super_admin')`. Ingen versions-/audience-bump på själva JWT:n — fältet är bara additivt.
2026-06-07	1.0.0 FROZEN	Fryst efter end-to-end-verifiering: /sso-claim mintar RS256 Hub-JWT (kid hub-2026-06-07), publik JWKS exponerad på /api/public/jwks.json, audit-rad skrivs till hub.sso_audit. Inga brytande ändringar mot 0.1.
2026-06-06	0.1 DRAFT	Första utkast. Definierar flöde B (claim) först, flöde A (issue/exchange) skissat men ej krävt för MVP.

Syfte
Produkt-apparna kör egen Supabase Auth i sina Lovable Cloud-instanser. Hub
äger den kanoniska användar/org/entitlement-modellen. För att produkt-
apparna ska kunna anropa Hubs cross-app-endpoints (t.ex. org-info)
behöver de en Hub-utfärdad JWT att skicka som Authorization: Bearer.

Detta kontrakt definierar hur den token utfärdas.

Token-shape (Hub-JWT)
{
  "iss": "https://hub.kickinit.se",
  "sub": "<hub_user_id>",
  "aud": "tipspromenad" | "eventit",
  "email": "user@example.com",
  "org_id": "<hub_org_id> | null",
  "role": "org_admin" | "org_editor" | "org_viewer" | "super_admin",
  "is_super_admin": true | false,
  "acting_as_org_id": "<hub_org_id>",   // valfri; sätts när super_admin agerar i annan org
  "exp": <now + 3600>,
  "iat": <now>,
  "jti": "<uuid>"
}
Signeras med Hubs JWKS-nyckel (RS256, kid `hub-2026-06-07`). Verifieras av mottagande
endpoint (org-info m.fl.) mot Hubs publika JWKS: https://hub.kickinit.se/api/public/jwks.json

`is_super_admin` (v1.1.0)
- Typ: boolean, alltid med.
- Värde: resultatet av `public.has_role(<hub_user_id>, 'super_admin')` vid mint-tillfället.
- Mintas av båda endpoints: `/sso-claim` (flöde B) och `/sso-exchange` (flöde A).
- Använd för UI-gating i satellitappar (t.ex. EventIT visar `/platform/*` sidebar bara när true).
- SÄKERHET: klient-claim är ENDAST UI-gate. Varje serverside-operation i satellitapp som
  kräver plattformsadmin MÅSTE re-verifiera JWT-signaturen mot Hubs JWKS och läsa
  `is_super_admin` ur den verifierade payloaden. Lita aldrig på opåkad klientstate.
- Skillnad mot `role: "super_admin"`: `role` är *aktiv* roll i nuvarande org-kontext
  (kan vara `super_admin` även när användaren agerar i en specifik org). `is_super_admin`
  är *capability* — sant om användaren har plattformsadmin-rätt globalt, oberoende av
  vilken org token är scopad till. För plattforms-funktioner: använd `is_super_admin`.
  För org-roll-gating: använd `role`.

Flöde B — Claim från redan inloggad app-användare (MVP)
Användaren är inloggad i Tipspromenad. Tipspromenad behöver Hub-JWT.

Tipspromenad → POST {HUB}/sso-claim
  Authorization: Bearer <Tipspromenads Supabase-JWT>
  Body: { "app": "tipspromenad" }

Hub:
  1. Verifierar Authorization-JWT mot appens egen Auth-server (HTTP-introspection
     mot {APP_AUTH_URL}/auth/v1/user — ersatte JWKS-verifiering efter att Lovable
     Cloud började minta HS256 utan kid efter token-refresh).
  2. Extraherar `email` från svaret.
  3. Slår upp public.profiles via email (citext).
  4. Om hittad: läser org-membership, slår upp `is_super_admin` via has_role(),
     mintar Hub-JWT (TTL 1h) med alla claims inkl. `is_super_admin`.
  5. Returnerar token.

Hub → 200 { "hub_jwt": "...", "expires_at": "ISO", "hub_user_id": "...", "email": "..." }
Hub → 401 { "error": { "code": "invalid_token", "message": "..." } }
Hub → 404 { "error": { "code": "no_hub_user", "message": "Onboard via Launchpad" } }
Hub → 409 { "error": { "code": "org_selection_required" }, "orgs": [...] }

Flöde A — Användaren startar i Hub
Hub-UI har "Öppna EventIT/Tipspromenad"-länk. Hub mintar engångskod, satellit
växlar koden mot Hub-JWT via /sso-exchange. Samma `is_super_admin`-claim
inkluderas i den slutgiltiga Hub-JWT:n.

Refresh
Klienter ska claim:a ny token när nuvarande har <5 min kvar. Vid varje ny claim
läser satellitappen om `is_super_admin` och uppdaterar UI-state. Inga refresh-
tokens — bara ny claim mot redan-giltig app-JWT.

Felmodell
Standardform: { "error": { "code": "", "message": "..." } }.

Kod	Betydelse
missing_auth	Authorization-header saknas
invalid_token	App-JWT signaturfel, utgången, eller invalidated av app-Auth
no_hub_user	Email finns inte i public.profiles
app_not_allowed	app-parameter ej i whitelist
org_selection_required	Användaren är medlem i flera orgs; klienten måste skicka ?org=<slug>
org_not_member	Begärd org finns men användaren är inte medlem
internal_error	Oväntat fel

Säkerhetsöverväganden
- App-JWT som skickas till /sso-claim får inte loggas.
- jti på Hub-JWT spåras i hub.sso_audit för audit/återkallning.
- TTL 1h. Klienten claim:ar ny token vid <5 min kvar.
- `is_super_admin` får INTE läggas till i någon utgående app-JWT eller spridas
  utanför satellitappens egen process — den är en capability-flagga för Hub-
  scope, inte för satellitens egna data.
- Satellitappar lagrar aldrig roller själva. All rollinfo kommer från Hubs JWT.

Migrationsnotering för konsumerande appar
- v1.0.0 → v1.1.0 är non-breaking. Befintliga klienter kan ignorera fältet.
- Klienter som vill använda fältet: läs `payload.is_super_admin === true` ur
  den verifierade JWT:n. Default false. Använd för UI-gating; re-verifiera
  alltid serverside för auktorisation.
