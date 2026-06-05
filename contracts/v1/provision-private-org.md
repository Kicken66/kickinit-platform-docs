# Contract: provision-private-org

**Version: 1.0.0** · **Status: FROZEN**
**Direction:** Launchpad → Tipspromenad
**Endpoint:** `POST {TIPSPROMENAD_FUNCTIONS_URL}/provision-private-org`

Skapar en privatkund-organisation, auth-user och prenumeration efter ett
genomfört köp i Launchpad-webshoppen. **Idempotent** på `external_order_ref`.

## Auth

Header: `x-provision-key: <PRIVATE_ORG_PROVISION_KEY>`
Shared secret konfigurerad i båda sidornas secret-store.

## Request

```json
{
  "external_order_ref": "kickinit-2026-000123",
  "email": "kalle@example.com",
  "full_name": "Kalle Anka",
  "organization_name": "Kalles Tipspromenad",
  "product_key": "private_tipspromenad_year",
  "phone": "+46701234567",
  "metadata": {
    "stripe_session_id": "cs_live_..."
  }
}
```

| Fält | Typ | Krav | Notering |
|---|---|---|---|
| `external_order_ref` | string | required | Globalt unik; idempotensnyckel. Format `kickinit-YYYY-NNNNNN`. |
| `email` | string | required | Lowercase, normaliserad. |
| `full_name` | string | required | |
| `organization_name` | string | optional | Om tom: fylls i som `<Förnamn>s Tipspromenad`. |
| `product_key` | string | required | Måste finnas i katalogen, ha `effect_type=provision_plan`. |
| `phone` | string | optional | E.164. |
| `metadata` | object | optional | Loggas, inte använt i logik. |

## Response 200

```json
{
  "status": "provisioned",
  "organization_id": "uuid",
  "user_id": "uuid",
  "magic_link": "https://app.kickinit.se/auth/callback?token=...",
  "reused": false,
  "effective_plan": "private_tipspromenad"
}
```

| Fält | Notering |
|---|---|
| `status` | `"provisioned"` (ny) eller `"already_provisioned"` (idempotent träff). |
| `magic_link` | Engångslänk, TTL 1h. Mailas av Launchpad till kunden. |
| `reused` | `true` om `external_order_ref` redan setts. |

## Error responses

| Status | `code` | Betydelse |
|---|---|---|
| 400 | `invalid_payload` | Validation failed. `details` beskriver fält. |
| 401 | `invalid_key` | Saknad/felaktig `x-provision-key`. |
| 404 | `unknown_product` | `product_key` finns inte i katalogen. |
| 409 | `email_conflict` | E-post finns redan kopplad till annan org. |
| 500 | `internal_error` | Logga `request_id` ur svaret. |

Alla felsvar har formen `{ "error": { "code": "...", "message": "...", "request_id": "..." } }`.

## Idempotens

- Samma `external_order_ref` med identisk payload → `200` + `reused: true`.
- Samma `external_order_ref` med **annan** payload → `409 idempotency_conflict`.
- Skapad org/user är permanenta — ingen "uppdatera via re-provision".

## Sidoeffekter

- `auth.users` rad (med email-confirmed).
- `quiz_organizations` rad med `plan='private_tipspromenad'`.
- `organization_members` rad (user är `org_owner`).
- `organization_subscriptions` rad (active, valid_until enl. plan).
- `org_quota_events` `provisioned`.
- Magic link genereras via Supabase admin-API.
