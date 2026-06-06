# KickinIT × Tipspromenad — Kontrakt v1

Frusna API- och dataformatkontrakt mellan **Launchpad** (kickinit.se, webshop),
**Hub** (plattformens master för org + entitlements),
**Tipspromenad-appen** (app.kickinit.se, admin & spel),
**EventIT-appen** och **CRM Suite** (master för produktkatalog och Stripe).

## Varför finns dessa?

Apparna körs på olika tech-stackar och kan inte dela kod. Istället
disciplinerar vi integrationen via versionerade kontrakt. **Inget i v1
får brytas** — om ett kontrakt måste ändras bakåtinkompatibelt skapas
v2 sida vid sida, och v1 lever vidare tills båda apparna har migrerat.

## Deployment-modell

Varje app har en **egen Lovable Cloud-instans** (egen Supabase). Ingen
delad databas. Cross-app-integration sker via Hubs publika endpoints:

- **`/jwks.json`** — Hub publicerar sina JWT-signeringsnycklar; övriga
  appar verifierar SSO-tokens mot denna.
- **`org-info`** edge function — produkt-apparna hämtar org + entitlements
  från Hub. Se `contracts/v1/org-info.md`.
- **`apply-purchase`** edge function — Launchpad/Stripe webhookar Hub
  vid köp; entitlements uppdateras i Hubs databas.

Se `adr/0002-instance-per-app.md` för bakgrund och konsekvenser.

## Innehåll

Kontrakten är av två slag:

- **Per-endpoint-kontrakt** beskriver ett konkret HTTP-anrop mellan två appar.
- **Tvärgående kontrakt** beskriver hur flera appar delar upp ett ansvarsområde
  (t.ex. Stripe) utan att vara ett enskilt endpoint.

### Per-endpoint-kontrakt

| Fil | Syfte |
|---|---|
| `provision-private-org.md` | Launchpad → Tipspromenad: skapa privatkund-org efter köp |
| `apply-purchase.md` | Launchpad → Hub: in-app-tillköp (rundor, dagar, entitlements) |
| `org-info.md` | Tipspromenad/EventIT → Hub: hämta org + entitlements för inloggad user |
| `inapp-products.md` | Tipspromenad → Launchpad: hämta katalog för in-app-shop |
| `sso-from-launchpad.md` | Launchpad → Hub: "Öppna admin"-flöde via magic link |
| `onboarding-status.md` | Launchpad → Tipspromenad: status-poll efter provisionering |

### Tvärgående kontrakt

| Fil | Syfte |
|---|---|
| `stripe-checkout-integration.md` | Hur CRM Suite, Hub och Launchpad delar Stripe-ansvar |

### Scheman & tokens

| Fil | Syfte |
|---|---|
| `kickinit-catalog-v1.schema.json` | JSON Schema för produktkatalogen |
| `kickinit-tokens.json` | Designtokens (färg, typografi, radii, shadows) |
| `CHANGELOG.md` | Per-kontrakt ändringslogg |

## Versionsdisciplin

- Varje kontraktsfil börjar med `**Version: 1.x.y**`.
- Patch (1.0.x): förtydliganden, ingen kodförändring krävs.
- Minor (1.x.0): nya **valfria** fält. Bakåtkompatibelt.
- Major (2.0.0): **nytt endpoint**. v1 lever parallellt tills båda apparna migrerat.

Vid varje ändring: uppdatera filen, CHANGELOG, och bumpa versionen.
