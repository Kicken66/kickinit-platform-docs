Contract — SSO (Hub ⇄ produkt-appar)
Version: 1.0.0 (FROZEN)
Datum: 2026-06-07
Status: FROZEN — v1.0.0 låst 2026-06-07. Implementation verifierad mot live Hub (/sso-claim) med RS256 + JWKS. Brytande ändringar kräver v2.
Riktning: Tipspromenad / EventIT ⇄ Hub

Versionshistorik
Datum	Version	Ändring
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
  "org_id": "<hub_org_id>",
  "role": "org_admin" | "org_editor" | "org_viewer" | "super_admin",
  "exp": <now + 3600>,
  "iat": <now>,
  "jti": "<uuid>"
}
Signeras med Hubs JWKS-nyckel. Verifieras av mottagande endpoint (org-info m.fl.)
mot Hubs interna JWKS.

Flöde B — Claim från redan inloggad app-användare (MVP)
Användaren är inloggad i Tipspromenad. Tipspromenad behöver Hub-JWT.

Tipspromenad → POST {HUB}/sso-claim
  Authorization: Bearer <Tipspromenads Supabase-JWT>
  Body: { "app": "tipspromenad" }

Hub:
  1. Verifierar Authorization-JWT mot Tipspromenads JWKS
     (Hub har whitelist över tillåtna app-JWKS-URL:er).
  2. Extraherar `email` från claims.
  3. Slår upp hub.users via email (case-insensitive).
  4. Om hittad: läser org-membership, mintar Hub-JWT (TTL 1h).
  5. Returnerar token.

Hub → 200 { "hub_jwt": "...", "expires_at": "ISO", "hub_user_id": "...", "email": "..." }
Hub → 401 { "error": { "code": "invalid_token", "message": "..." } }
Hub → 404 { "error": { "code": "no_hub_user", "message": "Onboard via Launchpad" } }
Krav på Hub-sidan
Endpoint POST /functions/v1/sso-claim.
JWKS-whitelist i Hub-config:
tipspromenad: https://wkmtgihiveqwyvozrlid.supabase.co/auth/v1/.well-known/jwks.json
eventit: (kommer)
hub.users-uppslag på email.
Token-mintning med Hubs signeringsnyckel (samma som org-info verifierar mot).
Flöde A — Användaren startar i Hub (senare)
Hub-UI har "Öppna Tipspromenad"-länk.

Hub UI → POST {HUB}/sso-issue { app: "tipspromenad" }   (Hub-session)
       ← { sso_code, expires_in: 60 }
Redirect: https://tipstjanst.online/sso/callback?code=<sso_code>

Tipspromenad → POST {SELF}/sso-from-hub { code }
  Server-side:
    → POST {HUB}/sso-exchange { code }   (HMAC mellan apps)
    ← { hub_jwt, expires_at, email }
    → supabase.auth.admin.generateLink({ type: "magiclink", email })
    ← magic_link
  Klient följer magic_link → lokal session etableras.
  Hub-JWT levereras tillbaka till klient för sessionStorage.
Mönstret är identiskt med befintliga sso-from-launchpad i Tipspromenad.

Refresh
Klienter ska claim:a ny token när nuvarande har <5 min kvar. Inga refresh-tokens
i v0.1 — bara ny claim mot redan-giltig app-JWT.

Felmodell
Standardform: { "error": { "code": "", "message": "..." } }.

Kod	Betydelse
missing_auth	Authorization-header saknas
invalid_token	App-JWT signaturfel, utgången, eller från oviterad JWKS
no_hub_user	Email finns inte i hub.users
app_not_allowed	app-parameter ej i whitelist
internal_error	Oväntat fel
Säkerhetsöverväganden
App-JWT som skickas till /sso-claim får inte loggas.
jti på Hub-JWT bör spåras i Hub för att kunna återkalla enskilda tokens.
TTL 1h är en kompromiss mellan UX och blast radius. Diskutera om 15 min är bättre och låt klienten claim:a oftare.
Email som lookup-nyckel kräver att produkt-apparna validerar e-postägarskap (Supabase Auth gör det redan).
Öppna frågor
Ska /sso-claim även returnera org-info-payload i samma svar för att spara en rundtur? (Optimerar MVP, men kopplar ihop kontrakten.)
Hur hanterar vi användare med medlemskap i flera orgs? ?org=<slug> på claim, eller returnera array och låt klienten välja?
Ska super_admin kunna agera som vilken org som helst, och i så fall hur uttrycks det i token? (org_id: null + separat capability?)
Nästa steg
Hub-team granskar och godkänner v0.1.
Hub implementerar /sso-claim mot Tipspromenads JWKS.
Tipspromenad har klient klar (HubSessionProvider + useHubOrgInfo); väntar på Hub-deploy.
Frys till v1.0.0 när första lyckade end-to-end-claim verifierats i staging.