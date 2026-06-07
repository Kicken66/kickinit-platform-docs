# Migration 0001 — Seed Tipspromenads orgs in Hub

**Status:** Proposed
**Datum:** 2026-06-07
**Berör:** Hub (master), Tipspromenad (källa)
**Refererar:** ADR-0002 § "Migration från nuläget"

## Bakgrund

ADR-0002 slår fast att **Hub är master för organisationer** och att produkt-apparna inte ska duplicera org-data. Tipspromenad har dock skapat sina orgs lokalt sedan långt innan Hub blev master — de finns alltså i `tipspromenad.public.quiz_organizations` men saknas i `hub.public.organizations` (utom `Hedareds BK` som finns med slug `hbk` i Hub).

Konsekvens i nuläget: när Tipspromenad-användare gör `sso-claim?org=<slug>` med sin lokala slug returnerar Hub `org_not_member` (eller faller tillbaka på Hub-default-org om slug saknas helt). Detta blockerar `org-info`-flödet för alla orgs utom `hbk`.

Den ursprungliga planen löser detta **utan alias-tabell** genom att en gång kopiera Tipspromenads orgs till Hub, med samma `id` och samma `slug` så att `?org=<slug>` Just Works framöver. Efter seed:en stängs Tipspromenads org-skapande av lokalt — nya orgs skapas via Hub.

## Mål

1. Varje produktionsrelevant Tipspromenad-org finns i `hub.public.organizations` med samma `id` och `slug`.
2. `Hedareds Bollklubb` slug-byts i Hub från `hbk` till `hedareds-bollklubb-000000` **eller** Tipspromenad byter sin lokala slug till `hbk` (Hub-teamet väljer — se § "Beslutspunkter").
3. Test-orgs och platform-orgen seedas inte (markerade nedan).
4. Tipspromenads lokala org-skapande markeras som deprecated i en uppföljande PR.

## Källdata (Tipspromenad — snapshot 2026-06-07)

| id | slug | name | plan | created | Seedas till Hub? |
|---|---|---|---|---|---|
| `a5cacc38-6ca0-43bb-9b30-fdfd1a0ae5ef` | `ericsson-boras-a5cacc` | Ericsson 💖 Borås | club | 2026-03-30 | **JA** |
| `c779e1c4-8421-4e0c-8561-78fa9fab09a5` | `fixhult-aik-c779e1` | Fixhult AIK | club | 2026-05-07 | **JA** |
| `00000000-0000-0000-0000-000000000bb1` | `hedareds-bollklubb-000000` | Hedareds Bollklubb | club | 2026-05-11 | **SE BESLUT** (kollision med Hub `hbk`) |
| `b7dcf3ff-75e9-4e4f-a757-6e6abe52db01` | `test5-klubben-b8d8f5` | Test5-klubben | private_tipspromenad | 2026-06-02 | Nej (test) |
| `089d0284-2f1d-49db-b1a8-377319befe6a` | `test10-9a7385` | Test10 | private_tipspromenad | 2026-06-04 | Nej (test) |
| `b5de24cc-99b6-4261-8831-5489bf8a1892` | `test11-b2290a` | Test11 | private_tipspromenad | 2026-06-04 | Nej (test) |
| `c40eb9c0-1022-4f10-8cc5-5bb865df7371` | `test12-3c8793` | Test12 | private_tipspromenad | 2026-06-04 | Nej (test) |
| `7295b618-546e-4710-849a-a4a12b1b7def` | `tester13s-tipspromenad-484d55` | tester13s tipspromenad | private_tipspromenad | 2026-06-04 | Nej (test) |
| `00000000-0000-0000-0000-00000000c1c1` | `kickinit-platform` | KickinIT | standard | 2026-06-04 | Nej (platform/intern) |
| `51a48242-55b3-4483-b422-c736a3fa3a42` | `super-event-37f617` | Super Event | private_tipspromenad | 2026-06-05 | **JA** (om aktiv betalkund — Hub-team verifierar mot Stripe) |

## SQL — körs i **Hubs** databas (av Hub-agenten/super-admin)

```sql
-- Förvänta att hub.organizations har: id uuid PK, slug text unique, name text,
-- type text, created_at timestamptz default now()
-- Justera kolumnnamn om Hub-schemat avviker.

INSERT INTO public.organizations (id, slug, name, type)
VALUES
  ('a5cacc38-6ca0-43bb-9b30-fdfd1a0ae5ef', 'ericsson-boras-a5cacc',   'Ericsson 💖 Borås', 'club'),
  ('c779e1c4-8421-4e0c-8561-78fa9fab09a5', 'fixhult-aik-c779e1',      'Fixhult AIK',       'club'),
  ('51a48242-55b3-4483-b422-c736a3fa3a42', 'super-event-37f617',      'Super Event',       'club')
ON CONFLICT (id) DO UPDATE
  SET slug = EXCLUDED.slug,
      name = EXCLUDED.name;
```

**För `Hedareds BK`** — beror på beslut nedan. Föredragen variant (Hub anpassar sig till Tipspromenads slug-konvention):

```sql
UPDATE public.organizations
   SET id   = '00000000-0000-0000-0000-000000000bb1',
       slug = 'hedareds-bollklubb-000000'
 WHERE slug = 'hbk';
```

⚠️ `UPDATE id` kräver att FK:er till Hub-organizations använder `ON UPDATE CASCADE` eller att raden migreras manuellt. Hub-team verifierar innan körning.

## Medlems-mappning (uppföljning, inte del av denna migration)

Seed:en kopierar **bara org-raderna**. `hub.user_organization_members` (eller motsvarande) behöver fyllas separat så att Tipspromenads användare blir medlemmar i rätt Hub-org. Det görs i `migration/0002-hub-membership-seed.md` (skrivs när vi vet exakt vilka users som finns i båda systemen).

## Beslutspunkter (Hub-team)

1. **Hedareds slug-kollision:** byta Hub `hbk` → `hedareds-bollklubb-000000`, eller byta Tipspromenad-slug till `hbk`?
   - Rekommendation: byt Hubs slug. Tipspromenad-konventionen är `<slug>-<6-char-id>`, det är värt att behålla.
2. **Super Event seed:** verifiera mot Stripe/Launchpad att detta är en aktiv betalkund innan seed.
3. **Test-orgs:** lämna i Tipspromenad. När Hub-master-läget aktiveras (H.x-tranche per ADR-0002) ska Tipspromenad sluta acceptera orgs som inte finns i Hub — test-orgs får då raderas eller migreras manuellt.

## Verifiering efter seed

Från Tipspromenad:

```sh
# Använd super_admin Hub-JWT (claim:a först)
curl -X POST "$HUB/sso-claim?org=fixhult-aik-c779e1" \
  -H "Authorization: Bearer $TIPS_JWT" -d '{"app":"tipspromenad"}'
# Förväntat: 200 med hub_jwt vars claims.org_id = c779e1c4-8421-4e0c-8561-78fa9fab09a5

curl -X POST "$HUB/org-info" \
  -H "Authorization: Bearer $HUB_JWT" -d '{}'
# Förväntat: org.slug = "fixhult-aik-c779e1", org.id = c779e1c4-…
```

I `/super-admin/docs-sync`-vyn (Hub-debug-panelen):
- Växla currentOrg till Fixhult AIK → `claimed org` ska bli `fixhult-aik-c779e1`
- Hub-svar ska visa samma slug
- Entitlement-jämförelse: alla rader bör visa "saknas" tills `apply-purchase` körts för orgen (det är OK i shadow-läget)

## Efter seed: nästa steg i Tipspromenad

- Aktivera shadow-loggning (`hub_entitlement_shadow_log`) på alla seedade orgs i 7 dagar.
- 0 diffs över 7 dagar → flippa `kickinit.useHubEntitlementsAuthoritative` till default-on.
- Skriv `migration/0002-hub-membership-seed.md`.
- Inför hard-block i Tipspromenad: org-skapande lokalt deprecated, alla nya orgs via Hub.
