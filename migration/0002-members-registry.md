# Migration 0002 — Members-import (registry + auto-claim)

**Status:** Live i Hub (2026-06-08)
**Scope:** Hub-internt. Tipspromenad/EventIT konsumerar resultatet indirekt via `org-info` (medlemskap → entitlements).

## Mål
Hub är source of truth för "vilka emails *får vara* medlem i en org". När en användare loggar in i Hub och har en `public.profiles`-rad vars email matchar en registry-rad så promotas hen automatiskt till `hub.org_members`.

## Tabell `hub.org_members_registry`

| kolumn      | typ                          | not null | default                |
|-------------|------------------------------|----------|------------------------|
| id          | uuid                         | ✓        | gen_random_uuid()      |
| org_id      | uuid → hub.organizations(id) | ✓        | (ON DELETE CASCADE)    |
| email       | citext                       | ✓        |                        |
| role        | hub.member_role              | ✓        | 'member'               |
| invited_at  | timestamptz                  | ✓        | now()                  |
| invited_by  | uuid (Hub-user)              |          |                        |
| claimed_at  | timestamptz                  |          | NULL tills auto-claim  |
| claimed_by  | uuid (profiles.id)           |          |                        |
| note        | text                         |          |                        |
| created_at  | timestamptz                  | ✓        | now()                  |
| updated_at  | timestamptz                  | ✓        | now()                  |

Unique: `(org_id, email)`. Index: `idx_registry_email`, `idx_registry_org`.

## RLS-policys
- `super_admin_all_registry` — `has_role(auth.uid(), 'super_admin')` (ALL).
- `org_members_read_registry` — medlemmar i samma org får SELECT ("vilka är inbjudna").
- Service-role bypass enligt projektnorm; edge-funktionen `members-registry` använder service-role.

## Auto-claim-funktion
`hub.claim_registry_for_profile(_profile_id uuid, _profile_email citext) RETURNS integer` (SECURITY DEFINER).

För varje registry-rad där `email = _profile_email AND claimed_at IS NULL`:
1. `INSERT INTO hub.org_members(org_id, user_id, role) VALUES (...) ON CONFLICT (org_id, user_id) DO UPDATE SET role = EXCLUDED.role`
2. `UPDATE registry SET claimed_at = now(), claimed_by = _profile_id`

Returnerar antal claimade rader.

## Triggers
1. `trg_profiles_claim_registry` — `AFTER INSERT OR UPDATE OF email ON public.profiles` → kallar funktionen med `NEW.id, NEW.email`. Täcker: ny användare signar upp / användare byter email.
2. `trg_registry_claim_existing` — `AFTER INSERT ON hub.org_members_registry` → kallar funktionen om matchande profil redan finns. Täcker: användaren onboardades innan inbjudan lades in.

Email-jämförelse är case-insensitive (citext på båda sidor).

## Edge function `members-registry`
Super_admin-only CRUD via service-role.

Actions:
- `list_orgs` → `{ orgs: [{ id, slug, name, type }] }`
- `list({orgId})` → `{ entries: [{ id, email, role, invited_at, invited_by, claimed_at, claimed_by, claimed_by_email, note }] }`
- `add({orgId, entries: [{email, role, note?}]})` → `{ added, conflicts, claimed, skipped: [{email, reason}] }`
- `import_csv({orgId, csv})` → `{ parsed, added, conflicts, claimed, skipped }` (kolumner `email,role,note`, header valfri, max 5 000 rader)
- `remove({id})` → `{ ok: true }` (påverkar inte redan claimade `org_members`-rader)

Validering: email-regex, `role ∈ { 'member', 'org_admin' }`, dubletter inom samma payload räknas som conflicts.

## Hub-UI
`/super-admin/members` — org-väljare, register-tabell (Email · Roll · Status · Claimed by · Action), inline "Lägg till", CSV-import med resultatsammanfattning. Länkad från startsidan när användaren är super_admin.

## Seed
`kicken@kickinit.se` som `org_admin` i org `hbk` (`07514ade-6a0d-40bb-b01b-9e2c34d61023`). Eftersom profilen redan fanns vid migrationen claimades raden direkt av `trg_registry_claim_existing` → matchande rad finns i `hub.org_members` och `claimed_at` är satt på registry-raden.

## Verifiering 2026-06-08
- ✓ `hub.org_members_registry` finns, RLS aktivt, triggers `trg_profiles_claim_registry` + `trg_registry_claim_existing` listade på sina tabeller.
- ✓ `members-registry` edge function deployad, `list_orgs` returnerar fyra orgs.
- ✓ `add` av icke-existerande email `noreply+e2e@kickinit.se` → `{added: 1, claimed: 0}` (förväntat: pending tills profil finns).
- ✓ Seed-raden för kicken är synlig som `claimed` i UI:t.
- ✓ Existerande `sso-claim`-flöde påverkas inte — super_admin-overlay i `sso-claim` lämnad orörd enligt designval.

## Inte med (medvetna val)
- Ingen push/pull mot Tipspromenad — registry är manuellt i v1.
- Ingen self-service "Bjud in" i org-admin-UI — endast super_admin.
- Ingen email-notifiering till inbjudna.
- Super_admin-overlay i `sso-claim` (acting_as_org_id) lämnad orörd.
