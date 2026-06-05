# Contract Changelog

## 1.2.0 — 2026-06-04

Lagt till:

- `onboarding-status.md` (v1.0.0) — read-endpoint som Launchpad pollar för att
  följa när en provisionerad org genomför sin first-run-wizard och publicerar
  sin första tipspromenad. Server-to-server, gated på `PRIVATE_ORG_PROVISION_KEY`.



## 1.1.0 — 2026-06-03

Lagt till:

- `stripe-checkout-integration.md` — delat kontrakt som beskriver hur
  CRM Suite (master / Stripe-ägare), Tipspromenad (in-app checkout) och
  Launchpad/Shop (anonyma köp) delar upp Stripe-integrationen. Slug är
  enda nyckeln som korsar systemgränser; `price.lookup_key = slug`.
  Konsumenter lagrar aldrig Stripes interna ID:n.

Ingen förändring i befintliga v1-kontrakt — de förblir frusna.

## 1.0.0 — 2026-05-28

Frozen v1. Innehåller:

- `provision-private-org` — privatkund-org-provisionering, idempotent.
- `apply-purchase` — in-app-tillköp, idempotent.
- `inapp-products` — katalog-sync (Launchpad → Tipspromenad pull).
- `sso-from-launchpad` — magic-link-bridge för "Öppna admin"-flöde.
- `kickinit-catalog-v1.schema.json` — produktkatalog-validering.
- `kickinit-tokens.json` — designtokens (färg, typografi, radii, shadows).

Bakgrund: skapad inför utflytt av tipspromenad-modulen från memberhub
till eget Lovable-projekt med egen Supabase. Se `.lovable/plan.md`.
