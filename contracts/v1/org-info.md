# Contract — `org-info` (Hub → produkt-appar)

**Version:** 1.0.0 (DRAFT — fryses när Hub implementerar)
**Riktning:** Tipspromenad / EventIT → Hub
**Endpoint:** `POST {HUB_FUNCTIONS_URL}/org-info`

## Syfte

Produkt-apparna (Tipspromenad, EventIT) behöver veta vilken organisation
den inloggade användaren tillhör, vilken roll hen har, och vilka
entitlements organisationen har köpt. Hub är master. Produkt-apparna
**lagrar inte** denna information lokalt utöver kort cache.

## Auth

`Authorization: Bearer <hub_jwt>` — JWT utfärdat av Hub vid SSO. Verifieras
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
      "plan": "private",
      "rounds_remaining": 12,
      "days_remaining": 30,
      "addons": ["sponsors", "lottery"]
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

Standardform: `{ "error": { "code": "<snake_case>", "message": "..." } }`.

| Kod | Betydelse |
|---|---|
| `missing_auth` | Authorization-header saknas |
| `invalid_token` | JWT signaturfel eller utgången |
| `no_org` | Användare existerar men har ingen org |
| `internal_error` | Oväntat fel |

## Migration / introduktion

Tas i bruk i Tipspromenad i en separat H.x-tranche **efter** att Hub har:
1. JWKS-endpoint publik
2. `org-info` deployad och stabil i Hubs produktionsinstans
3. Minst en testorg seedad med matchande data mellan Hub och Tipspromenad
