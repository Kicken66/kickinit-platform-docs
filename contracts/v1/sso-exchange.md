Contract — sso-exchange (Hub ⇄ produkt-appar)
Version: 0.1 DRAFT — föreslagen av Tipspromenad 2026-06-07
Status: Väntar Hub-bekräftelse innan Tipspromenad implementerar /sso-from-hub
Riktning: produkt-app (server-side edge) → Hub
Relaterat: contracts/v1/sso.md v1.0.0 (flöde A skissat, wire-format för exchange ospecificerat)

Syfte
Flöde A i sso.md: Hub utfärdar engångskod via /sso-issue och redirectar
användaren till {APP}/sso/callback?code=<sso_code>. Appens edge-funktion
sso-from-hub växlar koden mot en Hub-JWT genom server-till-server-anrop
mot POST {HUB}/sso-exchange. Detta dokument låser wire-format,
HMAC-input och felmodell för det anropet.

Mönstret speglar sso-from-launchpad v1.0.0 i Tipspromenad (samma
b64u(payload).b64u(sig)-token, samma TTL-regler, samma felkoder) så att
båda riktningarna delar samma verifieringskod på Tipspromenads sida.

Request — POST {HUB}/functions/v1/sso-exchange
Headers
Content-Type: application/json
X-Correlation-Id: <uuid>          # optional, ekas i felresponse
Ingen Authorization-header. Autentiseringen sker via HMAC-signerad
payload (se nedan). Detta hindrar att en stulen kod kan växlas in från
fel app — bara den app som delar HMAC-secret med Hub kan signera.

Body
{
  "token": "<b64u(payload)>.<b64u(hmac_sha256(secret, b64u(payload)))>"
}
payload är JSON med följande shape:

{
  "code": "<sso_code från /sso-issue>",
  "app":  "tipspromenad",
  "iat":  1717787149,
  "exp":  1717787209,
  "nonce": "<uuid v4>"
}
Regler:

Fält	Krav
code	exakt strängen från /sso-issue-svaret, oförändrad
app	"tipspromenad" eller "eventit" — måste matcha appen som anropar
iat	unix-sekunder, nu
exp	unix-sekunder, iat + ≤ 60s (Hub avvisar längre TTL)
nonce	slumpad uuid v4, lagras av Hub i hub.sso_codes/audit för replay-skydd
code är redan en engångskod med 60s TTL från /sso-issue, men
HMAC-payloadens TTL skyddar mot att samma signerade exchange-anrop
replayas av en man-in-the-middle som snor token-fältet.

HMAC-input (exakt)
signature = b64u( HMAC_SHA256( shared_secret_bytes, ascii_bytes( b64u(payload_json_utf8) ) ) )
b64u = base64url utan padding (/ → _, + → -, inga =).
Hashen körs över base64url-strängen, inte över rå JSON. Detta gör
signaturen oberoende av JSON-key-ordning och whitespace.
shared_secret_bytes = UTF-8 bytes av miljövariabeln (se "Secrets" nedan).
Detta är identiskt med hmacSha256()-helpern i Tipspromenads
supabase/functions/sso-from-launchpad/index.ts (rader 16–37). Hub kan
återanvända samma Deno-kod på sin sida.

Response
200 OK
{
  "hub_jwt": "<RS256 Hub-JWT, kid=hub-2026-06-07>",
  "expires_at": "2026-06-07T19:05:49Z",
  "hub_user_id": "<uuid>",
  "email": "user@example.com",
  "org_id": "<uuid>",
  "role": "org_admin"
}
Samma hub_jwt-shape som /sso-claim returnerar (kontrakt frozen i
sso.md §"Token-shape"). Appen lagrar hub_jwt + expires_at i sin
HubSessionProvider (sessionStorage) och kan därefter anropa
/org-info direkt utan att gå via /sso-claim.

org_id + role ingår så att appen kan visa rätt org i UI:t direkt
efter callback utan extra round-trip till /org-info. Får utelämnas
om Hub inte vill committa — appen klarar sig med bara hub_jwt.

Felmodell
Standardform: { "error": { "code": "...", "message": "...", "request_id": "..." } }

Status	Kod	När
400	invalid_token	Token malformed (saknar ., b64u-decode-fel, HMAC mismatch, payload ej JSON, TTL > 60s, saknar required fields)
400	app_mismatch	payload.app matchar inte appen som secret är registrerad för
401	token_expired	payload.exp < now
401	replay_detected	payload.nonce har använts förut
404	code_not_found	payload.code finns inte i hub.sso_codes (eller redan konsumerad)
410	code_expired	Koden finns men TTL har passerat (Hub satte 60s i /sso-issue)
410	code_consumed	Koden har redan växlats in en gång
500	internal_error	Oväntat fel
Tipspromenad mappar dessa direkt till klient-felen i HubSessionContext

visar dem i debug-panelen på /super-admin/docs-sync.
Secrets
Symmetrisk HMAC-secret per app, delad mellan Hub och appen:

Sida	Secret-namn	Värde
Hub	TIPSPROMENAD_EXCHANGE_SECRET	Slumpad 32-byte b64-sträng
Tipspromenad	HUB_EXCHANGE_SECRET	samma värde
Föreslagen process:

Hub genererar värdet (openssl rand -base64 32) och lagrar som
TIPSPROMENAD_EXCHANGE_SECRET.
Hub delar värdet med Tipspromenad-ägaren via säker kanal (1Password
shared item eller motsvarande).
Tipspromenad lägger in det som HUB_EXCHANGE_SECRET via Lovables
secrets-UI.
Rotation: båda sidor byter samtidigt; ingen overlap behövs eftersom
exchange-anropet är synkront och kortlivat.
Samma mönster appliceras för EventIT med
EVENTIT_EXCHANGE_SECRET / HUB_EXCHANGE_SECRET (per-app secret på
EventIT-sidan, per-app secret på Hub-sidan).

Callback-URL
Tipspromenads /sso-issue-konfiguration på Hub-sidan ska peka på:

TIPSPROMENAD_CALLBACK_URL = https://app.kickinit.se/sso/callback
För staging kan Hub temporärt peka på Lovable-preview-URL — men eftersom
sso_code:n är kortlivad (60s) räcker det att appen är publicerad innan
slutgiltig verifiering.

End-to-end-sekvens
Hub UI  ──POST /sso-issue { app:"tipspromenad" }──▶ Hub
        ◀─{ sso_code, expires_in: 60 }──
        302 → https://app.kickinit.se/sso/callback?code=<sso_code>

Browser hits /sso/callback (Tipspromenad SPA-route)
  client → POST {SELF}/functions/v1/sso-from-hub { code }

Tipspromenad edge `sso-from-hub`:
  1. Bygger payload { code, app:"tipspromenad", iat, exp:iat+60, nonce:uuid() }
  2. Signerar: token = b64u(payload) + "." + b64u(hmac_sha256(HUB_EXCHANGE_SECRET, b64u(payload)))
  3. POST {HUB}/functions/v1/sso-exchange  { token }
  4. På 200: lagrar { hub_jwt, expires_at } för klient,
     genererar magic_link mot lokala Supabase Auth (auth.admin.generateLink, type:"magiclink", email)
  5. Returnerar { magic_link, hub_jwt, expires_at } till klient

Browser:
  - sessionStorage["kickinit.hubToken.v1"] = { jwt: hub_jwt, expires_at, ... }
  - window.location.href = magic_link
  - magic_link → /auth/callback → lokal Supabase-session etableras
  - användaren landar på /dashboard, redan med Hub-JWT i sessionStorage
Curl-exempel
1. Skapa exchange-token lokalt (för manuell test)
SECRET='<HUB_EXCHANGE_SECRET-värdet>'
CODE='abc123-from-sso-issue'

PAYLOAD=$(jq -nc \
  --arg code "$CODE" \
  --arg nonce "$(uuidgen)" \
  --argjson iat $(date +%s) \
  --argjson exp $(( $(date +%s) + 60 )) \
  '{code:$code, app:"tipspromenad", iat:$iat, exp:$exp, nonce:$nonce}')

P_B64=$(printf '%s' "$PAYLOAD" | basenc --base64url -w0 | tr -d '=')
SIG=$(printf '%s' "$P_B64" \
  | openssl dgst -sha256 -hmac "$SECRET" -binary \
  | basenc --base64url -w0 | tr -d '=')
TOKEN="$P_B64.$SIG"
2. Anropa Hub
curl -sS -X POST "https://yeyubjxotynuelcahwpy.supabase.co/functions/v1/sso-exchange" \
  -H 'Content-Type: application/json' \
  -H "X-Correlation-Id: $(uuidgen)" \
  -d "{\"token\":\"$TOKEN\"}"
3. Förväntat svar
{
  "hub_jwt": "eyJhbGciOiJSUzI1NiIsImtpZCI6Imh1Yi0yMDI2LTA2LTA3In0...",
  "expires_at": "2026-06-07T20:05:49.000Z",
  "hub_user_id": "8c2a...",
  "email": "kicken@kickinit.se",
  "org_id": "0d1f...",
  "role": "super_admin"
}
4. Felexempel — HMAC mismatch
# Förvränger signatur:
curl -sS -X POST "https://.../sso-exchange" -d "{\"token\":\"$P_B64.BADSIG\"}"
# →
# {"error":{"code":"invalid_token","message":"Signature mismatch","request_id":"..."}}
Öppna frågor till Hub-teamet
Wire-format OK? Eller föredrar Hub X-App-Signature-header
rå JSON-body istället för token-strängen? (Header-varianten är
marginellt enklare att debugga i nätverkspaneler.)
org_id + role i 200-svaret — vill Hub returnera dem direkt,
eller ska appen alltid gå via /org-info efter exchange?
TTL 60s på HMAC-payload rimligt? Eller bara 30s för att minska
klock-skew-risken?
Audit: ska sso-exchange skriva en separat rad i
hub.sso_audit, eller återanvända samma tabell som /sso-claim
med en flow discriminator ("claim" | "exchange")?
Secret-rotation: behöver vi stödja två aktiva secrets samtidigt
(för zero-downtime-rotation), eller räcker hård rotation i koordinerat
fönster?
Nästa steg
Hub-teamet granskar, justerar oklarheter ovan, pushar v0.1 → v1.0
i contracts/v1/sso-exchange.md (eller mergar in i sso.md v1.1).
Hub deployar /sso-exchange enligt slutgiltig spec, registrerar
TIPSPROMENAD_CALLBACK_URL + TIPSPROMENAD_EXCHANGE_SECRET.
Tipspromenad implementerar edge-funktion sso-from-hub +
route /sso/callback + secret HUB_EXCHANGE_SECRET.
End-to-end-smoke från Hub-UI → app, dokumenteras i båda
status/hub.md och status/tipspromenad.md.