# Tipspromenad — status

Per-app statusdokument. Här bor det som rör **detta** Lovable-projekt och
inte de andra (Hub, EventIT). Delade kontrakt: se `docs/contracts/README.md`.

## Aktuellt fokus

- H.2.13 / E3 (se `docs/migration/H.2.13.E3-plan.md`).
- StationPicker-fix enligt `.lovable/plan.md` (väntar verifiering).
- Docs-arkitektur etablerad (alt C hybrid) — kontrakt flyttade till
  `kickinit-platform-docs`, push via edge-funktionen `docs-sync`.
- **Arkitektur uppdaterad: instance-per-app** (ADR-0002 i docs-repot).
  Hub har egen Lovable Cloud-instans; Tipspromenad fortsätter med sin
  egen DB och kommer i en framtida H.x-tranche att läsa org +
  entitlements från Hubs `org-info`-endpoint istället för lokala
  `organizations`/`user_roles`.
- **`org-info` live och verifierad hos Hub** — kontrakt fryst på v1.0.0.
- **Klient implementerad** bakom flagga `HUB_ORG_INFO_ENABLED` +
  `HUB_ENTITLEMENTS_AUTHORITATIVE` (shadow-läge default).
- **SSO-claim-verifiering** blockerad på Hub-sidan (HS256 vs JWKS) —
  lösning: HTTP-introspection mot Tipspromenads `/auth/v1/user`.
  Tipspromenad-sidan färdig; väntar Hub-deploy.
- **Migration 0001 — Hub org seed pushed** ([`b51817c`](https://github.com/Kicken66/kickinit-platform-docs/commit/b51817c2ca57be6a512c7e6370ff584140911719))
  Ericsson, Fixhult AIK, Super Event.
- **Migration 0001b — Hedareds slug-flip pushed + applicerad** (2026-06-07)
  ([`c1798ee`](https://github.com/Kicken66/kickinit-platform-docs/commit/c1798ee41511c81b335ab8a14ae6818e056f57c2)).
  Tipspromenad döpte om sin lokala slug till `hbk`.
- **Cross-app-smoke 2026-06-07**: SSO-claim + `/org-info` OK för Ericsson
  (slug-parity bekräftad). Hub returnerar `entitlements: {}` — saknar
  plan-implicit logik. Hbk-smoke skippad.
- **Migration 0001c — Hub entitlement seed pushed** (2026-06-07)
  - Commit: [`37ee35a`](https://github.com/Kicken66/kickinit-platform-docs/commit/37ee35ae0672419895cf3e5b7a65e7f9b6cf1f0b) `migration/0001c-hub-entitlement-seed.md`
  - Nyckellista: [`88c1dda`](https://github.com/Kicken66/kickinit-platform-docs/commit/88c1dda1160c4c59e6ee8c6f0b63afd4a72dc422) `contracts/v1/entitlement_keys.md` v1.0.0
  - Beslut: **inga rader seedas** i Hubs entitlements-tabell. Istället
    plan-implicit logik i Hubs `org-info` — `association` (club) får
    `results_email + member_lookup + paper_print + quiz_other_modes`
    gratis (speglar `useOrgEntitlements.ts`). `private` (Super Event)
    får tom lista tills `apply-purchase` körs. `sponsor_control` är
    alltid explicit; ingen av de fyra orgs har det idag.
  - Lokala `org_entitlements`: alla fyra orgs som finns i Hub har **0
    rader** lokalt. Den enda existerande raden (`sponsor_control` på
    `tester13s-tipspromenad-484d55`) tillhör test-org som inte seedats.
  - **Bollen hos Hub** att deploya uppdaterad `org-info`. Därefter kör
    vi cross-app-smoke igen — förväntat: shadow-diffar = 0 för alla
    fyra orgs.

## Senaste sign-offs

| Workstream | Sign-off |
|---|---|
| H.2.13 | `docs/migration/H.2.13-signoff.md` |
| H.2.12 | `docs/migration/H.2.12-signoff.md` |
| H.2.11 | `docs/migration/H.2.11-signoff.md` |
| H.2.10 | `docs/migration/H.2.10-signoff.md` |
| H.3 cutover | `docs/migration/H.3-cutover-checklist.md` |

## Externa beroenden

- **Launchpad** (kickinit.se) — anropar `provision-private-org`,
  `apply-purchase`, `sso-from-launchpad`, pollar `onboarding-status`.
- **CRM Suite** — äger Stripe. Slug är delad nyckel; se
  `contracts/v1/stripe-checkout-integration.md` i docs-repot.

## Backlog

Se `docs/backlog/feature-parity.md`.
