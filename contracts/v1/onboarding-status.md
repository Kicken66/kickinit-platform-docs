# Contract: onboarding-status

**Version: 1.0.0** · **Status: DRAFT**
**Direction:** Launchpad → Tipspromenad
**Endpoint:** `GET {TIPSPROMENAD_FUNCTIONS_URL}/onboarding-status?organization_id=<uuid>`

Server-to-server read-endpoint som Launchpad pollar för att stänga loopen mellan
köp (`/tack/$orderId`) och första aktiverade tipspromenad. Read-only.

## Auth

Header: `x-provision-key: <PRIVATE_ORG_PROVISION_KEY>`
Samma shared secret som `provision-private-org`.

## Request

Query params:

| Param | Krav | Notering |
|---|---|---|
| `organization_id` | required | UUID på orgen Launchpad provisionerat. |

## Response 200

```json
{
  "organization_id": "uuid",
  "name": "Kalles Tipspromenad",
  "plan": "private_tipspromenad",
  "onboarded_at": "2026-06-04T10:11:12Z",
  "created_at": "2026-06-04T09:00:00Z",
  "has_round": true,
  "has_published_round": false,
  "events": [
    { "event": "onboarding_started",        "step": 1, "created_at": "...", "metadata": {} },
    { "event": "onboarding_step_completed", "step": 1, "created_at": "...", "metadata": { "round_id": "..." } },
    { "event": "onboarding_step_completed", "step": 2, "created_at": "...", "metadata": { "round_id": "..." } },
    { "event": "onboarding_finished",       "step": 3, "created_at": "...", "metadata": {} }
  ]
}
```

| Fält | Notering |
|---|---|
| `onboarded_at` | `null` tills användaren slutfört eller medvetet hoppat över wizarden. |
| `has_round` | `true` så snart minst en runda är skapad (även `draft`). |
| `has_published_round` | `true` när minst en runda är `active` eller `completed`. |
| `events` | Sorterade stigande på `created_at`. |

### Event-typer

- `onboarding_started`
- `onboarding_step_completed` (step 1|2|3)
- `onboarding_skipped` (step 1|2|3)
- `onboarding_finished`

## Error responses

| Status | `code` | Betydelse |
|---|---|---|
| 400 | `invalid_payload` | `organization_id` saknas. |
| 401 | `invalid_key` | Saknad/felaktig `x-provision-key`. |
| 404 | `unknown_org` | Orgen finns inte. |
| 405 | `method_not_allowed` | Endast GET. |
| 500 | `internal_error` | |

## Pollnings-rekommendation för Launchpad

Polla högst var 30:e sekund första 10 min efter `provision-private-org`-svar,
sedan glesare. Sluta polla så snart `has_published_round=true` eller efter 24h.
