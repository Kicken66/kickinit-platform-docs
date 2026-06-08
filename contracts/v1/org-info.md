# Contract — `org-info` (Hub → produkt-appar)

**Version:** 1.1.0  
**Datum:** 2026-06-08  
**Riktning:** Tipspromenad / EventIT → Hub  
**Endpoint:** `POST {HUB_FUNCTIONS_URL}/org-info`

## Versionshistorik

| Datum | Version | Ändring |
|---|---|---|
| 2026-06-06 | 1.0.0 | Första release. Hub-schema, JWKS-endpoint och `org-info` verifierade i produktion. DRAFT-tag borttagen. |
| 2026-06-08 | 1.1.0 | Additivt: kända addon-nycklar (`results_email`, `member_lookup`, `paper_print`) exponeras även som top-level boolean-fält i `entitlements.<app>` utöver `addons[]`. Plan-implicit-regeln (`club`/`standard`/`association`) gäller. Bakåtkompatibelt — `addons[]` bibehålls. |

## Syfte

Produkt-apparna (Tipspromenad, EventIT) behöver veta vilken organisation
den inloggade användaren tillhör, vilken roll hen har, och vilka
entitlements organisationen har köpt. Hub är master. Produkt-apparna
**lagrar inte** denna information lokalt utöver kort cache.

## Auth

`Authorization: Bearer <jwt>` — JWT utfärdat av Hub vid SSO. Verifieras
av `org-info` med Hubs interna JWKS (samma instans), och kan separat
verifieras av produkt-appen mot Hubs publika `/jwks.json`.

## Request

Ingen body (eller `{}`). Användaren identifieras via tokenens `sub`.

Valfri query param `?app=tipspromenad|eventit` används för att filtrera
entitlements till relevant app. Saknas → returnera alla.

## Response 200

```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com"
  },
  "org": {
    "id": "uuid",
    "slug": "hbk",
    "name": "Hedareds BK",
    "type": "association"
  },
  "role": "org_admin",
  "entitlements": {
    "tipspromenad": {
      "plan": "club",
      "rounds_remaining": 12,
      "days_remaining": 30,
      "addons": ["results_email", "member_lookup", "paper_print", "sponsor_control"],
      "results_email": true,
      "member_lookup": true,
      "paper_print": true
    },
    "eventit": {
      "plan": "trial",
      "events_remaining": 1
    }
  },
  "cached_until": "2026-06-06T12:34:56Z"
}
```

Fältet `cached_until` är en hint till klienten — sätt `staleTime` i
React Query till `cached_until - now`. Servern svarar oavsett om klienten
respekterar hinten eller inte.

## Boolean addon-nycklar (v1.1.0)

Utöver `addons[]`-arrayen exponerar Hub följande nycklar som top-level
boolean-fält i `entitlements.<app>` när de är aktiva:

| App | Nyckel | Källa |
|---|---|---|
| `tipspromenad` | `results_email` | `addons[]` innehåller `results_email` ELLER plan-implicit (`club`/`standard`/`association`) |
| `tipspromenad` | `member_lookup` | `addons[]` innehåller `member_lookup` ELLER plan-implicit |
| `tipspromenad` | `paper_print`   | `addons[]` innehåller `paper_print` ELLER plan-implicit |

Apparna SKA läsa boolean-formen i ny kod. `addons[]`-arrayen behålls
bakåtkompatibelt och får inte tas bort utan major-bump.

## Response 401

Token saknas eller är ogiltig. Klient ska kasta tillbaka till SSO-flödet.

## Response 404

Användaren har ingen org i Hub. Klient visar "Ingen organisation"-vy och
länk till Launchpad/onboarding.

## Cache-strategi

- Klient: React Query `staleTime: 60_000`, `gcTime: 5 * 60_000`.
- Edge: `Cache-Control: private, max-age=60`.
- Invalidering: när `apply-purchase` har körts kan klienten anropa
  `org-info?refresh=1` för att hoppa över cachen.

## Felmodell

Standardform: `{ "error": { "code": "", "message": "..." } }`.

| Kod | Betydelse |
|---|---|
| `missing_auth` | Authorization-header saknas |
| `invalid_token` | JWT signaturfel eller utgången |
| `no_org` | Användare existerar men har ingen org |
| `internal_error` | Oväntat fel |

## Migration / introduktion

Kontraktet är aktivt sedan 2026-06-06. Hub har:
1. ✅ JWKS-endpoint publik (`/api/public/jwks.json`)
2. ✅ `org-info` deployad och stabil i Hubs produktionsinstans
3. ✅ `hub.*`-schema med organisationer, medlemskap och entitlements
4. ✅ Minst en testorg seedad med matchande data

Nästa steg för produkt-appar (Tipspromenad / EventIT):
- Hämta org-info vid inloggning via SSO.
- Cacha svaret enligt `cached_until`.
- Använd `role` och `entitlements` för att styra UI och funktionsåtkomst.
