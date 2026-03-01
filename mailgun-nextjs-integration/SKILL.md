---
name: mailgun-nextjs-integration
description: Integrates Mailgun email sending into a Next.js (App Router) project. Use when setting up transactional emails, email templates, purchase confirmations, or any email sending via Mailgun API. No SDK needed — uses fetch directly. Based on a proven production pattern from ales-kalina.cz projects.
---

# Mailgun + Next.js Integration

Production-tested pattern pro odesílání emailů přes Mailgun v Next.js App Router projektu. **Bez SDK** — pouze nativní `fetch()`.

Reference projekt: `/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit`

## Setup

### Žádné balíčky k instalaci — stačí fetch

### Environment variables

```bash
MAILGUN_SENDING_API_KEY=key-...      # API klíč pro odesílání
MAILGUN_DOMAIN=mindsoft.cz           # tvoje Mailgun doména
```

---

## Základní funkce pro odesílání

Vytvoř helper `src/lib/mailgun.ts`:

```typescript
const MAILGUN_API_KEY = process.env.MAILGUN_SENDING_API_KEY || '';
const MAILGUN_DOMAIN = process.env.MAILGUN_DOMAIN || '';

// Pro EU doménu použij: https://api.eu.mailgun.net/v3/
const MAILGUN_ENDPOINT = `https://api.mailgun.net/v3/${MAILGUN_DOMAIN}/messages`;

export async function sendEmail({
  to,
  from,
  subject,
  html,
  text,
}: {
  to: string;
  from: string;
  subject: string;
  html?: string;
  text?: string;
}) {
  const body = new URLSearchParams({ from, to, subject });
  if (html) body.append('html', html);
  if (text) body.append('text', text);

  const response = await fetch(MAILGUN_ENDPOINT, {
    method: 'POST',
    headers: {
      Authorization: `Basic ${Buffer.from(`api:${MAILGUN_API_KEY}`).toString('base64')}`,
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: body.toString(),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Mailgun error: ${response.status} ${error}`);
  }

  return await response.json();
}
```

---

## HTML šablony

Ulož šablony jako soubory v `src/templates/emails/`:

```html
<!-- src/templates/emails/purchase-confirmation.html -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <style>
    body { font-family: sans-serif; color: #333; }
    .container { max-width: 600px; margin: 0 auto; padding: 24px; }
  </style>
</head>
<body>
  <div class="container">
    <h1>Díky za objednávku, {{to_name}}!</h1>
    <p>Potvrzujeme přijetí platby za {{product_name}}.</p>
  </div>
</body>
</html>
```

### Helper pro načtení šablony a nahrazení proměnných

```typescript
// src/utils/emailTemplate.ts
import fs from 'fs';
import path from 'path';

export function loadTemplate(filename: string): string {
  const filePath = path.join(process.cwd(), 'src/templates/emails', filename);
  return fs.readFileSync(filePath, 'utf-8');
}

export function replaceVariables(
  template: string,
  variables: Record<string, string>
): string {
  return Object.entries(variables).reduce(
    (html, [key, value]) => html.replaceAll(`{{${key}}}`, value),
    template
  );
}
```

---

## API Route pro odeslání emailu

```typescript
// src/app/api/send-confirmation/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { sendEmail } from '@/lib/mailgun';
import { loadTemplate, replaceVariables } from '@/utils/emailTemplate';

export async function POST(request: NextRequest) {
  const { email, name, productName } = await request.json();

  const template = loadTemplate('purchase-confirmation.html');
  const html = replaceVariables(template, {
    to_name: name,
    product_name: productName,
  });

  await sendEmail({
    to: email,
    from: 'Tvoje jméno <noreply@tvoje-domena.cz>',
    subject: `Potvrzení objednávky — ${productName}`,
    html,
  });

  return NextResponse.json({ sent: true });
}
```

---

## Odeslání emailu po Stripe platbě

Volej přímo z webhook handleru po úspěšné platbě:

```typescript
// V src/app/api/webhook/route.ts nebo src/lib/stripeWebhook.ts
import { sendEmail } from '@/lib/mailgun';
import { loadTemplate, replaceVariables } from '@/utils/emailTemplate';

// Po checkout.session.completed:
const template = loadTemplate('purchase-confirmation.html');
const html = replaceVariables(template, {
  to_name: meta.name,
  product_name: meta.product_type,
});

await sendEmail({
  to: meta.email,
  from: 'Tvoje jméno <noreply@tvoje-domena.cz>',
  subject: 'Potvrzení objednávky 🎉',
  html,
});
```

---

## Odeslání notifikace adminovi

```typescript
await sendEmail({
  to: 'admin@tvoje-domena.cz',
  from: 'System <noreply@tvoje-domena.cz>',
  subject: `Nová platba: ${meta.product_type}`,
  html: `<p>Email: ${meta.email}</p><p>Produkt: ${meta.product_type}</p><p>Cena: ${meta.product_price} Kč</p>`,
});
```

---

## Mailgun Dashboard Setup

1. Přihlás se na [mailgun.com](https://mailgun.com)
2. **Sending → Domains** — přidej doménu a over DNS záznamy
3. **API Keys** — vytvoř Sending API key → do `MAILGUN_SENDING_API_KEY`
4. Doménu zkopíruj do `MAILGUN_DOMAIN`

### DNS záznamy (přidej u svého registrátora)
Mailgun ti přesně ukáže co přidat — typicky TXT záznamy pro SPF a DKIM.

---

## Lokální testování

Mailgun nemá emulátor — testuj přímo:

```javascript
// scripts/test-mailgun.js
const apiKey = process.env.MAILGUN_SENDING_API_KEY;
const domain = process.env.MAILGUN_DOMAIN;

const res = await fetch(`https://api.mailgun.net/v3/${domain}/messages`, {
  method: 'POST',
  headers: {
    Authorization: `Basic ${Buffer.from(`api:${apiKey}`).toString('base64')}`,
    'Content-Type': 'application/x-www-form-urlencoded',
  },
  body: new URLSearchParams({
    from: `Test <noreply@${domain}>`,
    to: 'tvuj@email.cz',
    subject: 'Test email',
    text: 'Funguje to!',
  }).toString(),
});

console.log(await res.json());
```

```bash
node -e "$(cat scripts/test-mailgun.js)"
```

---

## Prevence duplicit

Pokud posíláš email z webhooku, přidej throttle:

```typescript
// Jednoduché řešení přes Set v paměti (nebo Redis pro produkci)
const sentEmails = new Set<string>();

function canSendEmail(key: string, windowMs = 30 * 60 * 1000): boolean {
  if (sentEmails.has(key)) return false;
  sentEmails.add(key);
  setTimeout(() => sentEmails.delete(key), windowMs);
  return true;
}

// Použití:
if (canSendEmail(`purchase-${sessionId}`)) {
  await sendEmail({ ... });
}
```

---

## Reference projekt

Plná produkční implementace: `/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit`

Klíčové soubory:
- `src/lib/stripeWebhook.ts` — odesílání emailu po platbě
- `src/utils/emailTemplate.ts` — načítání a nahrazování proměnných v šablonách
- `src/templates/emails/*.html` — HTML šablony emailů
- `src/app/api/send-result-email/route.ts` — příklad API route pro email
- `scripts/03-testing/test-mailgun.js` — testovací skript
