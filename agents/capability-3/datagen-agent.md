# DataGen Agent

**Single Responsibility**: Generate synthetic test data fixtures using `@faker-js/faker` and
equivalence partitioning principles. Outputs TypeScript factory functions and static JSON data files.

---

## Inputs (passed by calling command)

| Parameter | Type | Description |
|---|---|---|
| `entities` | string[] | Entity names to generate (e.g., `["user", "product"]`) |
| `count` | number | Rows per entity in JSON files (default: 5) |
| `boundary` | boolean | Generate boundary value sets |
| `negative` | boolean | Generate negative/invalid data sets |
| `schema` | object | null | JSON Schema object if provided |
| `intakeFile` | string | null | Path to `intake.summary.json` for auto-entity detection |
| `kitConfig` | object | Full kit/kit.config.json (resolve via PLUGIN_DIR) |

---

## Step 1: Determine entities

Priority:
1. If `schema` provided → extract entity names and field definitions from JSON Schema
2. If `entities` array provided → use as entity names; infer field types from name patterns
3. If `intakeFile` provided → read test cases → extract data values from step `value` fields → infer entities

**Field type inference from field name patterns**:
| Name pattern | Faker method |
|---|---|
| `*email*` | `faker.internet.email()` |
| `*password*` | `faker.internet.password({ length: 12 })` |
| `*firstName*`, `*first_name*` | `faker.person.firstName()` |
| `*lastName*`, `*last_name*` | `faker.person.lastName()` |
| `*phone*` | `faker.phone.number()` |
| `*address*`, `*street*` | `faker.location.streetAddress()` |
| `*city*` | `faker.location.city()` |
| `*zip*`, `*postal*` | `faker.location.zipCode()` |
| `*company*` | `faker.company.name()` |
| `*title*`, `*productName*` | `faker.commerce.productName()` |
| `*price*`, `*amount*` | `faker.commerce.price()` |
| `*description*` | `faker.commerce.productDescription()` |
| `*id*`, `*uuid*` | `faker.string.uuid()` |
| `*date*` | `faker.date.recent().toISOString()` |
| default | `faker.lorem.word()` |

---

## Step 2: Generate `factories.ts`

Load template: `templates/factories.ts.tmpl` (resolve via PLUGIN_DIR)

Build context: one factory function per entity with all inferred fields.

Write to: `<outputDir>/<projectName>/src/fixtures/factories.ts`

**Example output**:
```typescript
import { faker } from '@faker-js/faker';

export const userFactory = () => ({
  email:     faker.internet.email(),
  password:  faker.internet.password({ length: 12, memorable: false }),
  firstName: faker.person.firstName(),
  lastName:  faker.person.lastName(),
});

export const productFactory = () => ({
  title:       faker.commerce.productName(),
  price:       parseFloat(faker.commerce.price()),
  description: faker.commerce.productDescription(),
  id:          faker.string.uuid(),
});
```

---

## Step 3: Generate valid JSON data files

For each entity, call the factory `count` times and write to:
`<outputDir>/<projectName>/src/fixtures/data/<entity>.valid.json`

```json
[
  { "email": "john.doe@example.com", "password": "xK9#mP2qL5", ... },
  { "email": "jane.smith@test.org",  "password": "aB7$nR4vW8", ... }
]
```

---

## Step 4: Generate boundary value sets (if `boundary = true`)

Write to: `<outputDir>/<projectName>/src/fixtures/data/<entity>.boundary.json`

Standard boundary classes:
- **Empty**: `{ "email": "", "password": "" }`
- **Min length** (1 char): `{ "email": "a@b.c", "password": "A" }`
- **Max length** (256 chars): long string at field max
- **Null**: `{ "email": null, "password": null }`
- **Whitespace only**: `{ "email": "   ", "password": "   " }`

---

## Step 5: Generate negative/invalid sets (if `negative = true`)

Write to: `<outputDir>/<projectName>/src/fixtures/data/<entity>.invalid.json`

| Type | Example |
|---|---|
| Malformed email | `"notanemail"`, `"@nodomain"`, `"missing@"` |
| SQL injection | `"' OR 1=1 --"`, `"1; DROP TABLE users"` |
| XSS | `"<script>alert(1)</script>"`, `"javascript:void(0)"` |
| Unicode edge cases | `"用户@test.com"`, `"тест@пример.рф"` |
| Overflow | 10000-character string |

---

## Validation

1. `npx tsc --noEmit` — `factories.ts` must compile with no errors
2. Confirm `@faker-js/faker` is in `package.json` devDependencies; if missing, add it
3. Report file paths and row counts for each generated file
