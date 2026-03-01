---
name: email-validation-nextjs
description: Multi-layer email validation for Next.js forms. Detects typos in email domains (gmail.cmo → gmail.com), blocks disposable/fake emails (mailinator, yopmail, tempmail, etc.), and validates format on both client and server. Use when adding email input validation, form email checking, or protecting lead forms from fake emails. Based on production pattern from ales-kalina.cz.
---

# Validace emailů v Next.js

Třívrstvý systém validace emailů pro formuláře. Bez externích knihoven — vše vlastní logika.

Reference projekt: `/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit`
- Implementace: `src/lib/email-validator.ts`
- Použití ve formuláři: `src/components/EmailModal.tsx`

---

## Jak systém funguje

```
Uživatel zadá email
        ↓
1. onBlur → checkFakeEmail()    ← blokující (červené varování, smaže email)
        ↓
   onBlur → checkEmailDomain()  ← neblokující (žluté varování, návrh opravy)
        ↓
2. onSubmit → regex validace    ← blokující
        ↓
3. API route → validateEmail()  ← poslední pojistka na serveru
```

---

## Soubor `src/lib/email-validator.ts`

Zkopíruj celý soubor z referenčního projektu:
`/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit/src/lib/email-validator.ts`

Obsahuje 3 exportované funkce:

| Funkce | Kde | Co dělá |
|--------|-----|---------|
| `checkFakeEmail(email)` | client onBlur | Blokuje 70+ disposable domén + regex vzory |
| `checkEmailDomain(email)` | client onBlur | Detekuje překlepy (gmail.cmo → gmail.com) |
| `validateEmail(email)` | server API | Validace formátu, délky, struktury |

---

## Použití ve formuláři (React)

```typescript
// State
const [emailWarning, setEmailWarning] = useState('');
const [suggestedEmail, setSuggestedEmail] = useState('');

// onBlur handler — spustí se když uživatel opustí pole
const handleEmailBlur = async () => {
  if (!email) return;

  const { checkEmailDomain, checkFakeEmail } = await import('@/lib/email-validator');

  // 1. Nejdřív fake/disposable (vyšší priorita)
  const fakeResult = checkFakeEmail(email);
  if (fakeResult.isFake) {
    setEmailWarning(fakeResult.reason || 'Použij reálnou emailovou adresu.');
    setSuggestedEmail('');
    return;
  }

  // 2. Pak překlepy
  const typoResult = checkEmailDomain(email);
  if (typoResult.warning) {
    setEmailWarning(typoResult.warning);
    if (typoResult.suggestedDomain) {
      setSuggestedEmail(`${email.split('@')[0]}@${typoResult.suggestedDomain}`);
    }
  } else {
    setEmailWarning('');
    setSuggestedEmail('');
  }
};

// Před odesláním formuláře — blokuj pokud je aktivní varování
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  if (emailWarning) return;  // MUSÍ se vyřešit varování

  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    setError('Prosím zadejte platnou emailovou adresu.');
    return;
  }
  // ... odeslání
};
```

### Input s validací

```tsx
<input
  type="email"
  value={email}
  onChange={(e) => setEmail(e.target.value)}
  onBlur={handleEmailBlur}
  pattern="[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}"
  title="Zadej platnou emailovou adresu (např. jmeno@domena.cz)"
  required
/>
```

### Zobrazení varování

```tsx
{emailWarning && (
  <div className={`mt-2 p-3 rounded border-2 ${
    suggestedEmail ? 'bg-yellow-50 border-yellow-300' : 'bg-red-50 border-red-300'
  }`}>
    <p className={suggestedEmail ? 'text-yellow-800' : 'text-red-800'}>
      {emailWarning}
    </p>

    {suggestedEmail ? (
      // Překlep — nabídni opravu
      <div className="flex gap-2 mt-2">
        <button onClick={() => { setEmail(suggestedEmail); setEmailWarning(''); setSuggestedEmail(''); }}>
          Použít {suggestedEmail}
        </button>
        <button onClick={() => { setEmailWarning(''); setSuggestedEmail(''); }}>
          Ponechat můj email
        </button>
      </div>
    ) : (
      // Fake email — smaž a nech znovu zadat
      <button onClick={() => { setEmailWarning(''); setEmail(''); }}>
        Změnit email
      </button>
    )}
  </div>
)}

{/* Submit button — disabled pokud je varování */}
<button type="submit" disabled={!!emailWarning}>
  {emailWarning ? 'Oprav email nejdříve' : 'Odeslat'}
</button>
```

---

## Server-side validace (API route)

```typescript
// src/app/api/submit-user/route.ts
import { validateEmail } from '@/lib/email-validator';

export async function POST(request: NextRequest) {
  const { email } = await request.json();

  const emailValidation = validateEmail(email.trim());
  if (!emailValidation.isValid) {
    return NextResponse.json(
      { error: emailValidation.error },
      { status: 400 }
    );
  }

  // ... zpracování
}
```

---

## Co `checkFakeEmail` detekuje

- **RFC 2606 testovací domény**: example.com, example.org, example.net
- **Obecné testovací**: test.com, test.cz, testing.com, test123.com
- **Fake domény**: fake.com, fakemail.com, fakeemail.com
- **70+ disposable služeb**: mailinator.com, yopmail.com, tempmail.com, 10minutemail.com, guerrillamail.com, trashmail.com, maildrop.cc, ...
- **Regex vzory**: cokoli začínající `temp*`, `fake*`, `trash*`, `spam*`, `throwaway*`, `\d+min*` atd.

## Co `checkEmailDomain` detekuje

Porovnává doménu pomocí **Levenshtein distance** (≤2 znaky rozdíl) oproti:
- CZ/SK: seznam.cz, email.cz, centrum.cz, volny.cz, azet.sk, zoznam.sk
- Světové: gmail.com, outlook.com, hotmail.com, yahoo.com, icloud.com, live.com
- Ostatní: protonmail.com, proton.me, zoho.com, aol.com, gmx.com

Příklady: `gmail.cmo` → `gmail.com`, `sezanm.cz` → `seznam.cz`

---

## Poznámka k Mailgun Email Validation API

V referenčním projektu je implementace Mailgun API validace (`MAILGUN_PRIVATE_API_KEY`) — ale je **vypnutá** z důvodu ceny (~50 $/měsíc).

Volá: `https://api.mailgun.net/v4/address/validate?address={email}`

Vrací: `result` (deliverable/undeliverable), `risk` (low/medium/high), `is_disposable_address`, `did_you_mean`

Pokud bys to chtěl zapnout — viz zakomentovaný kód v:
`/Users/aleskalina/CascadeProjects/Zustat-nebo-odejit/src/app/api/validate-email/route.ts`
