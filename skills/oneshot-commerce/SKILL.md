---
name: oneshot-commerce
description: |
  Search for and autonomously purchase physical products with the OneShot SDK, paid in USDC via
  x402. Use when an agent needs to find products by query and buy one — providing a product URL and
  shipping address — with a price guard. Requires OneShot wallet setup — see the `oneshot` skill first.
metadata:
  author: oneshotagent
  version: "2.0.0"
  homepage: "https://oneshotagent.com"
---

# OneShot — Commerce

Set up auth/wallet via the **`oneshot`** skill, then:

```typescript
import { OneShot } from '@oneshot-agent/sdk';
const agent = await OneShot.create({ cdp: true });
```

## Search products — `agent.commerceSearch(options)`

```typescript
const out = await agent.commerceSearch({ query: 'wireless noise-cancelling headphones', limit: 10 });
// out.products: [{ product_url, title, price, currency, vendor?, rating?, in_stock?, image_url? }]
```

Search itself is free (you pay only when buying). `limit` defaults to 10.

## Buy a product — `agent.commerceBuy(options)`

```typescript
const order = await agent.commerceBuy({
  product_url: 'https://www.amazon.com/dp/B0XXXXXXX',
  shipping_address: {
    first_name: 'Jane',
    last_name: 'Doe',
    street: '123 Main St',
    street2: 'Apt 4',        // optional
    city: 'San Francisco',
    state: 'CA',
    zip_code: '94102',
    country: 'US',           // optional
    phone: '+15551234567',   // required
    email: 'jane@example.com', // optional
  },
  quantity: 1,               // optional
  variant_id: 'size-L',      // optional
  maxCost: 120.00,           // STRONGLY recommended — guards total incl. shipping/tax/fee
  memo: 'replacement headphones',
});
// order.{ status, order_id, order_status, tracking_url? }
```

**`shipping_address` required fields** (`ShippingAddress`): `first_name`, `last_name`, `street`,
`city`, `state`, `zip_code`, `phone`. Optional: `street2`, `country`, `email`.

The buy flow quotes total (subtotal + shipping + tax + fee); set `maxCost` to the most you'll
pay or the call fails fast before signing. It's an async order — polls to completion by default.

## Pricing

Product search is free; a purchase costs the product total plus a service fee (guard with
`maxCost`). See current per-tool pricing at https://docs.oneshotagent.com/pricing.
