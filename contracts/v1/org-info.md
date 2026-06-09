# Contract — `org-info` (Hub → produkt-appar)

**Version:** 1.2.0  
**Datum:** 2026-06-09  
**Riktning:** Tipspromenad / EventIT → Hub  
**Endpoint:** `POST {HUB_FUNCTIONS_URL}/org-info`

## Versionshistorik

| Datum | Version | Ändring |
|---|---|---|
| 2026-06-06 | 1.0.0 | Första release. Hub-schema, JWKS-endpoint och `org-info` verifierade i produktion. DRAFT-tag borttagen. |
| 2026-06-09 | 1.2.0 | EventIT-planer och per-app boolean addon-nycklar tillagda. `entitlements.eventit` returnerar nu samma struktur som `tipspromenad` med `addons[]` + booleans. Ingen fallback krävs i klienten. |

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
      "plan": "club",
      "events_remaining": 1,
      "addons": ["qr_checkin", "attendee_export", "email_invites", "branded_page", "seat_map"],
      "qr_checkin": true,
      "attendee_export": true,
      "email_invites": true,
      "branded_page": true,
      "seat_map": true
    }
  },
  "cached_until": "2026-06-09T12:34:56Z"
}
```

Fältet `cached_until` är en hint till klienten — sätt `staleTime` i
React Query till `cached_until - now`. Servern svarar oavsett om klienten
respekterar hinten eller inte.

## Per-app boolean addon-nycklar

Varje app har en egen uppsättning boolean-nycklar som alltid finns med
i respektive entitlement-objekt. Nycklarnas värde är `true` om nyckeln
finns i `addons[]`, annars `false`. Nycklar från en app läcker aldrig
över till en annan.

| App | Boolean-nycklar |
|---|---|
| **Tipspromenad** | `results_email`, `member_lookup`, `paper_print` |
| **EventIT** | `qr_checkin`, `attendee_export`, `email_invites`, `branded_page`, `seat_map` |

EventIT kan alltså lita på att dessa fem nycklar alltid finns när
`eventit`-objektet returneras — oavsett om raden är explicit i
databasen eller synthesiserad via plan-implicit fallback.

## Plan-implicit fallback

När ingen explicit entitlement-rad finns för den efterfrågade appen
synthesiserar Hub en implicit rad utifrån org-typ:

| App | Org-typ | Plan | Implicita addons |
|---|---|---|---|
| Tipspromenad | `association` | `club` | `results_email`, `member_lookup`, `paper_print`, `sponsor_control` |
| Tipspromenad | `private` | `standard` | *(inga)* |
| EventIT | `association` | `club` | `qr_checkin`, `attendee_export`, `email_invites`, `branded_page`, `seat_map` |
| EventIT | `private` | `standard` | `qr_checkin`, `attendee_export`, `email_invites` |

En implicit rad markeras med `"implicit": true` så att klienten kan
skilja den från ett köpt abonnemang. Explicita rader har inget
`implicit`-fält.

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
- EventIT: läs boolean-nycklarna (`qr_checkin`, `attendee_export`, `email_invites`, `branded_page`, `seat_map`) direkt — ingen klient-fallback behövs.
