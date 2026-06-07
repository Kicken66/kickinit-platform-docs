# Migration 0001c — Hub entitlement seed (Tipspromenad-orgs)

Status: Accepted
Datum: 2026-06-07
Berör: Hub (master för entitlements), Tipspromenad (källa)
Refererar: ADR-0002, migration 0001, 0001b, `contracts/v1/entitlement_keys.md` v1.0.0

## Bakgrund

Efter 0001/0001b finns fyra Tipspromenad-orgs i Hub:

| Hub-slug                  | Namn               | TP-plan                | Hub-type     |
|---------------------------|--------------------|------------------------|--------------|
| `ericsson-boras-a5cacc`   | Ericsson 💖 Borås  | `club`                 | association  |
| `fixhult-aik-c779e1`      | Fixhult AIK        | `club`                 | association  |
| `super-event-37f617`      | Super Event        | `private_tipspromenad` | private      |
| `hbk`                     | Hedareds Bollklubb | `club`                 | association  |

Cross-app-smoke 2026-06-07 visar `entitlements.tipspromenad = {}` för
Ericsson, dvs Hub returnerar **inga** addons trots att club-plan
implicit har `results_email`, `member_lookup`, `paper_print`,
`quiz_other_modes` aktiva i Tipspromenad.

## Källdata (Tipspromenad — snapshot 2026-06-07)

Lokal `org_entitlements` för de fyra orgs som finns i Hub:

| Org                       | Rader i `org_entitlements` |
|---------------------------|----------------------------|
| Ericsson 💖 Borås         | inga                       |
| Fixhult AIK               | inga                       |
| Super Event               | inga                       |
| Hedareds Bollklubb (hbk)  | inga                       |

Övriga lokala entitlements (tester13s: `sponsor_control`) tillhör
test-orgs som **inte** seedats till Hub — ignoreras.

## Beslut

Hub seedar **inga** addon-rader i denna migration. Istället implementeras
**plan-implicit logik** i Hubs `org-info`-handler enligt
`contracts/v1/entitlement_keys.md`:

```
if app == 'tipspromenad' and org.type in ('association',):
    addons += ['results_email', 'member_lookup', 'paper_print', 'quiz_other_modes']
addons += <explicit rows from hub.entitlements where org_id=... and app='tipspromenad'>
addons = sorted(set(addons))
```

Motivering:

1. Speglar Tipspromenads egna kod (`useOrgEntitlements.ts`): club/standard
   får legacy-addons gratis utan rader.
2. Undviker att seeda rader som inte representerar köp — alla "köpta"
   addons kommer från `apply-purchase` och får då explicit rad i Hub.
3. Idempotent och rättvisande: om en club-org senare migreras till
   private behåller plan-styrningen sin sanning utan att vi måste
   städa rader.

Super Event (`private`) får inga addons i Hub eftersom inga köp ännu
gjorts via `apply-purchase` mot Hub. Stripe-koppling saknas (per 0001).
Backfillas via `apply-purchase` när kunden förnyar.

`sponsor_control` är inte plan-implicit för någon plan — alltid explicit
grant. Ingen av de fyra orgs har det idag.

## SQL — körs i Hubs databas (av Hub-agenten/super-admin)

**Inga `INSERT`/`UPDATE`-statements behövs.** Hela implementationen ligger
i `org-info`-handlern. Hubs entitlements-tabell förblir tom för dessa
fyra orgs efter migrationen.

### Verifiering efter deploy av uppdaterad `org-info`

Från Tipspromenad super-admin (`/super-admin/docs-sync` → debug-panel):

| Org-slug                | Förväntad `entitlements.tipspromenad.addons`                                  |
|-------------------------|-------------------------------------------------------------------------------|
| `ericsson-boras-a5cacc` | `["member_lookup","paper_print","quiz_other_modes","results_email"]`         |
| `fixhult-aik-c779e1`    | `["member_lookup","paper_print","quiz_other_modes","results_email"]`         |
| `hbk`                   | `["member_lookup","paper_print","quiz_other_modes","results_email"]`         |
| `super-event-37f617`    | `[]`                                                                          |

Diff-tabellen i Tipspromenads shadow-läge ska efter detta visa
**0 mismatchar** för club-orgs (alla tre legacy-addons matchar `true`).
`sponsor_control` ska fortsatt visa `false/false` (saknas båda sidor).

## Nästa steg

1. Hub deployar uppdaterad `org-info` med plan-implicit-regeln.
2. Tipspromenad kör cross-app-smoke och uppdaterar
   `docs/status/tipspromenad.md` med utfall.
3. Vid 0 diffs i 7 dagar → flippa `kickinit.useHubEntitlementsAuthoritative`
   till default-on i Tipspromenad.
4. Framtida `apply-purchase` för Super Event → explicit rad i Hub,
   speglas tillbaka till Tipspromenad via shadow-läget.
