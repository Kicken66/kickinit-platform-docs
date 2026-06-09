# Entitlement keys — v1

Status: Accepted (v1.1.0)
Datum: 2026-06-09
Berör: Hub (master för entitlements cross-app), Tipspromenad, EventIT
Refererar: `contracts/v1/org-info.md` (fältet `entitlements.<app>.addons[]` + per-app boolean addon-keys)

## Syfte

Definierar den **gemensamma nyckeluppsättningen** för per-app addons som
Hub returnerar i `org-info.entitlements.<app>.addons[]` och som top-level
boolean-fält. Apparna mappar dessa nycklar till sina lokala
feature-toggles. Nya nycklar läggs till här först, sedan i respektive
apps mapping-lager.

Nyckelregel: `snake_case`, stabila för all framtid, aldrig återanvänd.
**Boolean-keys är per-app** — `results_email` är Tipspromenads egendom
och får aldrig dyka upp på `entitlements.eventit`, och vice versa.

## Tipspromenad (`tipspromenad`)

### Plans

| Plan | För | Beskrivning |
|------------|------------|---------------------------------------------------------------|
| `club` | Förening | Obegränsat antal rundor per säsong + alla baspaket-addons. |
| `standard` | Privat | Per-rundeköp; samma baspaket-addons aktiva som `club`. |
| `private` | Privat (legacy) | Bakåtkompatibel benämning som förekommer i seedade rader. |

### Addons

| Key | Beskrivning | Lokal feature-key (TP) | Plan-implicit för? |
|------------------|-----------------------------------------------------------------------------|------------------------|------------------------------|
| `results_email` | Skicka resultat-mail till deltagare efter rundan. | `results_email` | `club`, `standard` (alltid) |
| `member_lookup` | Slå upp deltagare mot organisationens medlemsregister. | `member_lookup` | `club`, `standard` (alltid) |
| `paper_print` | Skriva ut och rätta papperslappar (PDF + manuell inmatning). | `paper_print` | `club`, `standard` (alltid) |
| `sponsor_control`| Egna sponsorer + sponsor-portal (annars plattformssponsorer). | `sponsor_control` | — (separat tillval alltid) |
| `quiz_other_modes` | Permanenta slingor/serier (`loop`, `series`, `trail`) på privat-plan. | `quiz_other_modes` (i `organization_subscriptions`, inte `org_entitlements`) | `club`, `standard` (alltid) |

### Plan-implicit-regel

För `club`/`standard`-plan i Tipspromenad är `results_email`,
`member_lookup`, `paper_print` och `quiz_other_modes` **alltid sanna**
oavsett om en `org_entitlements`-rad finns lokalt. Hub MÅSTE därför
returnera dessa addons för club/standard-orgs **utan** att kräva
explicita rader i Hubs entitlements-tabell — antingen via plan-implicit
logik i `org-info`-handlern eller via seeded explicit-rader.

Endast `sponsor_control` (och framtida tillval) kräver explicit grant.

Speglar `src/hooks/useOrgEntitlements.ts` i Tipspromenad.

## EventIT (`eventit`)

### Plans

| Plan | För | Beskrivning |
|------------|------------|----------------------------------------------------------------------|
| `club` | Förening | Obegränsat antal events per säsong + fullt baspaket av addons. |
| `standard` | Privat | Per-event-köp; basaddons för incheckning/export/mail aktiva. |
| `trial` | Alla | 1 event, inga addons. Default när `apply-purchase` ännu inte körts. |

### Addons

| Key | Beskrivning | Lokal feature-key (EventIT) | Plan-implicit för? |
|-------------------|-----------------------------------------------------------------------------|-----------------------------|------------------------------|
| `qr_checkin` | QR-incheckning vid entré (scanner-vy + närvarolista). | `qr_checkin` | `club`, `standard` (alltid) |
| `attendee_export` | Exportera deltagarlista (CSV/Excel). | `attendee_export` | `club`, `standard` (alltid) |
| `email_invites` | Skicka mailinbjudningar och påminnelser från EventIT. | `email_invites` | `club`, `standard` (alltid) |
| `branded_page` | Egen logotyp/färger på publik eventsida (annars plattformsbranding). | `branded_page` | `club` (alltid) |
| `seat_map` | Numrerade platser / bordsplacering. | `seat_map` | `club` (alltid) |
| `sponsor_control` | Egna sponsorer på eventsida + sponsor-portal. | `sponsor_control` | — (separat tillval alltid) |
| `payments` | Ta betalt för biljetter via Stripe. | `payments` | — (separat tillval alltid) |

### Plan-implicit-regel

För EventIT gäller följande fallback när ingen explicit
`hub.entitlements`-rad finns för orgen (mappat på `organizations.type`):

- `association` → `plan="club"`, addons = `[qr_checkin, attendee_export, email_invites, branded_page, seat_map]`, alla fem boolean-keys = `true`, `implicit: true`.
- `private` → `plan="standard"`, addons = `[qr_checkin, attendee_export, email_invites]`, dessa tre boolean-keys = `true`, `branded_page` + `seat_map` = `false`, `implicit: true`.
- Okänd `org_type` eller när handlern inte kan resolva → `plan="trial"` med `events_remaining: 1`, inga addons (skapas typiskt som explicit rad av `apply-purchase` vid första onboarding).

`sponsor_control` och `payments` kräver alltid explicit grant — de är
tillval som styrs av separat köp via `apply-purchase` och dyker aldrig
upp i en plan-implicit fallback.

Samma princip som Tipspromenad: `club`/`standard`-orgs får baspaketet
**utan** att Hubs `entitlements`-tabell måste seedas på förhand.

## Lägga till en ny nyckel

1. PR till denna fil med ny rad i tabellen (key + beskrivning + lokal mappning).
2. Hubs `org-info`-handler uppdateras att returnera nyckeln när tillämpligt
   (`BOOLEAN_ADDON_KEYS_BY_APP` i `supabase/functions/org-info/entitlements.ts`).
3. Mottagande app uppdaterar sitt mapping-lager + UI.
4. Bumpa version i headern (patch = ny addon, minor = ny plan eller ändrad fallback).

## Changelog

- **v1.1.0 (2026-06-09)** — EventIT-sektionen spec:ad. Plans: `trial`/`standard`/`club`. Addons: `qr_checkin`, `attendee_export`, `email_invites`, `branded_page`, `seat_map`, `sponsor_control`, `payments`. Plan-implicit fallback: `association`→`club` (5 addons), `private`→`standard` (3 addons), okänd→`trial`. Klarställt att boolean-keys är per-app och inte får läcka mellan appar.
- **v1.0.0 (2026-06-07)** — Initial release. Tipspromenad-keys frusna.
