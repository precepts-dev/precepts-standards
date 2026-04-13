---
identifier: "INTG-STD-003"
name: "Money and Currency"
version: "1.0.0"
status: "MANDATORY"

domain: "INTEGRATION"
documentType: "standard"
category: "format"
appliesTo: ["api", "events", "a2a", "files", "mcp"]

lastUpdated: "2026-04-10"
owner: "Integration Architecture Board"

standardsCompliance:
  iso: ["ISO-4217"]
  rfc: []
  w3c: []
  other: ["Fowler-Money-Pattern", "JSR-354"]

taxonomy:
  capability: "data-format"
  subCapability: "monetary"
  layer: "contract"

enforcement:
  method: "automated"
  validationRules:
    amountType: "string"
    amountPattern: "^-?[0-9]+(\\.[0-9]+)?$"
    currencyCodePattern: "^[A-Z]{3}$"
  rejectionCriteria:
    - "amount encoded as a JSON number (float or integer)"
    - "currency_code not in ISO 4217 alpha-3 uppercase format"
    - "amount decimal places do not match ISO 4217 minor unit count for the currency"
    - "scientific notation in amount"
    - "amount and currency_code not transmitted together"

dependsOn: ["INTG-STD-004", "INTG-STD-002"]
supersedes: ""
---

# Money and Currency

## Purpose

Monetary values represented as IEEE 754 floating-point numbers suffer from precision loss: `0.1 + 0.2 ≠ 0.3` in binary floating-point arithmetic. Even a single cent of drift, compounded across millions of transactions, produces material financial discrepancies, audit failures, and regulatory liability. This standard mandates a canonical, precision-safe money representation for all integration boundaries.

> *Normative language (**MUST**, **MUST NOT**, **SHOULD**, **MAY**) follows RFC 2119 semantics.*

---

## Conceptual Model

A monetary value is a **value object** composed of exactly two attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| `amount` | string | Decimal number in string form — preserves exact precision |
| `currency_code` | string | ISO 4217 alpha-3 currency code (uppercase) |

These two attributes are always transmitted together. An amount without a currency is meaningless. A currency without an amount is equally incomplete. Neither attribute **MUST** ever appear alone on an integration boundary.

```json
{ "amount": "149.99", "currency_code": "USD" }
```

---

## Rules

### R-1: Canonical Money Object

All monetary values **MUST** be represented as a JSON object with exactly these two fields:

- `amount` — the numeric value as a **string**.
- `currency_code` — the ISO 4217 alpha-3 code as an **uppercase string**.

Additional fields (e.g., exchange rate metadata per R-9) **MAY** be included alongside `amount` and `currency_code` but **MUST NOT** replace them.

### R-2: Amount as String

- `amount` **MUST** be encoded as a JSON **string**, not a JSON number.
- Floating-point types (`float`, `double`) **MUST NOT** be used for monetary amounts at any integration boundary.
- Integer types **MUST NOT** be used to represent amounts with fractional minor units (e.g., representing `$10.99` as `1099` cents).
- Decimal notation **MUST** be used: `"149.99"` not `"1.4999e2"`.

### R-3: Currency Code

- `currency_code` **MUST** be a valid ISO 4217 three-letter alphabetic (alpha-3) code.
- `currency_code` **MUST** be uppercase: `"USD"` not `"usd"` or `"Usd"`.
- Codes not in the current ISO 4217 list (including unofficial or crypto codes) **MUST NOT** be used without explicit bilateral agreement documented in the API specification or event schema.

### R-4: Decimal Precision

- `amount` **MUST** use exactly the number of decimal places defined by ISO 4217 for the given `currency_code` (the "minor unit" count).
- Trailing zeros **MUST** be included to reach the required precision.
- Over-precision (more decimal places than defined) **MUST NOT** be used.

| Currency | ISO 4217 Minor Units | Valid Example | Invalid Examples |
|----------|---------------------|---------------|-----------------|
| USD (US Dollar) | 2 | `"100.00"` | `"100"`, `"100.0"`, `"100.000"` |
| EUR (Euro) | 2 | `"49.99"` | `"49.990"`, `"49"` |
| GBP (British Pound) | 2 | `"29.95"` | `"29.9"` |
| JPY (Japanese Yen) | 0 | `"1000"` | `"1000.00"`, `"1000.0"` |
| KWD (Kuwaiti Dinar) | 3 | `"10.500"` | `"10.50"`, `"10.5"` |
| BHD (Bahraini Dinar) | 3 | `"5.000"` | `"5.00"` |
| TND (Tunisian Dinar) | 3 | `"99.999"` | `"99.99"` |

### R-5: Negative Amounts and Zero

- Negative amounts (refunds, credits, deductions) **MUST** use a leading minus sign: `"-25.00"`.
- Zero **MUST** be expressed with full currency precision: `"0.00"` for USD, `"0"` for JPY.
- Parentheses notation for negative values (e.g., `"(25.00)"`) **MUST NOT** be used.
- An explicit positive sign prefix **MUST NOT** be used: `"+100.00"` is invalid.

### R-6: Null vs Zero

- A `null` or absent money field **MUST** semantically represent "not applicable" or "unknown" — never a zero amount.
- A zero amount **MUST** be expressed as an explicit money object: `{"amount": "0.00", "currency_code": "USD"}`, not as `null`.
- APIs and event schemas **MUST** document whether a money field is nullable and the semantic difference between `null` and zero for that specific field.

### R-7: No Scientific Notation

- `amount` **MUST NOT** use scientific or exponential notation.
- Values such as `"1.5e2"`, `"1.5E2"`, and `"1.5e+2"` are invalid regardless of their numeric equivalence to `"150.00"`.

### R-8: Rounding

- All intermediate monetary calculations **MUST** use **HALF_EVEN** rounding (also called "banker's rounding") to minimize systematic bias.
- When rounding is applied before transmission, the rounding mode **MUST** be documented in the API specification or event schema.
- Producers **MUST NOT** silently truncate amounts: `"10.9"` transmitted as `"10.00"` for a USD field is an error — it **MUST** be `"10.90"`.
- Tax and financial regulatory contexts **MAY** mandate a specific rounding mode (e.g., HALF_UP per local VAT directives); any override **MUST** be documented in the API contract.

### R-9: Exchange Rate Representation

When an API or event needs to convey that a currency conversion occurred, producers **SHOULD** include structured exchange rate metadata:

```json
{
  "amount": "135.47",
  "currency_code": "EUR",
  "exchange_rate": {
    "rate": "1.08380",
    "base_currency_code": "USD",
    "quote_currency_code": "EUR",
    "rate_timestamp": "2026-04-10T12:00:00Z",
    "rate_source": "ecb"
  }
}
```

- `rate` **MUST** be a string with at least 5 significant decimal digits.
- `rate_timestamp` **MUST** follow INTG-STD-002 (ISO 8601 UTC with `Z` suffix).
- `base_currency_code` and `quote_currency_code` **MUST** follow R-3.
- Producers **MUST NOT** include exchange rate metadata unless an actual conversion was performed for that specific transmission.

### R-10: Cross-Currency Prohibition

- Producers **MUST NOT** aggregate or sum amounts of different currencies into a single `amount` field without performing a documented currency conversion (R-9).
- Systems receiving multi-currency amounts **MUST NOT** silently cast values to a single currency without explicit business logic and an auditable record of the conversion.

### R-11: Price Ranges

When representing a price range, producers **MUST** use a structured object with discrete money fields — not a single field with mixed semantics:

```json
{
  "price_range": {
    "min": { "amount": "9.99", "currency_code": "USD" },
    "max": { "amount": "99.99", "currency_code": "USD" }
  }
}
```

- `min.currency_code` and `max.currency_code` **MUST** be identical within the same price range object.

---

## Examples

### Valid money representations

```json
{ "amount": "149.99", "currency_code": "USD" }
{ "amount": "-25.00", "currency_code": "EUR" }
{ "amount": "0.00",   "currency_code": "GBP" }
{ "amount": "1000",   "currency_code": "JPY" }
{ "amount": "10.500", "currency_code": "KWD" }
```

### Invalid money representations

```json
{ "amount": 149.99,    "currency_code": "USD" }  // INVALID: amount is a JSON number
{ "amount": "149.9",   "currency_code": "USD" }  // INVALID: USD requires exactly 2 decimal places
{ "amount": "1000.00", "currency_code": "JPY" }  // INVALID: JPY has 0 minor units
{ "amount": "1.4999e2","currency_code": "USD" }  // INVALID: scientific notation
{ "amount": "(25.00)", "currency_code": "USD" }  // INVALID: parentheses for negative
{ "amount": "+100.00", "currency_code": "USD" }  // INVALID: explicit positive sign
{ "amount": "100",     "currency_code": "usd"  } // INVALID: currency code must be uppercase
```

### Complete example in an API response

```json
{
  "order_id": "ord_82f3k",
  "subtotal":    { "amount": "89.97", "currency_code": "USD" },
  "tax":         { "amount": "7.65",  "currency_code": "USD" },
  "shipping":    { "amount": "9.99",  "currency_code": "USD" },
  "total":       { "amount": "107.61","currency_code": "USD" },
  "amount_paid": { "amount": "107.61","currency_code": "USD" },
  "refund":      null
}
```

`refund` is `null` — meaning no refund has been issued (not applicable), not a zero-value refund. If a zero refund were explicitly required: `{"amount": "0.00", "currency_code": "USD"}`.

---

## Allowed Representations

| Context | Required Format |
|---------|----------------|
| Standard money object | `{"amount": "<string>", "currency_code": "<ISO-4217>"}` |
| With exchange rate | `{"amount": "...", "currency_code": "...", "exchange_rate": {...}}` |
| Price range | `{"min": {"amount": "...", "currency_code": "..."}, "max": {...}}` |
| Nullable field | `null` (represents "not applicable" — never zero) |

`amount` **MUST** be a string in all contexts, without exception.

---

## Validation Rules

| Rule | Constraint |
|------|------------|
| `amount` type | **MUST** be a JSON string |
| `amount` format | **MUST** match `^-?[0-9]+(\.[0-9]+)?$` |
| `amount` notation | **MUST NOT** match `[eE]` (no scientific notation) |
| `amount` precision | Decimal places **MUST** equal ISO 4217 minor unit count for `currency_code` |
| `currency_code` format | **MUST** match `^[A-Z]{3}$` |
| `currency_code` value | **MUST** be a valid ISO 4217 alpha-3 code |
| Object completeness | Both `amount` and `currency_code` **MUST** be present together |
| OpenAPI schema | `amount` **MUST** be declared `type: string`, never `type: number` |

---

## Enforcement Rules

The following **MUST** be rejected at API gateway, schema registry, or consumer validation:

| Violation | Response |
|-----------|----------|
| `amount` is a JSON number | 400 Bad Request |
| `currency_code` is lowercase | 400 Bad Request |
| `currency_code` not in ISO 4217 | 400 Bad Request |
| Decimal places mismatch for `currency_code` | 400 Bad Request |
| Scientific notation in `amount` | 400 Bad Request |
| `amount` present without `currency_code`, or vice versa | 400 Bad Request |

OpenAPI schemas **MUST** declare `amount` as `type: string` with `pattern: "^-?[0-9]+(\\.[0-9]+)?$"`. They **MUST NOT** declare it as `type: number` or `type: integer`.

---

## References

### Normative

- [ISO 4217 — Currency Codes](https://www.iso.org/iso-4217-currency-codes.html)
- [INTG-STD-004 — Naming Conventions](../foundational/naming-conventions.md) (field naming rules)
- [INTG-STD-002 — Datetime Formats](./datetime-formats.md) (timestamps in exchange rate objects)

### Informative

- [Martin Fowler — Money Pattern](https://martinfowler.com/eaaCatalog/money.html)
- [JSR-354 — Money and Currency API for Java](https://jcp.org/en/jsr/detail?id=354)
- [IEEE 754 — Floating-Point Arithmetic](https://ieeexplore.ieee.org/document/8766229) (rationale for avoiding float)
- [Stripe API — Currencies and Minor Units](https://docs.stripe.com/currencies#minor-units)

---

## Rationale

**Why string for `amount`, not JSON number?**
IEEE 754 double-precision cannot exactly represent most decimal fractions. `0.1 + 0.2` evaluates to `0.30000000000000004`. Representing `"10.99"` as a string guarantees lossless round-trip fidelity across any language and serializer. Decimal libraries (`BigDecimal` in Java, `Decimal` in Python, `decimal` in .NET) parse strings without precision loss.

**Why not integers (minor units as cents)?**
Integer encoding (e.g., `1099` to represent `$10.99`) hides the currency's scale in the field value. Code receiving `1099 JPY` and `1099 USD` must externally know which has 0 and which has 2 decimal places — a persistent source of integration bugs. String form is self-describing and human-readable.

**Why HALF_EVEN (banker's rounding)?**
HALF_UP introduces a systematic positive bias: values ending exactly in `.5` always round up, inflating aggregates over large transaction volumes. HALF_EVEN distributes rounding evenly by rounding to the nearest even digit, minimizing cumulative error. This is the IEEE 754 default and standard in financial systems.

**Why require trailing zeros (full precision)?**
A consumer parsing `"100"` for a USD amount cannot determine whether this is `100.00` exactly or a malformed value with missing digits. Requiring `"100.00"` makes precision explicit, eliminates parsing ambiguity, and signals that the producer is aware of the currency's minor unit requirements.

**Why prohibit mixing currencies in aggregates?**
Summing values across currencies requires an exchange rate that has its own precision, validity window, and audit requirements. Silently mixing currencies produces nonsensical results with no traceability to the conversion assumptions used.

---

## Version History

| Version | Date       | Change             |
| ------- | ---------- | ------------------ |
| 1.0.0   | 2026-04-10 | Initial definition |
