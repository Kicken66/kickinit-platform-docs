# KickinIT × Tipspromenad — Kontrakt v1

Frusna API- och dataformatkontrakt mellan **Launchpad** (kickinit.se, webshop)
och **Tipspromenad-appen** (app.kickinit.se, admin & spel).

## Varför finns dessa?

Apparna körs på olika tech-stackar och kan inte dela kod. Istället
disciplinerar vi integrationen via versionerade kontrakt. **Inget i v1
får brytas** — om ett kontrakt måste ändras bakåtinkompatibelt skapas
v2 sida vid sida, och v1 lever vidare tills båda apparna har migrerat.

## Innehåll

| Fil | Syfte |
|---|---|
| `provision-private-org.md` | Launchpad → Tipspromenad: skapa privatkund-org efter köp |
| `apply-purchase.md` | Launchpad → Tipspromenad: in-app-tillköp (rundor, dagar, entitlements) |
| `inapp-products.md` | Tipspromenad → Launchpad: hämta katalog för in-app-shop |
| `sso-from-launchpad.md` | Launchpad → Tipspromenad: "Öppna admin"-flöde via magic link |
| `kickinit-catalog-v1.schema.json` | JSON Schema för produktkatalogen |
| `CHANGELOG.md` | Per-kontrakt ändringslogg |

## Versionsdisciplin

- Varje kontraktsfil börjar med `**Version: 1.x.y**`.
- Patch (1.0.x): förtydliganden, ingen kodförändring krävs.
- Minor (1.x.0): nya **valfria** fält. Bakåtkompatibelt.
- Major (2.0.0): **nytt endpoint**. v1 lever parallellt tills båda apparna migrerat.

Vid varje ändring: uppdatera filen, CHANGELOG, och bumpa versionen.
