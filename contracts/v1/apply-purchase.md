# Contract: apply-purchase

**Version: 1.0.0** · **Status: FROZEN**
**Direction:** Launchpad → Tipspromenad
**Endpoint:** `POST {TIPSPROMENAD_FUNCTIONS_URL}/apply-purchase`

Applicerar effekterna av ett in-app-tillköp (extra rundor, förlängd tid,
nya entitlements) på en befintlig organisation. **Idempotent** på `order_ref`.

## Auth

Header: `x-provision-key: <PRIVATE_ORG_PROVISION_KEY>` (samma secret som provision).

## Request

```json
{
  "order_ref": "kickinit-2026-000456",
  "organization_id": "uuid",
  "product_key": "topup_10_rounds",
  "metadata": {
    "stripe_session_id": "cs_live_..."
  }
}
```

| Fält | Typ | Krav |
|---|---|---|
| `order_ref` | string | required, unik idempotensnyckel |
| `organization_id` | uuid | required |
| `product_key` | string | required, måste finnas i katalogen |
| `metadata` | object | optional |

## Response 200

```json
{
  "status": "applied",
  "order_ref": "kickinit-2026-000456",
  "product_key": "topup_10_rounds",
  "result_summary": "added_10_rounds",
  "reused": false
}
```

| `status` | Betydelse |
|---|---|
| `applied` | Effekt applicerades nu. |
| `already_applied` | `order_ref` redan setts; ingen dubbelapplicering. |

`result_summary` är textuell (`added_10_rounds+extended_30_days+granted_paper_print`)
och avsedd för loggning, inte parsning.

## Error responses

| Status | `code` | Betydelse |
|---|---|---|
| 400 | `invalid_payload` | |
| 401 | `invalid_key` | |
| 404 | `unknown_org` | `organization_id` finns inte. |
| 404 | `unknown_product` | |
| 409 | `no_subscription` | `add_rounds`/`extend_days` kräver befintlig prenumeration. |
| 500 | `internal_error` | |

## Effekttyper (förankrade i `effect_type` i katalogen)

Se `kickinit-catalog-v1.schema.json` för fullständig lista. Sammanfattning:

- `provision_plan` — skapar prenumeration. Används primärt från `provision-private-org`.
- `add_rounds` — `{ rounds: int }`.
- `extend_days` — `{ days: int }`.
- `grant_feature` — `{ feature: "results_email" | "member_lookup" | "paper_print" }`.

Nya effekttyper kräver minor-bump (`1.x.0`) **och** schema-uppdatering.
