# Hub – status

## Nuläge
Hub-projektet skapat. Egen Lovable Cloud-instans provisionerad. Auth (email/password + Google) på plats. `user_roles` + `has_role` + `docs_sync_log` migrerade. Edge-funktionen `docs-sync` deployad. Admin-UI på `/super-admin/docs-sync`. Inga domän-tabeller ännu — `hub.*`-schemat byggs i nästa steg.

## Pågående arbete
Verifiering av docs-sync (manuell push från preview).

## Backlog
- `hub.*`-schema: organizations, members, subscriptions, entitlements, invoices.
- Edge-funktioner: `provision-org`, `apply-purchase`, `org-info`.
- JWKS-endpoint för Hub-utfärdade JWT (SSO).
- UI: "Mitt KickinIT", organisationsväljare, fakturor, medlemshantering, "Mina produkter".
- Launchpad pekar om provisioning till Hub.

## Beroenden mot andra projekt
- **Tipspromenad** och **EventIT** verifierar Hub-utfärdade JWT via JWKS, hämtar org/entitlements via `org-info`.
- **Launchpad/CRM Suite** anropar `apply-purchase` (webhook från Stripe).
- **kickinit-platform-docs** (GitHub) är single source of truth för kontrakt/ADR — denna fil pushas dit som `status/hub.md`.
