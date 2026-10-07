# @formfusion/iban

Set of validation rules for worldwide IBAN numbers.

A zero-dependency lookup table of **79 country-specific regex patterns** for validating IBAN (International Bank Account Number) values. Every pattern is anchored and works directly as an HTML [`pattern`](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/pattern) attribute value, so you can use it with plain HTML, React, FormFusion, or `new RegExp()`.

## Why

IBAN length and shape are fixed per country, and they are nowhere near alike. Germany is 22 characters: `DE`, two check digits, then 18 digits. France is 27: `FR`, two check digits, 10 digits, 11 alphanumerics, 2 digits. Norway is 15. Malta mixes four letters, five digits and 18 alphanumerics. One pattern cannot cover all of them, so every app that collects bank details ends up re-implementing and re-maintaining this table.

This package ships it as one flat object so you don't have to.

## Installation

```bash
npm install @formfusion/iban
```

```bash
yarn add @formfusion/iban
```

## Usage

The package exports a single default object mapping lowercase ISO 3166-1 alpha-2 country codes to regex **strings**.

### ES modules

```js
import iban from '@formfusion/iban';

console.log(iban.de); // "^(DE[0-9]{2})\\d{18}$"
```

### CommonJS

```js
const iban = require('@formfusion/iban').default;

new RegExp(iban.de).test('DE89370400440532013000'); // true
new RegExp(iban.de).test('DE8937040044053201300'); // false - wrong length
new RegExp(iban.de).test('de89370400440532013000'); // false - lowercase
```

### Plain HTML

The patterns are valid `pattern` attribute values, so they work without any JavaScript:

```html
<label for="iban">IBAN (Germany)</label>
<input id="iban" name="iban" type="text" pattern="^(DE[0-9]{2})\d{18}$" required />
```

### With FormFusion

FormFusion passes unknown `type` values straight through to the input's `pattern` attribute, so you can hand it a pattern directly:

```jsx
import React from 'react';
import { Form, Input } from 'formfusion';
import 'formfusion/style.css';
import iban from '@formfusion/iban';

const MyForm = () => (
  <Form onSubmit={(data) => console.log('Submitted', data)}>
    <Input id="iban" name="iban" type={iban.de} label="IBAN" required />
    <button type="submit">Submit</button>
  </Form>
);
```

Patterns compose with FormFusion's `rules` combinators if you need to accept more than one country:

```jsx
import { Input, rules } from 'formfusion';
import iban from '@formfusion/iban';

// Accept either a German or an Austrian IBAN
<Input name="iban" type={rules.existIn([iban.de, iban.at])} />
```

### Dynamic country selection

```jsx
const [country, setCountry] = useState('de');

<Select name="country" value={country} onChange={setCountry}>
  {Object.keys(iban).map((code) => (
    <option key={code} value={code}>
      {code.toUpperCase()}
    </option>
  ))}
</Select>

{iban[country] ? (
  <Input name="iban" type={iban[country]} label="IBAN" required />
) : (
  <Input name="iban" type="text" label="IBAN" required />
)}
```

### Standalone validation

Strip the display spaces and uppercase before testing, then add a mod-97 check if you want real verification:

```js
import iban from '@formfusion/iban';

export function isValidIban(value, country) {
  const pattern = iban[String(country).toLowerCase()];

  if (!pattern) return false; // unknown country

  const compact = value.replace(/\s+/g, '').toUpperCase();
  return new RegExp(pattern).test(compact);
}

isValidIban('DE89 3704 0044 0532 0130 00', 'DE'); // true
isValidIban('DE8937040044053201300', 'DE'); // false
```

### TypeScript

Typings are hand-written in `index.d.ts` and mirror the lowercase keys via a mapped type. Because the declaration uses `export =`, you need `esModuleInterop` or `allowSyntheticDefaultImports`.

```ts
import iban from '@formfusion/iban';

const de: string = iban.de;
// @ts-expect-error - unknown country
const xx: string = iban.xx;
```

## API

The export is a plain object with no functions or classes:

```ts
{ [countryCode: string]: string }
```

Country codes are **lowercase** (`de`, `gb`, `at`). Lookups are case-sensitive, so normalize user input first.

The country prefix **and its two check digits are required in every pattern**. `iban.de` matches `DE89370400440532013000` and rejects the same value without the `DE` prefix.

Greece is keyed `gr` here. In [`@formfusion/vat`](https://www.npmjs.com/package/@formfusion/vat) it is keyed `el`, the EU-standard prefix.

Coverage is 79 countries. There are no `us`, `ca`, `au`, `jp`, `in`, `cn`, `kr`, `ru`, `mx`, `ar`, `za`, `hk`, `sg` or `nz` entries, because those countries do not use the IBAN system.

Enumerate the available codes at runtime with `Object.keys(iban)`.

## Caveats

Read these before relying on the patterns.

**Format only, no checksum.** These are shape checks. IBAN defines a mod-97 checksum (ISO 13616) that this package does not run, so `DE00370400440532013000` passes while `DE89370400440532013000` is the valid one. Verify the checksum yourself if it matters:

```js
function mod97(value) {
  const reordered = value.slice(4) + value.slice(0, 4);
  let remainder = 0;

  for (const char of reordered) {
    const digits = /[0-9]/.test(char) ? char : String(char.charCodeAt(0) - 55);
    for (const digit of digits) remainder = (remainder * 10 + Number(digit)) % 97;
  }

  return remainder;
}

mod97('DE89370400440532013000'); // 1 - valid
mod97('DE00370400440532013000'); // 9 - not valid
```

**Spaces are rejected.** IBANs are printed in blocks of four (`DE89 3704 0044 0532 0130 00`), but no pattern accepts whitespace. Strip spaces before validating.

**Lowercase input is rejected.** Every pattern expects an uppercase country prefix and uppercase BBAN characters. Uppercase the value first.

**Unknown keys are `undefined`, and `new RegExp(undefined)` matches everything.** `iban.xx` is not `null`, it is `undefined`, and the resulting regex is `(?:)`. Always guard before use.

## Development

```bash
git clone https://github.com/mitevskasara/formfusion-iban.git
cd formfusion-iban
npm install
npm run build
```

### How it works

All source lives in [`src/index.js`](src/index.js) as a single object of uppercase country codes. The last step lowercases every key before exporting, so `AT` becomes `at`.

[`esbuild.js`](esbuild.js) bundles that into a minified CommonJS `index.js` at the repo root, targeting Node 14. Consumers get the built file, so **changes are not live until you rebuild and commit `index.js`**:

```bash
npm run build
```

### Commit convention

This repo follows [Conventional Commits](https://www.conventionalcommits.org/), and `CHANGELOG.md` is generated from those subjects:

```
Feat: add Albanian IBAN pattern
Fix: correct Maltese length
```

### Adding a country

1. Add the entry to `src/index.js`, using an uppercase country code.
2. Add the same uppercase code to the `IbanPatterns` type in `index.d.ts`. The mapped type derives the lowercase key for you.
3. Run `npm run build` and commit the regenerated `index.js`.

## Related packages

Part of the FormFusion family of extracted validation rule sets:

- [`@formfusion/postcodes`](https://www.npmjs.com/package/@formfusion/postcodes)
- [`@formfusion/licence-plates`](https://www.npmjs.com/package/@formfusion/licence-plates)
- [`@formfusion/passports`](https://www.npmjs.com/package/@formfusion/passports)
- [`@formfusion/phones`](https://www.npmjs.com/package/@formfusion/phones)
- [`@formfusion/tin`](https://www.npmjs.com/package/@formfusion/tin)
- [`@formfusion/vat`](https://www.npmjs.com/package/@formfusion/vat)
- [`formfusion`](https://www.npmjs.com/package/formfusion) — the core library

## Issues

Report bugs and feature requests at https://github.com/mitevskasara/formfusion-iban/issues.

## License

BSD-2-Clause. Copyright (c) 2023, Mitevska Sara.
