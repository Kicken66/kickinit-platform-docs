# Contract: stripe-checkout-integration

**Version: 1.0.0** · **Status: FROZEN**
**Scope:** Hur KickinIT-projekten delar upp Stripe-ansvar.
**Berörda projekt:**

- **KickinIT CRM Suite** (master för produkter, Stripe-ägare)
- **KickinIT Tipspromenad** (denna app — in-app-checkout för inloggade org-admins)
- **KickinIT Launchpad / kickinit.se/shop** (anonyma och marknadsköp)

Detta är **single source of truth** för hur Stripe-integrationen är uppdelad mellan
projekten. Brott mot denna fil kräver major-bump.

---

## 1. Roller

| Projekt | Ansvar |
|---|---|
| **CRM Suite** | Master-katalog (DB). Skapar/uppdaterar Stripe `product` + `price`. Sätter `lookup_key = slug`. Publicerar `/api/public/inapp-products`. |
| **Tipspromenad** | Egen `create-checkout` (embedded) + `payments-webhook` för inloggade org-admins. Äger `apply-purchase`-endpointen — **enda provisioneringskanalen**. |
| **Launchpad / Shop** | Egen `create-checkout` + `payments-webhook` för anonyma köp och marknadsbesök. Anropar Tipspromenads `apply-purchase` efter betalning. |

Ingen konsument får anropa Stripe Admin API för att skapa produkter eller priser. Det är CRM:s ansvar.

---

## 2. Slug-kontrakt (nyckeln som korsar systemgränser)

- Regex: `^[a-z0-9_]+$`
- **Immutable** — en `slug` får aldrig återanvändas eller byta betydelse.
- Samma värde används i:
  - CRM-DB (`products.slug`)
  - Stripe `price.lookup_key` (och `product.metadata.slug`)
  - Katalog-JSON (`products[].slug` enligt `kickinit-catalog-v1.schema.json`)
  - `apply-purchase`-body (`product_key`)

Konsumenter får **aldrig** lagra Stripes interna `prod_xxx` / `price_xxx`. De slår
alltid upp aktuellt pris vid checkout-tillfället:

```ts
stripe.prices.list({ lookup_keys: [slug], active: true, limit: 1 })
```

---

## 3. CRM:s ansvar (master)

### Aktivera Stripe
Använd Lovables inbyggda Stripe-integration (`enable_stripe_payments`). Test-läge
först; live-läge när företagskonto finns.

### När en produkt skapas
```ts
const product = await stripe.products.create({
  name,
  description,
  metadata: { slug },
});
await stripe.prices.create({
  product: product.id,
  unit_amount: price_cents,
  currency: 'sek',
  lookup_key: slug,
  transfer_lookup_key: true,
  ...(recurring && { recurring: { interval: 'year' } }),
});
```

### När pris ändras
Skapa ett **nytt** `price` med samma `lookup_key` och `transfer_lookup_key: true`
— Stripe flyttar automatiskt nyckeln och arkiverar det gamla priset. Konsumenter
behöver inte göra något.

### När produkt deaktiveras
Arkivera priset och produkten i Stripe. Sätt `active=false` i CRM-katalogen.
**Radera aldrig** — historiska orders måste kunna refereras.

### Skatt
Företaget är momsbefriat tills vidare. **Sätt inga `tax_code`.** Konsumenter
sätter heller inte `automatic_tax` eller `managed_payments` på sessioner. När
omsättningstaket nås: bump till v1.1.0, sätt `tax_code` på alla produkter, och
konsumenter aktiverar `automatic_tax: { enabled: true }`.

### Spegling
Spara `stripe_product_id` och `stripe_price_id` på CRM-raden för admin-visning
och felsökning. Konsumenter **läser dem inte**.

---

## 4. Konsumenters ansvar (Tipspromenad + Shop)

Båda projekten implementerar samma mönster:

### `create-checkout` edge function

```ts
// 1. Validera auth (Tipspromenad: org-admin-gate. Shop: optional auth.)
// 2. Slå upp Stripe-pris:
const prices = await stripe.prices.list({ lookup_keys: [slug], active: true, limit: 1 });
if (!prices.data.length) return 404 "product_not_in_stripe";
const price = prices.data[0];

// 3. Resolva Stripe Customer med metadata.userId (+ orgId om finns)
// 4. Skapa session:
const session = await stripe.checkout.sessions.create({
  line_items: [{ price: price.id, quantity: 1 }],
  mode: price.type === 'recurring' ? 'subscription' : 'payment',
  ui_mode: 'embedded_page',
  return_url,
  customer: customerId,
  metadata: { slug, orgId, userId, source: 'tipspromenad' /* eller 'shop' */ },
  // INGEN automatic_tax, INGEN managed_payments (momsbefriat).
});
return { clientSecret: session.client_secret };
```

### `payments-webhook` edge function

Lyssnar på `checkout.session.completed`. Anropar Tipspromenads `apply-purchase`:

```ts
await fetch(`${TIPSPROMENAD_FUNCTIONS_URL}/apply-purchase`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-provision-key': PRIVATE_ORG_PROVISION_KEY,
  },
  body: JSON.stringify({
    order_ref: session.id,        // automatisk idempotens
    organization_id: session.metadata.orgId,
    product_key: session.metadata.slug,
    metadata: { stripe_session_id: session.id, source: session.metadata.source },
  }),
});
```

Returnera 200 även på ohanterade event-typer.

---

## 5. Provisioneringskontrakt

Se [`apply-purchase.md`](./apply-purchase.md). Sammanfattning:

- **Endpoint:** `POST {TIPSPROMENAD_FUNCTIONS_URL}/apply-purchase`
- **Header:** `x-provision-key: $PRIVATE_ORG_PROVISION_KEY`
- **Body:** `{ order_ref, organization_id, product_key, metadata }`
- **Idempotens:** `order_ref` (använd Stripe `session.id` rakt av)

---

## 6. UX-uppdelning

| Besökare | Köpväg |
|---|---|
| Inloggad org-admin (`org_owner` / `org_admin`) i Tipspromenad | In-app embedded checkout |
| Org-medlem utan admin-rättighet | "Be en admin köpa" — ingen knapp |
| Anonym besökare, marknadsbesök, nya kunder | kickinit.se/shop |

Båda flödena landar i samma org-state efter `apply-purchase`. En `slug` kan
säljas via båda kanalerna parallellt utan synkproblem.

---

## 7. Felmoder och hantering

| Fel | Hantering |
|---|---|
| `lookup_key` saknas i Stripe | `create-checkout` returnerar 404 — säg "Produkten är inte tillgänglig just nu, försök igen om en stund". CRM behöver pusha. |
| Stripe API nere | Visa felmeddelande, behåll shop-länken som fallback i Tipspromenad. |
| Webhook förlorad | Stripe retryar i 3 dagar. Idempotens via `order_ref = session.id` säkrar att vi inte dubbel-provisionerar vid retry. |
| `apply-purchase` returnerar 409 `no_subscription` | Loggas. Användaren behöver först köpa en `provision_plan` innan top-ups fungerar. |
| Användare betalar men byter org mellan checkout och webhook | Webhook använder `metadata.orgId` från session — provisioneras alltid på rätt org. |

---

## 8. Testflöde end-to-end

1. CRM skapar/uppdaterar produkt → verifiera att `lookup_key = slug` finns i Stripe sandbox.
2. Öppna AddonsPage i Tipspromenad som org-admin (eller `/shop?product=<slug>` i Launchpad).
3. Embedded checkout öppnas. Använd `4242 4242 4242 4242`, valfritt framtida datum, valfri CVC.
4. Bekräfta att webhook tas emot (`payments-webhook`-logg).
5. Bekräfta att `provision_orders`-raden finns + att `organization_subscriptions` / `organization_entitlements` uppdaterades.
6. Verifiera idempotens: trigga webhooken igen manuellt → ska svara `already_applied`.

---

## 9. Versionering

Semver:

- **Major** — något i §2 (slug-kontrakt) eller §5 (apply-purchase-body) ändras på ett bakåtinkompatibelt sätt.
- **Minor** — nya effekt-typer, nya valfria fält, ny `tax_code`-policy.
- **Patch** — förtydliganden, fix av exempel.

Aktuell version: **1.0.0**. Brott mot detta dokument utan version-bump = bugg.
