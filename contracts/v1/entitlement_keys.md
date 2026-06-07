# Entitlement keys — v1

Status: Accepted (v1.0.0)
Datum: 2026-06-07
Berör: Hub (master för entitlements cross-app), Tipspromenad, EventIT
Refererar: `contracts/v1/org-info.md` v1.0.0 (fältet `entitlements.<app>.addons[]`)

## Syfte

Definierar den **gemensamma nyckeluppsättningen** för per-app addons som
Hub returnerar i `org-info.entitlements.<app>.addons[]`. Apparna mappar
dessa nycklar till sina lokala feature-toggles. Nya nycklar läggs till
här först, sedan i respektive apps mapping-lager.

Nyckelregel: `snake_case`, stabila för all framtid, aldrig återanvänd.

## Tipspromenad (`tipspromenad`)

| Key              | Beskrivning                                                                 | Lokal feature-key (TP) | Plan-implicit för?           |
|------------------|-----------------------------------------------------------------------------|------------------------|------------------------------|
| `results_email`  | Skicka resultat-mail till deltagare efter rundan.                           | `results_email`        | `club`, `standard` (alltid)  |
| `member_lookup`  | Slå upp deltagare mot organisationens medlemsregister.                      | `member_lookup`        | `club`, `standard` (alltid)  |
| `paper_print`    | Skriva ut och rätta papperslappar (PDF + manuell inmatning).                | `paper_print`          | `club`, `standard` (alltid)  |
| `sponsor_control`| Egna sponsorer + sponsor-portal (annars plattformssponsorer).               | `sponsor_control`      | — (separat tillval alltid)   |
| `quiz_other_modes` | Permanenta slingor/serier (`loop`, `series`, `trail`) på privat-plan.    | `quiz_other_modes` (i `organization_subscriptions`, inte `org_entitlements`) | `club`, `standard` (alltid) |

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

TBD — fylls i när EventIT-kontraktet skrivs.

## Lägga till en ny nyckel

1. PR till denna fil med ny rad i tabellen (key + beskrivning + lokal mappning).
2. Hubs `org-info`-handler uppdateras att returnera nyckeln när tillämpligt.
3. Tipspromenad uppdaterar `src/integrations/hub/entitlementMapping.ts`.
4. Bump i `contracts/v1/org-info.md` om payload-formen ändras (annars inte).
