# Migration 0002 — SSO Flöde A live (Tipspromenad)

**Datum:** 2026-06-07  
**App:** Tipspromenad (`wkmtgihiveqwyvozrlid`)  
**Kontrakt:** [`contracts/v1/sso-exchange.md`](../contracts/v1/sso-exchange.md) v1.0.0  
**Status:** ✅ Live på `https://app.kickinit.se`

## Vad som verifierats

End-to-end-test (kicken@kickinit.se, org `ericsson-boras-a5cacc`):

1. Hub redirectar till `https://app.kickinit.se/sso/callback?code=<43-tecken>`
2. Klient invokar `sso-from-hub` (Tipspromenad edge function)
3. `sso-from-hub` HMAC-signerar `{code, app=tipspromenad, iat, exp, nonce}` med `HUB_EXCHANGE_SECRET` → POST mot Hubs `/sso-exchange`
4. Hub returnerar `{hub_jwt, expires_at, hub_user_id, email, org_id, role}`
5. Tipspromenad genererar magic_link mot lokala Auth → redirect → `/dashboard`
6. Hub-JWT lagras i `sessionStorage["kickinit.hubToken.v1"]`

Bevis (från `sso_debug_log`, super_admin-only viewer på `/super-admin/docs-sync`):

| Tid (UTC) | Origin | Fas | Status |
|---|---|---|---|
| 2026-06-07 21:20:12 | `app.kickinit.se` | `callback_received` | code=p1ITMfON… (43) |
| 2026-06-07 21:20:14 | `app.kickinit.se` | `exchange_ok` | email + hub_user_id matchar |

## Hub-konfiguration (verifierad)

- `TIPSPROMENAD_EXCHANGE_SECRET` = samma värde som Tipspromenads `HUB_EXCHANGE_SECRET`
- `TIPSPROMENAD_CALLBACK_URL` = `https://app.kickinit.se/sso/callback` (alla miljöer)

## Diagnostik

Tipspromenad har egen diagnostik-tabell `public.sso_debug_log` (16 cols, super_admin RLS) som tar emot fire-and-forget-rapporter från `/sso/callback` via edge `sso-debug-log`. Loggar fas, origin, referer, user-agent, code_prefix, exchange-status, felkoder, hub_user_id, email, env_hint. Per-miljö-trubbelskytning utan Hub-åtkomst.

## Kvarstående

- **Flöde B (`/sso-claim`)** fortfarande blockerad hos Hub — HS256-introspection-fix kvar (se Tipspromenads status från 2026-06-07 morgon). Flöde A räcker för Hub→Tipspromenad-djuplänk; `useHubOrgInfo` använder Flöde-A-tokenet direkt.
- **Hub `org-info` med plan-implicit entitlements** (migration 0001c) — väntar på Hub-deploy. Då kan cross-app-smoke köras igen och shadow-diffarna ska gå till noll.

## Klient-fix samtidigt (Tipspromenad)

`HubSessionProvider` skippar nu auto-claim när sessionStorage redan innehåller en färsk Hub-JWT (>5 min kvar), så att Flöde-A-tokenet inte skrivs över av ett misslyckat `/sso-claim`.
