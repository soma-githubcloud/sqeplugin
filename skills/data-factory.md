# Data Factory — Reusable Skill

Patterns and rules for generating synthetic test data using `@faker-js/faker` and
equivalence partitioning. Load this into the `agents/capability-3/datagen-agent.md` for all fixture generation tasks.

---

## Core Principles

1. **Factories are functions, not constants** — called fresh per test to avoid state sharing
2. **Three data classes**: positive (valid), boundary (edge cases), negative (invalid/malicious)
3. **Deterministic when seeded**: use `faker.seed(123)` in test setup for reproducible data
4. **Never hardcode** test data in spec files — always call a factory or load from JSON file

---

## Field Type → Faker Method Mapping

| Field Name Pattern | Faker Method | Example Output |
|---|---|---|
| `*email*` | `faker.internet.email()` | `"john.doe@example.net"` |
| `*password*` | `faker.internet.password({ length: 12, memorable: false })` | `"xK9#mP2qL5rT"` |
| `*firstName*`, `*first_name*` | `faker.person.firstName()` | `"Emily"` |
| `*lastName*`, `*last_name*` | `faker.person.lastName()` | `"Thompson"` |
| `*fullName*`, `*name*` | `faker.person.fullName()` | `"Emily Thompson"` |
| `*phone*`, `*mobile*` | `faker.phone.number()` | `"555-234-5678"` |
| `*address*`, `*street*` | `faker.location.streetAddress()` | `"123 Main St"` |
| `*city*` | `faker.location.city()` | `"Springfield"` |
| `*state*`, `*region*` | `faker.location.state()` | `"California"` |
| `*zip*`, `*postal*` | `faker.location.zipCode()` | `"90210"` |
| `*country*` | `faker.location.country()` | `"United States"` |
| `*company*`, `*org*` | `faker.company.name()` | `"Acme Corp"` |
| `*productName*`, `*title*` | `faker.commerce.productName()` | `"Ergonomic Chair"` |
| `*price*`, `*amount*` | `parseFloat(faker.commerce.price())` | `49.99` |
| `*description*` | `faker.commerce.productDescription()` | `"Quality product..."` |
| `*uuid*`, `*id*` | `faker.string.uuid()` | `"a1b2c3d4-..."` |
| `*date*`, `*createdAt*` | `faker.date.recent().toISOString()` | `"2026-03-01T..."` |
| `*url*`, `*website*` | `faker.internet.url()` | `"https://example.com"` |
| `*username*` | `faker.internet.userName()` | `"john_doe42"` |
| `*bio*`, `*about*` | `faker.lorem.sentence()` | `"A short sentence."` |
| default / unknown | `faker.lorem.word()` | `"lorem"` |

---

## Factory Template Pattern

```typescript
import { faker } from '@faker-js/faker';

// ─── User Factory ─────────────────────────────────────────────────────────────
export const userFactory = (overrides: Partial<User> = {}): User => ({
  email:     faker.internet.email(),
  password:  faker.internet.password({ length: 12, memorable: false }),
  firstName: faker.person.firstName(),
  lastName:  faker.person.lastName(),
  ...overrides,   // allow callers to override specific fields
});

// Usage: const user = userFactory({ email: 'fixed@test.com' });
```

---

## Boundary Value Classes

For string fields, generate these boundary cases:

| Class | Rule | Example |
|---|---|---|
| Empty | `""` | Empty string |
| Minimum | 1 character | `"a"` |
| Near minimum | 2-3 characters | `"ab"` |
| Valid | Normal valid value | Normal faker output |
| Near maximum | maxLength - 1 | `"a".repeat(maxLength - 1)` |
| Maximum | maxLength | `"a".repeat(maxLength)` |
| Over maximum | maxLength + 1 | `"a".repeat(maxLength + 1)` |
| Null | `null` | Null value |
| Undefined | `undefined` | Undefined value |
| Whitespace | `"   "` | Spaces only |

For numeric fields: 0, 1, -1, MIN_INT, MAX_INT, 0.001, MAX_FLOAT, NaN, Infinity

---

## Negative / Invalid Data Classes

| Type | Examples |
|---|---|
| Malformed email | `"notanemail"`, `"@nodomain.com"`, `"missing@"`, `"double@@test.com"` |
| SQL injection | `"' OR '1'='1"`, `"1; DROP TABLE users--"`, `"' UNION SELECT * FROM users--"` |
| XSS payloads | `"<script>alert('xss')</script>"`, `"<img src=x onerror=alert(1)>"` |
| Path traversal | `"../../etc/passwd"`, `"..\\windows\\system32"` |
| Unicode edge cases | `"用户@test.com"`, `"😀"`, `"\u0000"` (null byte) |
| Long strings | `"a".repeat(10000)` |
| HTML entities | `"&lt;script&gt;"`, `"&#x3C;script&#x3E;"` |

---

## JSON Data File Format

```json
{
  "_meta": { "entity": "user", "type": "valid", "count": 5, "generatedAt": "..." },
  "data": [
    { "email": "john.doe@example.com", "password": "xK9#mP2qL5", ... },
    { "email": "jane.smith@test.org",  "password": "aB7$nR4vW8", ... }
  ]
}
```

---

## Integration with Playwright Tests

```typescript
// In spec file (example usage pattern):
import { userFactory } from '@fixtures/factories';
import userData from '@fixtures/data/user.valid.json';

test('login with generated user', async ({ page }) => {
  const user = userFactory();             // fresh random data
  await loginPage.login(user.email, user.password);
});

test.each(userData.data)('login with %s', async ({ email, password }) => {
  await loginPage.login(email, password);  // data-driven from JSON
});
```
