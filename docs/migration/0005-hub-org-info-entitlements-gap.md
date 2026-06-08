# 0005 — Hub org-info: tre saknade entitlement-nycklar

**Status:** ✅ RESOLVED (2026-06-08)

## Datum
2026-06-08 (öppnad + stängd samma dag)


## Bakgrund
Tipspromenads shadow-diff mellan lokal `useOrgEntitlements` och Hubs `org-info` är nere på **4 av 5 nycklar** efter fix av plan-implicit logik (se `0001c`). Återstående diff:

| Nyckel | Lokal (alltid true för association) | Hub org-info (saknas) |
|--------|-------------------------------------|----------------------|
| `results_email` | `true` | — |
| `member_lookup` | `true` | — |
| `paper_print` | `true` | — |

`sponsor_control` är redan korrekt (explicit i båda ändar) ✅

## Konsekvens
Så länge nycklarna saknas i `org-info` kan Tipspromenad **inte** slå på `HUB_AS_SOURCE_OF_TRUTH` utan att de tre funktionerna blir `false` för alla orgs — en regression.

## Önskemål till Hub
1. Uppdatera `contracts/v1/entitlement_keys.md` (v1.1.0) med:
   - `results_email`
   - `member_lookup`
   - `paper_print`
2. Uppdatera Hubs `org-info`-implementation så att `association`-planer returnerar de tre nycklarna med värde `true` (speglar dagens plan-implicit logik i Tipspromenad).
3. Bekräfta i denna migration när deploy är gjord.

## Resolution (2026-06-08)
- Hub deployade `20ccc7c` + `6bbd714`: top-level boolean-fält för `results_email`, `member_lookup`, `paper_print` (+ `sponsor_control`) i `org-info`-svaret, samt `entitlement_keys.md` v1.1.0.
- Tipspromenad uppdaterade `hubToLocalEntitlements` (additivt — läser top-level booleans när de finns, faller annars tillbaka på `addons[]` + plan-implicit).
- Shadow-diff i preview-konsolen: **0 avvikelser** för samtliga testade orgs.
- `VITE_HUB_ENTITLEMENTS_AUTHORITATIVE="true"` satt i `.env.production`. Hub är nu primär källa; lokal `org_has_entitlement` = fallback.

## Plan-kantfall (stängt 2026-06-08)
- **Tidigare risk:** om Hub skickade `tipspromenad`-appen utan `plan`-fält tolkades `""` inte som privat → plan-implicit kickade in och org fick gratis fullt add-on-paket.
- **Fix:** `hubToLocalEntitlements` använder nu en **allowlist** (`IMPLICIT_PLANS = {association, club}`) istället för en negerad privat-lista. Saknad/okänd plan → inga implicita features (säker default). Explicit booleans och `addons[]` fungerar oförändrat.
- **Verifierat:** `src/integrations/hub/entitlementMapping.test.ts` har två nya cases (saknad plan + okänd plan) som båda förväntar `false` på alla 4 nycklar utan explicit grant. 14/14 mapping-tester gröna.
- **Kontraktsstatus:** Hub bör fortfarande alltid skicka `plan`-fält (kontraktkrav i `org-info.md` v1.1.0), men appen är nu robust mot regression.
