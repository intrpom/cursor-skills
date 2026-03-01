---
name: stripe-nextjs-integration
description: Integrates Stripe payments into a Next.js (App Router) project. Use when setting up Stripe checkout, webhooks, upsells, or payment processing in a Next.js app. Covers checkout session creation, webhook handling, product configuration, and success/cancel pages. Based on a proven production pattern from ales-kalina.cz Stripe account.
---

# Stripe + Next.js Integration

Production-tested pattern for integrating Stripe Checkout into a Next.js App Router project. Uses the same Stripe account as the **Zůstat nebo odejit** project (`/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit`) — you can reference it for real examples.

## Stack

- `stripe` (server SDK) + `@stripe/stripe-js` (client)
- Next.js App Router API routes
- No database needed — all data stored in Stripe metadata

## Setup

### 1. Install packages

```bash
npm install stripe @stripe/stripe-js
```

### 2. Environment variables

```bash
STRIPE_SECRET_KEY=sk_live_...        # or sk_test_... for testing
STRIPE_WEBHOOK_SECRET=whsec_...       # from Stripe Dashboard → Webhooks
NEXT_PUBLIC_BASE_URL=https://your-domain.cz
```

---

## File Structure

```
src/
├── constants/
│   └── stripeConfig.ts          # Product definitions
├── app/api/
│   ├── create-checkout-session/route.ts
│   ├── webhook/route.ts
│   └── get-checkout-session/route.ts  # optional, for upsells
└── app/
    ├── objednavka-uspesna/page.tsx    # success page
    └── stripe-return/page.tsx         # cancel page
```

---

## Product Config (`src/constants/stripeConfig.ts`)

```typescript
import Stripe from 'stripe';

export interface ProductConfig {
  name: string;
  unit_amount: number;   // in haléře (CZK cents), e.g. 59900 = 599 Kč
  currency: 'czk';
  mode: Stripe.Checkout.SessionCreateParams.Mode;
  product_type: string;
  product_tag: string;
  success_url: (baseUrl: string) => string;
  cancel_url: (baseUrl: string) => string;
}

export const STRIPE_PRODUCTS: Record<string, ProductConfig> = {
  'my-product': {
    name: 'Název produktu',
    unit_amount: 59900,   // 599 Kč
    currency: 'czk',
    mode: 'payment',
    product_type: 'my_product',
    product_tag: 'my-product-tag',
    success_url: (base) => `${base}/objednavka-uspesna`,
    cancel_url: (base) => `${base}/`,
  },
};
```

---

## Checkout Session API (`src/app/api/create-checkout-session/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { STRIPE_PRODUCTS } from '@/constants/stripeConfig';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || '', {
  apiVersion: '2025-04-30.basil',
});

export async function POST(request: NextRequest) {
  const { email, name, phone, productKey, currentPrice } = await request.json();

  const productConfig = STRIPE_PRODUCTS[productKey];
  if (!productConfig) {
    return NextResponse.json({ error: 'Unknown product' }, { status: 400 });
  }

  const baseUrl = process.env.NEXT_PUBLIC_BASE_URL || '';
  const unitAmount = currentPrice
    ? Math.round(currentPrice * 100)
    : productConfig.unit_amount;

  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    line_items: [{
      price_data: {
        currency: productConfig.currency,
        product_data: { name: productConfig.name },
        unit_amount: unitAmount,
      },
      quantity: 1,
    }],
    mode: productConfig.mode,
    customer_creation: 'always',
    payment_intent_data: { setup_future_usage: 'off_session' },  // enables upsells
    success_url: productConfig.success_url(baseUrl),
    cancel_url: productConfig.cancel_url(baseUrl),
    customer_email: email || undefined,
    metadata: {
      name, email, phone,
      product_type: productConfig.product_type,
      product_tag: productConfig.product_tag,
      product_price: String(unitAmount / 100),
      app_identifier: 'your-app-name',  // used in webhook to filter events
    },
  });

  return NextResponse.json({ url: session.url });
}
```

---

## Webhook (`src/app/api/webhook/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || '', {
  apiVersion: '2025-04-30.basil',
});

export async function POST(request: NextRequest) {
  const body = await request.text();
  const sig = request.headers.get('stripe-signature') || '';

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET || '');
  } catch {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object as Stripe.Checkout.Session;
    const meta = session.metadata || {};

    // Filter: only process events from this app
    if (meta.app_identifier !== 'your-app-name') {
      return NextResponse.json({ received: true, ignored: true });
    }

    // TODO: your business logic here
    // - send confirmation email
    // - add customer to CRM
    // - unlock access to product
    console.log('Payment received:', {
      email: meta.email,
      product: meta.product_type,
      price: meta.product_price,
    });
  }

  return NextResponse.json({ received: true });
}
```

> **Důležité**: Webhook route musí mít vypnutý body parser. V Next.js App Routeru je to výchozí pro `request.text()` — žádná extra konfigurace není potřeba.

---

## Frontend: Spuštění platby

```typescript
// In a client component or hook
async function handlePayment() {
  const res = await fetch('/api/create-checkout-session', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: 'user@example.com',
      name: 'Jan Novák',
      phone: '+420601234567',
      productKey: 'my-product',
      currentPrice: 599,  // optional override
    }),
  });
  const { url } = await res.json();
  window.location.href = url;  // redirect to Stripe Checkout
}
```

---

## Stripe Dashboard Setup

1. **Webhook endpoint**: Stripe Dashboard → Developers → Webhooks → Add endpoint
   - URL: `https://your-domain.cz/api/webhook`
   - Events: `checkout.session.completed`
   - Copy the signing secret → `STRIPE_WEBHOOK_SECRET`

2. **Local testing**: Use Stripe CLI
   ```bash
   stripe listen --forward-to localhost:3000/api/webhook
   ```

---

## Optional: Upsell (one-click second purchase)

After first purchase, retrieve the customer ID and charge again without a new checkout:

```typescript
// GET /api/get-checkout-session?session_id=...
const session = await stripe.checkout.sessions.retrieve(sessionId);
const customerId = session.customer as string;

// POST /api/create-upsell-session
const paymentIntent = await stripe.paymentIntents.create({
  amount: 150000,  // 1500 Kč
  currency: 'czk',
  customer: customerId,
  payment_method: session.payment_method_configs?.[0] as string,
  confirm: true,
  off_session: true,
  metadata: { product_type: 'upsell', app_identifier: 'your-app-name' },
});
```

---

## Reference project

Full production implementation: `/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit`

Key files to copy/adapt:
- `src/constants/stripeConfig.ts` — product definitions
- `src/app/api/create-checkout-session/route.ts` — checkout creation
- `src/app/api/webhook/route.ts` — webhook handler
- `src/lib/stripeWebhook.ts` — webhook processing logic
- `src/lib/webhookValidation.ts` — security validation
