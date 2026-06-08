# 0005 — Hub org-info: tre saknade entitlement-nycklar

## Datum
2026-06-08

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

## När Hub bekräftat
- Kör ny shadow-diff i Tipspromenad-preview.
- Förväntat: diff = 0 för samtliga seedade orgs.
- Flippar `kickinit.useHubEntitlementsAuthoritative` → default `true` / permanent env.
