# Contract: inapp-products (katalog-sync)

**Version: 1.0.0** · **Status: FROZEN**
**Direction:** Tipspromenad → Launchpad (pull)
**Endpoint:** `GET {LAUNCHPAD_URL}/api/public/inapp-products`

Master-katalogen för in-app-produkter lever i Launchpad CRM. Tipspromenad-appen
pollar denna endpoint (cron eller on-demand) och speglar resultatet i sin lokala
`webshop_products`-tabell.

## Auth

Publik endpoint (read-only). Ingen header krävs.

## Response 200

```json
{
  "version": "1.0.0",
  "generated_at": "2026-05-28T12:00:00Z",
  "products": [
    {
      "slug": "private_tipspromenad_year",
      "name": "Tipspromenad — årsprenumeration",
      "description": "...",
      "price_sek": 1990,
      "currency": "SEK",
      "active": true,
      "in_app_only": false,
      "effect_type": "provision_plan",
      "effect_payload": { "duration_days": 365, "included_rounds": 12 },
      "position": 10
    }
  ]
}
```

Varje produkt MÅSTE validera mot `kickinit-catalog-v1.schema.json`.

## Sync-regler

- Klienten matchar på `slug` (primärnyckel).
- Produkter som **saknas** i svaret men finns lokalt → markeras `active=false`,
  raderas **aldrig** (bevara historik för gamla orders).
- Nya produkter → infogas.
- Befintliga → uppdateras fält för fält.

## Fel & fallback

- Om endpoint är nere: senast cachade katalog behålls. Logga varning.
- Om JSON inte validerar mot schemat: hela syncen aborteras, ingen ändring.
  Larma admin.
