# Migration 0001b — Hedareds slug-flip (Tipspromenad → `hbk`)

**Owner:** Tipspromenad
**Depends on:** `migration/0001-hub-org-seed.md` (kör efter)
**Status:** READY — kör efter att Hub bekräftat 0001 applicerad
**Beslutspunkt:** ADR-0002 + Hub-feedback 2026-06-07, variant (b) vald

## Bakgrund

Hub har sedan tidigare en org med slug `hbk` (Hedareds Bollklubb,
id `07514ade-…`, 1 medlem, 2 entitlements). När Hub seedar resterande
orgs från Tipspromenad enligt 0001 kan **inte** Hedareds tas med — det
finns ingen sätt att FK-koppla utan att antingen
(a) byta Hubs befintliga `hbk`-id (bryter FK i Hub) eller
(b) anpassa Tipspromenads slug till `hbk` (kandidat, denna migration) eller
(c) bygga en alias-tabell (utvärderat och avfärdat i ADR-0002 — vi vill
inte blanda namnrymder för exakt en org).

Variant **(b)** vald: Tipspromenad döper om sin lokala slug
`hedareds-bollklubb-000000` → `hbk`. Detta bryter Tipspromenads
`<name>-<6char-id>`-konvention för exakt en historisk org, men är
den lägsta totala risken: ingen FK rörs i någondera DB, ingen
extra mapping-tabell krävs, och SSO-claim kan matcha org direkt
på slug utan special-casing.

## Påverkan

- **Tipspromenad DB:** 1 rad uppdateras i `public.quiz_organizations`.
  Alla FK pekar på `id` (uuid) — ingen kaskad-effekt.
- **Tipspromenad klient:** inga hårdkodade referenser till
  `hedareds-bollklubb-000000` (verifierat med `rg` 2026-06-07).
  Tenant-resolve sker via `id`/`domain`, inte slug.
- **Hub:** ingen ändring krävs. Efter denna migration matchar Hubs
  `hbk` Tipspromenads `hbk` → `org-info` och `sso-claim` kan lösa
  org direkt på slug.
- **Launchpad / Stripe:** slug är delad nyckel i
  `stripe-checkout-integration.md`. Hedareds har dock inga
  Launchpad-bundlade köp (kvar i Tipspromenads gamla flöde) →
  ingen risk.

## SQL

Kör som **en** migration i Tipspromenads Lovable Cloud (kommer som
separat `supabase--migration`-call efter att Hub bekräftat 0001):

```sql
-- 0001b-hedareds-slug-flip.sql
-- Tipspromenad-sidan av Hedareds-konsolidering.
-- Kör EFTER att Hub applicerat migration/0001.

BEGIN;

-- Sanity: org finns och slug är som väntat
DO $$
DECLARE
  v_count int;
BEGIN
  SELECT count(*) INTO v_count
  FROM public.quiz_organizations
  WHERE id = '00000000-0000-0000-0000-000000000bb1'
    AND slug = 'hedareds-bollklubb-000000';
  IF v_count <> 1 THEN
    RAISE EXCEPTION 'Hedareds-org saknas eller har oväntad slug (count=%)', v_count;
  END IF;
END $$;

-- Säkerställ att 'hbk' inte redan är upptagen lokalt
DO $$
DECLARE
  v_count int;
BEGIN
  SELECT count(*) INTO v_count
  FROM public.quiz_organizations
  WHERE slug = 'hbk';
  IF v_count <> 0 THEN
    RAISE EXCEPTION 'slug=hbk redan upptagen lokalt (count=%)', v_count;
  END IF;
END $$;

UPDATE public.quiz_organizations
SET slug = 'hbk',
    updated_at = now()
WHERE id = '00000000-0000-0000-0000-000000000bb1';

COMMIT;
```

## Verifiering

```sql
-- Förväntat: 1 rad, slug='hbk'
SELECT id, slug, name FROM public.quiz_organizations
WHERE id = '00000000-0000-0000-0000-000000000bb1';

-- Förväntat: 0 rader (gamla sluggen borta)
SELECT count(*) FROM public.quiz_organizations
WHERE slug = 'hedareds-bollklubb-000000';
```

Post-deploy smoke (manuell):
1. Logga in som Hedareds-admin i Tipspromenad → `currentOrg.slug` ska
   visa `hbk` i debug-panelen på `/super-admin/docs-sync`.
2. Med `kickinit.useHubOrgInfo=true` (när Hubs SSO-claim är fixad,
   se `docs/status/tipspromenad.md`) → `org-info`-svaret ska peka på
   Hubs `hbk` med samma medlems-roll.

## Rollback

```sql
BEGIN;
UPDATE public.quiz_organizations
SET slug = 'hedareds-bollklubb-000000',
    updated_at = now()
WHERE id = '00000000-0000-0000-0000-000000000bb1'
  AND slug = 'hbk';
COMMIT;
```

Säker så länge Hub inte hunnit referera Tipspromenads `hbk` i något
nytt cross-app-flöde (apply-purchase, etc.) — vilket vid 0001b-tid
ännu inte är aktivt.

## Checklista

- [ ] Hub har applicerat och bekräftat 0001 (commit `b51817c`)
- [ ] Tipspromenad kör SQL ovan via `supabase--migration`
- [ ] Verifierings-queries returnerar förväntat resultat
- [ ] Uppdatera `docs/status/tipspromenad.md` med migrations-datum
- [ ] Ping Hub i `docs/status/hub.md`: "Tipspromenad slug='hbk' aktiv,
      cross-app-resolution klar att aktivera"
