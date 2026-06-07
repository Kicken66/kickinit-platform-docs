Migration 0001 — Seed Tipspromenads orgs in Hub
Status: Accepted (revision 2)
Datum: 2026-06-07
Berör: Hub (master), Tipspromenad (källa)
Refererar: ADR-0002 § "Migration från nuläget"

Bakgrund
ADR-0002 slår fast att Hub är master för organisationer. Tipspromenad har skapat orgs lokalt sedan långt innan Hub blev master — de finns alltså i tipspromenad.public.quiz_organizations men saknas i hub.hub.organizations (utom Hedareds BK som finns med slug hbk i Hub).

Konsekvens: sso-claim?org=<lokal-slug> returnerar org_not_member för alla Tipspromenad-orgs utom hbk. Den ursprungliga planen löser detta utan alias-tabell genom en engångskopiering av Tipspromenads orgs till Hub, med samma id och samma slug.

Beslut (efter samråd med Hub-agenten 2026-06-07)
Schema-mapping Tipspromenad → Hub:
club → association
private_tipspromenad → private
standard → association
Hedareds slug-kollision: skjuts upp till separat migration 0001b. Seedas inte här. Lösning där: Tipspromenad byter sin lokala slug till hbk (variant b) — icke-invasivt för Hub, bryter Tipspromenads <namn>-<6char-id>-konvention för exakt en historisk org.
Super Event: seedas i denna migration trots att Stripe-koppling saknas. Aktiv kund i Tipspromenad sedan 2026-06-05. Entitlements backfillas separat när apply-purchase är live.
Test-orgs och platform-org: seedas inte. Lämnas i Tipspromenad fram till H.x-cutover.
Källdata (Tipspromenad — snapshot 2026-06-07)
id	slug	name	plan (TP)	type (Hub)	Seedas i 0001?
a5cacc38-6ca0-43bb-9b30-fdfd1a0ae5ef	ericsson-boras-a5cacc	Ericsson 💖 Borås	club	association	JA
c779e1c4-8421-4e0c-8561-78fa9fab09a5	fixhult-aik-c779e1	Fixhult AIK	club	association	JA
51a48242-55b3-4483-b422-c736a3fa3a42	super-event-37f617	Super Event	private_tipspromenad	private	JA
00000000-0000-0000-0000-000000000bb1	hedareds-bollklubb-000000	Hedareds Bollklubb	club	—	Nej (→ 0001b)
b7dcf3ff-… Test5, 089d0284-… Test10, b5de24cc-… Test11, c40eb9c0-… Test12, 7295b618-… tester13s	div	div	private_tipspromenad	—	Nej (test)
00000000-…-c1c1	kickinit-platform	KickinIT	standard	—	Nej (platform/intern)
SQL — körs i Hubs databas (av Hub-agenten/super-admin)
-- Hub-schema (per Hub-agent 2026-06-07):
--   hub.organizations (id uuid PK, slug text unique, name text, type hub.org_type, created_at timestamptz)
--   hub.org_type enum: 'association' | 'company' | 'private'

INSERT INTO hub.organizations (id, slug, name, type)
VALUES
  ('a5cacc38-6ca0-43bb-9b30-fdfd1a0ae5ef', 'ericsson-boras-a5cacc', 'Ericsson 💖 Borås', 'association'),
  ('c779e1c4-8421-4e0c-8561-78fa9fab09a5', 'fixhult-aik-c779e1',    'Fixhult AIK',       'association'),
  ('51a48242-55b3-4483-b422-c736a3fa3a42', 'super-event-37f617',    'Super Event',       'private')
ON CONFLICT (id) DO UPDATE
  SET slug = EXCLUDED.slug,
      name = EXCLUDED.name,
      type = EXCLUDED.type;
Inga entitlements seedas. hub.org_members fylls i migration/0002-hub-membership-seed.md.

Migration 0001b — Hedareds (separat dokument, kommer)
Plan:

Tipspromenad: UPDATE public.quiz_organizations SET slug = 'hbk' WHERE id = '00000000-0000-0000-0000-000000000bb1'; (id bevaras, lokala FK:er orörda).
Hub: ingen ändring. hbk finns redan med 1 medlem + 2 entitlements.
Verifiering: sso-claim?org=hbk från Tipspromenad ger Hub-JWT mot Hubs hbk-org-id (07514ade-…). Eftersom Tipspromenad inte längre delar id med Hub för denna org, måste Tipspromenads kod använda Hub-org-id (claims.org_id) som primär identitet vid cross-app-anrop, inte sitt lokala 00000000-…-0bb1. Detta är ändå mönstret per ADR-0002.
Verifiering efter seed 0001
Från Tipspromenad super-admin (/super-admin/docs-sync):

Växla currentOrg till Fixhult AIK → claimed org ska bli fixhult-aik-c779e1, Hub-svar samma slug, org.id = c779e1c4-….
Samma för Ericsson och Super Event.
Entitlement-diff: alla rader visar "saknas på Hub-sidan" tills apply-purchase körts. OK i shadow-läget.
Nästa steg efter seed
Shadow-loggning (hub_entitlement_shadow_log) på alla 3 seedade orgs + Hedareds i 7 dagar.
0 diffs över 7 dagar → flippa kickinit.useHubEntitlementsAuthoritative till default-on.
Skriv migration/0001b-hedareds-slug-fix.md och migration/0002-hub-membership-seed.md.
Hard-block: org-skapande lokalt i Tipspromenad markeras deprecated.