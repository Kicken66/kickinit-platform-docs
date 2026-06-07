# Migration 0003 — `org-info` end-to-end via Flöde A (Tipspromenad)

**Datum:** 2026-06-07  
**App:** Tipspromenad (`wkmtgihiveqwyvozrlid`)  
**Kontrakt:** [`contracts/v1/org-info.md`](../contracts/v1/org-info.md) v1.0.0  
**Status:** ✅ Live på `https://app.kickinit.se`

## Vad som verifierats

End-to-end-kedja från Hub → Tipspromenad → Hub `org-info`:

1. **Flöde A**: Hub redirectar till `https://app.kickinit.se/sso/callback?code=<43-tecken>`
2. `sso-from-hub` (Tipspromenad edge function) HMAC-signerar och växlar mot Hubs `/sso-exchange`
3. Hub returnerar `{hub_jwt, expires_at, hub_user_id, email, org_id, role}`
4. Tipspromenad genererar magic_link → `/auth/callback` → session → `/dashboard`
5. **Hub-JWT lagras i `sessionStorage["kickinit.hubToken.v1"]`**
6. **`useHubOrgInfo` använder Flöde-A-tokenet** direkt — ingen `/sso-claim` krävs
7. **`/org-info` (Hub edge function) validerar ES256 Hub-JWT via JWKS** och returnerar fullt svar

Verifierat på `kicken@kickinit.se`, org `ericsson-boras-a5cacc`:

| Fält | Värde |
|---|---|
| `user.id` | `c1ca2a8a-…` |
| `user.email` | `kicken@kickinit.se` |
| `org.slug` | `ericsson-boras-a5cacc` |
| `org.name` | `Ericsson` |
| `org.type` | `association` |
| `role` | `super_admin` |
| `entitlements.tipspromenad.plan` | `club` |
| `entitlements.tipspromenad.addons` | `[]` |
| `entitlements.results_email` | `true` (plan-implicit) |
| `entitlements.member_lookup` | `true` (plan-implicit) |
| `entitlements.paper_print` | `true` (plan-implicit) |
| `entitlements.quiz_other_modes` | `true` (plan-implicit) |
| `entitlements.sponsor_control` | `false` (explicit, matchar lokal) |

## Jämförelse lokal vs Hub

| Nyckel | Lokal | Hub | Diff |
|---|---|---|---|
| `results_email` | `true` | `true` | ✅ match |
| `member_lookup` | `true` | `true` | ✅ match |
| `paper_print` | `true` | `true` | ✅ match |
| `quiz_other_modes` | `true` | `true` | ✅ match |
| `sponsor_control` | `false` | `false` | ✅ match |

**Samtliga entitlement-nycklar matchar.** Tidigare mismatch (`sponsor_control: false vs true`) är jämkad sedan Hub seedade om med `sponsor_control = false`.

## Tekniska detaljer

- **Token-TTL vid verifiering:** ~3275 s (~54 min)
- **JWKS-verifiering:** Hub `/jwks.json` (ES256, kid `e683a410-…`) — validerar Hub-JWT från Flöde A utan problem
- **Slug-parity:** `ericsson-boras-a5cacc` == lokal `currentOrg.slug` ✅
- **Plan-implicit logik:** Hub `org-info` returnerar `association`-org-typen med `results_email + member_lookup + paper_print + quiz_other_modes` utan explicita rader i `hub_entitlements` (se migration 0001c)

## Kvarstående

- **Flöde B (`/sso-claim`)** fortsatt blockerad hos Hub (HS256-introspection-fix ej deployad). Påverkar ej Flöde A eftersom `HubSessionProvider` nu skippar auto-claim när en färsk Flöde-A-token redan finns.
- **Cross-app-smoke för övriga orgs** (Fixhult, Super Event, hbk) — ej körda än; förväntas noll diffar när Hub `org-info` är deployad med plan-implicit logik.

## Klient-fix samtidigt

`HubSessionProvider` auto-claim-logik uppdaterad: om `sessionStorage` redan innehåller en giltig Hub-JWT (>5 min kvar) vid första render efter inloggning, sätts `lastClaimKeyRef` och token används direkt utan att skriva över med ett misslyckat `/sso-claim`.
