# ADR 0002 — Instance-per-app (Lovable Cloud per projekt)

**Status:** Accepted
**Datum:** 2026-06-06
**Supersedes:** delar av ADR-0001 — specifikt antagandet att Hub, Tipspromenad och EventIT skulle dela en Supabase-instans med ett schema per app.

## Kontext

Ursprunglig plan för KickinIT-plattformen utgick från **en delad Supabase-instans** med tre schemas (`hub`, `tipspromenad`, `eventit`) som de tre Lovable-projekten skulle peka mot.

Det visade sig vara omöjligt att genomföra i Lovable Cloud:

- Varje Lovable-projekt med Cloud aktiverat provisionerar **alltid en ny, dedikerad Supabase-instans**.
- Det går **inte** att manuellt peka om ett Cloud-projekt till en befintlig Supabase-instans från ett annat projekt.
- `SUPABASE_URL` / publishable key i `.env` är auto-genererade och låsta per projekt.

Konsekvensen: schemas-modellen från ADR-0001 är inte genomförbar. Vi behöver en arkitektur som accepterar att varje app har sin egen instans, och som ändå låter Hub fungera som plattformens "sanning" för organisationer, användare och köpta produkter (entitlements).

## Beslut

**Varje app får sin egen Lovable Cloud-instans.** Ingen delad databas, inga cross-database-queries, inga delade schemas.

| App | Domän | Roll |
|---|---|---|
| **Hub** | (TBD) | Master för organisationer, användare, entitlements. Tar emot köp-webhooks från Launchpad/Stripe. Utfärdar SSO. |
| **Tipspromenad** | app.kickinit.se | Produkt-app. Egen domändata (rundor, frågor, deltagare). Verifierar Hub-tokens. |
| **EventIT** | (TBD) | Produkt-app. Egen domändata. Verifierar Hub-tokens. |

Cross-app-integration sker via tre mekanismer som alla bor på Hub:

1. **SSO / JWT** — Hub utfärdar tokens när användaren loggar in. Tipspromenad och EventIT verifierar token mot Hubs publika JWKS-endpoint. Sessionen i produkt-appen skapas utifrån Hub-tokens claims (`sub`, `org_id`, `email`).
2. **Edge function `org-info`** (på Hub) — produkt-apparna anropar denna med Hub-token för att hämta `{ org, entitlements, role }`. Cachas klient-/edge-side ~60 s. Se `contracts/v1/org-info.md`.
3. **Edge function `apply-purchase`** (på Hub) — webhook-mottagare från Launchpad/Stripe. Uppdaterar `entitlements` i Hubs databas. Produkt-apparna ser ändringen vid nästa `org-info`-anrop.

Domändata stannar i respektive app:
- Tipspromenad äger rundor, frågor, deltagare, sponsorer.
- EventIT äger event, biljetter, deltagare för event.
- Hub äger ingen domändata — bara organisationer, användare, roller, entitlements.

## Konsekvenser

**Plus**
- Tydlig isolering. Bugg i en app kan inte ta ner databasen för en annan.
- RLS skrivs per app utan att behöva tänka på de andra.
- Oberoende deploys, migrationer och rollbacks.
- Mindre "blast radius" vid läckta secrets — varje app har t.ex. egen `GITHUB_DOCS_PAT`.

**Minus**
- Latens: `org-info` är ett HTTP-hopp. Mitigeras med kort cache (~60 s) och att klienten håller resultatet i React Query.
- Konsistens: entitlements i produkt-app släpar max ~60 s efter köp. Acceptabelt; webhook → `apply-purchase` är ändå asynkront.
- Varje produkt-app behöver implementera JWKS-verifiering och `org-info`-anrop. En gång per app, sedan delat mönster.
- Tre olika Supabase-projekt att övervaka. Logg/larm sätts per app.

**Migration från nuläget (Tipspromenad)**
- Tipspromenad har idag lokala tabeller `organizations` och `user_roles`. Dessa fortsätter fungera tills Hub är live.
- När Hub har `org-info` + JWKS i produktion: ny H.x-tranche som ersätter lokala läsningar med Hub-anrop. Inga datakopior — Hub blir sanningen, Tipspromenads org-tabeller fryses/läses bara för bakåtkompatibilitet under övergången.
- Detaljplan skrivs när Hub är på plats; inte en del av denna ADR.

## Förbjudet framöver

- Försök att peka Cloud-projekt mot annan apps Supabase-instans.
- Cross-database joins eller foreign keys mellan apparna.
- Att duplicera entitlements i produkt-appens egen databas (utöver kort cache).
