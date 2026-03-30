# Internationalization & Localization Skill for Production Applications

> Universal i18n/l10n guidelines for full-stack production software (CRMs, ERPs, task managers, SaaS platforms, etc.)

---

## Role

You are a senior engineer building a production application that must work correctly for users across different languages, regions, and cultures. Every string, date, number, and currency you render must adapt to the user's locale. Internationalization is not a feature you add later — it is a foundation you build from day one. Retrofitting i18n into a hardcoded application is one of the most expensive refactors in software.

---

## Key Concepts

- **Internationalization (i18n)**: making the codebase capable of supporting multiple locales — externalizing strings, using locale-aware formatting, handling text direction. This is engineering work.
- **Localization (l10n)**: adapting the application for a specific locale — translating strings, adjusting date/currency formats, adapting imagery and tone. This is content work.
- **Locale**: a combination of language + region (e.g., `pt-BR` = Portuguese as used in Brazil, `en-US` = English as used in the United States).

---

## 1. String Externalization

### Rules
- Never hardcode user-facing strings in source code. Every string the user sees must come from a translation file or resource bundle.
- This includes: labels, buttons, placeholders, error messages, validation messages, tooltips, notifications, email subjects and bodies, PDF/report content.
- Organize translation files by locale:
  ```
  locales/
  ├── en-US.json
  ├── pt-BR.json
  ├── es-ES.json
  └── fr-FR.json
  ```
- Use a consistent key naming convention. Prefer namespaced, descriptive keys over generic ones:
  ```
  // BAD: ambiguous, will collide
  "save": "Save"
  "title": "Title"

  // GOOD: namespaced and contextual
  "invoice.actions.save": "Save Invoice"
  "invoice.form.title_label": "Invoice Title"
  "common.actions.cancel": "Cancel"
  ```
- Never concatenate strings to build sentences. Different languages have different word orders:
  ```
  // BAD: assumes English word order
  translate("showing") + " " + count + " " + translate("results")
  // Breaks in Japanese, German, Arabic, etc.

  // GOOD: use interpolation with a full sentence template
  translate("search.results_count", { count: 42 })
  // en-US: "Showing 42 results"
  // pt-BR: "Exibindo 42 resultados"
  // ja-JP: "42件の結果を表示中"
  ```

### Pluralization
- Never assume pluralization is just "add an s". Every language has different plural rules. Some have 1 form, others have up to 6.
- Use ICU MessageFormat or your framework's pluralization system:
  ```
  // Translation file
  "inbox.message_count": "{count, plural, =0 {No messages} one {1 message} other {{count} messages}}"

  // Polish has different rules:
  "inbox.message_count": "{count, plural, =0 {Brak wiadomości} one {1 wiadomość} few {{count} wiadomości} many {{count} wiadomości} other {{count} wiadomości}}"
  ```
- Always handle zero, one, and many/other cases at minimum.

### Text Length and Layout
- Translated text can be significantly longer or shorter than the source language. German is typically 30-40% longer than English. Chinese/Japanese can be shorter.
- Design UI components to accommodate text expansion: avoid fixed-width containers for text, use flexible layouts, test with long strings.
- Never truncate translated text without an overflow strategy (tooltip, expand, ellipsis with full text accessible).

---

## 2. Date and Time

### Rules
- Always store dates and times in UTC in the database. No exceptions.
- Convert to the user's local timezone only at the display layer (frontend or API response serialization).
- Never format dates with hardcoded patterns in application logic. Use the locale-aware Intl API or equivalent:
  ```
  // BAD: assumes MM/DD/YYYY
  `${date.getMonth()}/${date.getDate()}/${date.getFullYear()}`

  // GOOD: locale-aware
  new Intl.DateTimeFormat('pt-BR').format(date)  // "30/03/2026"
  new Intl.DateTimeFormat('en-US').format(date)  // "3/30/2026"
  new Intl.DateTimeFormat('ja-JP').format(date)  // "2026/3/30"
  ```
- Always include timezone information in API responses that contain timestamps. Use ISO 8601 format:
  ```
  "created_at": "2026-03-30T14:30:00Z"
  ```
- Store the user's preferred timezone in their profile. Fall back to browser-detected timezone, then to UTC.

### Relative Time
- Use relative time formatting for recent events ("5 minutes ago", "yesterday") with locale-aware libraries.
- Show the absolute date/time on hover or after a threshold (e.g., older than 7 days).
- Relative time must also be localized ("há 5 minutos", "il y a 5 minutes", "vor 5 Minuten").

### Calendar Systems
- If the application serves users in regions with non-Gregorian calendars (Hijri, Buddhist, Japanese Imperial), support calendar conversion at the display layer.
- Default to the Gregorian calendar but allow user preference override.

---

## 3. Numbers and Currency

### Number Formatting
- Never hardcode thousand separators or decimal points. These vary by locale:
  ```
  // 1,234,567.89 in en-US
  // 1.234.567,89 in pt-BR and de-DE
  // 1 234 567,89 in fr-FR
  ```
- Use locale-aware formatting APIs:
  ```
  new Intl.NumberFormat('pt-BR').format(1234567.89)  // "1.234.567,89"
  new Intl.NumberFormat('en-US').format(1234567.89)  // "1,234,567.89"
  ```
- Store numbers as actual numbers in the database (INTEGER, DECIMAL, FLOAT). Never store formatted strings.

### Currency
- Always store monetary values with their currency code (ISO 4217: BRL, USD, EUR, JPY).
- Never store currency as a floating point number. Use integer cents/minor units or a fixed-precision decimal type:
  ```
  // BAD: floating point
  { "amount": 19.99, "currency": "USD" }  // 0.1 + 0.2 ≠ 0.3

  // GOOD: integer minor units
  { "amount_cents": 1999, "currency": "USD" }

  // GOOD: fixed-precision decimal
  { "amount": "19.99", "currency": "USD" }  // stored as DECIMAL(19,4) in DB
  ```
- Format currency using locale-aware APIs:
  ```
  new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' }).format(19.99)
  // "R$ 19,99"

  new Intl.NumberFormat('ja-JP', { style: 'currency', currency: 'JPY' }).format(1999)
  // "￥1,999"
  ```
- Not all currencies have 2 decimal places. JPY has 0, BHD has 3. Use the locale's default for the currency.

### Units of Measurement
- If the application deals with measurements (weight, distance, temperature, volume), store in a base unit (metric) and convert at display time based on locale or user preference.
- Provide a user-level setting to choose between metric and imperial when relevant.

---

## 4. Text Direction (RTL Support)

### Rules
- If the application may serve users who read right-to-left languages (Arabic, Hebrew, Farsi, Urdu), implement bidirectional (bidi) text support.
- Use the `dir` attribute on the HTML root element and toggle based on locale:
  ```html
  <html lang="ar" dir="rtl">
  ```
- Use CSS logical properties instead of physical properties:
  ```css
  /* BAD: breaks in RTL */
  margin-left: 16px;
  padding-right: 8px;
  text-align: left;

  /* GOOD: adapts to text direction */
  margin-inline-start: 16px;
  padding-inline-end: 8px;
  text-align: start;
  ```
- Mirror the entire layout for RTL: navigation moves to the right, back arrows point right, progress bars fill from right to left.
- Icons that imply direction (arrows, reply, undo) must be flipped in RTL. Icons that are universal (search, settings, close) should not be flipped.
- Never assume text direction from language alone. Some content may be mixed (e.g., English brand names in an Arabic sentence). Use Unicode bidi algorithm and `<bdi>` tags for user-generated content.

---

## 5. Locale Detection and User Preferences

### Detection Priority
- Apply locale in this order of priority:
  1. **User's explicit preference** stored in their profile.
  2. **URL-based locale** if the application uses locale-prefixed routes (`/pt-BR/dashboard`, `/en/dashboard`).
  3. **Browser's `Accept-Language` header** as the initial default.
  4. **Application default locale** as the final fallback.

### Locale Switching
- Allow users to change their language/locale at any time without losing context (current page, form state, etc.).
- Store the preference server-side (user profile) so it persists across devices.
- When the user switches locale, reload translations but do not reload the entire application if possible.

### Fallback Strategy
- If a translation key is missing in the user's locale, fall back to the default locale (usually en-US) instead of showing a raw key like `invoice.actions.save`.
- Log missing translations as warnings in development and staging so they can be caught before production.
- Never show raw translation keys to users in production.

---

## 6. Backend Considerations

### API Design
- Accept and return locale information in API requests:
  - `Accept-Language` header for content negotiation.
  - User's locale stored in the session/token for server-rendered content.
- Return dates in ISO 8601 UTC. Let the client format for display.
- Return numbers as raw values. Let the client format for display.
- Return currency with amount and currency code separately. Let the client format.
- For error messages and validation messages returned by the API, use error codes (not translated strings) so the client can map them to the correct locale:
  ```
  // BAD: server returns translated string
  { "error": "O e-mail é obrigatório" }

  // GOOD: server returns a code, client translates
  { "error_code": "validation.email.required", "field": "email" }
  ```

### Database
- Use UTF-8 (specifically utf8mb4 in MySQL) for all text columns. This supports all languages, emojis, and special characters.
- Set the database collation to a locale-appropriate or universal option (e.g., `utf8mb4_unicode_ci`).
- If the application supports user-generated content in multiple languages, consider full-text search configurations per language (different stemmers, stop words).

### Email and Notifications
- Template emails and notifications per locale. Never assemble translated emails from concatenated parts.
- Send communications in the user's preferred locale.
- Include a plain-text fallback in addition to HTML for all localized emails.

---

## 7. Testing i18n

### Rules
- Include at least one non-Latin language in test suites (e.g., Japanese, Arabic, Chinese) to catch encoding issues early.
- Test with pseudo-localization: a fake locale that replaces characters with accented variants and adds padding to simulate text expansion:
  ```
  "Save Invoice" → "[Ŝåṿé Ìñṿöîçé______]"
  ```
  This reveals hardcoded strings (they won't be pseudo-translated) and layout overflow issues (the padding simulates text expansion).
- Test RTL layout with an actual RTL locale, not just by flipping CSS manually.
- Verify that dates, numbers, and currencies format correctly for at least 3 diverse locales (e.g., en-US, pt-BR, ja-JP).
- Test with long locale strings (German, Finnish) to verify that no UI element clips or overflows.
- Verify that the locale fallback chain works: remove a key from a locale file and confirm the default locale string appears instead of a raw key.

### Automated Checks
- Add a CI check that compares translation files and reports missing keys per locale.
- Add a linting rule that flags hardcoded user-facing strings in source code.
- Add visual regression tests for RTL layouts if RTL support is required.

---

## 8. Common Pitfalls

- **Sorting**: alphabetical sort order varies by language. Use locale-aware collation (`Intl.Collator` or equivalent), not naive string comparison.
- **Name formatting**: not everyone has a "first name" and "last name". Some cultures use a single name, others have patronymics. Use a single "full name" field or flexible name components.
- **Address formatting**: address structures vary wildly across countries. Use a flexible address model or a library like Google's `libaddressinput`.
- **Phone numbers**: always store in E.164 international format (`+5511999999999`). Use a library for parsing and formatting (libphonenumber).
- **Color and imagery**: colors have cultural associations (white = mourning in some Asian cultures, red = luck in China, danger in the West). Avoid encoding meaning solely through color.
- **Legal and regulatory**: some regions require specific content (cookie banners in the EU, LGPD consent in Brazil, specific disclaimer text in financial services). Treat these as locale-dependent content.

---

## 9. Code Review i18n Checklist

When reviewing or writing code, always verify:

- [ ] No user-facing strings are hardcoded in source code — all externalized to translation files
- [ ] String assembly uses interpolation templates, not concatenation
- [ ] Pluralization uses ICU MessageFormat or framework equivalent, not if/else with "s"
- [ ] Dates are stored in UTC and formatted at the display layer using locale-aware APIs
- [ ] Numbers and currencies are formatted with locale-aware APIs, not hardcoded patterns
- [ ] Monetary values are stored as integer cents or fixed-precision decimals, never floating point
- [ ] Currency is stored with its ISO 4217 code alongside the amount
- [ ] API returns error codes (not translated strings) for client-side translation
- [ ] Database uses UTF-8 encoding (utf8mb4 for MySQL)
- [ ] CSS uses logical properties (inline-start/end) instead of physical (left/right) if RTL is supported
- [ ] UI components handle text expansion without clipping or overflow
- [ ] Missing translations fall back gracefully to the default locale
- [ ] Translation files are complete — CI check confirms no missing keys
- [ ] User's locale preference is stored in their profile and respected across the application
