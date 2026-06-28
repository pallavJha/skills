## TypeScript

### JS1 — Use proper types
Always use explicit types. Do not use `any` or `unknown` unless genuinely required. If `unknown` is needed, narrow the type with a type guard before use.

```ts
// Bad
function process(data: any) {
  return data.name;
}

// Good
interface User {
  name: string;
  age: number;
}

function process(data: User): string {
  return data.name;
}
```

### JS2 — No implicit any
Do not suppress type errors with `@ts-ignore` or `as any`.

Grep: `@ts-ignore|as any|: any\b`

### JS3 — Abbreviations in names are fully uppercased
Acronyms and abbreviations in class, interface, variable, and function names must be fully uppercased. Do not title-case them.

Grep: `(Http|Url|Uri|Api|Json|Xml|Html|Uuid|Sql|Css)` — common abbreviations only; identifiers with other abbreviations are still judged by reading.

```ts
// Bad
export interface HttpHeaders {
export interface ApiResponse {
export interface JsonSchema {
export function parseUuid(): string {

// Good
export interface HTTPHeaders {
export interface APIResponse {
export interface JSONSchema {
export function parseUUID(): string {
```

### JS4 — Use path aliases instead of deep relative imports
Long relative paths are hard to read. Create path aliases and use those.

Grep: `(from |require\()'(\.\./){2,}`

```js
// Bad
import { getOrderForSession } from '../../../lib/orders/repository.js';
import { listOrderIds, getOrder } from '../../../lib/orders/queries.js';

<script src="../../scripts/orders/init.ts"></script>

// Good
import { getOrderForSession } from '@lib/orders/repository.js';
import { listOrderIds, getOrder } from '@lib/orders/queries.js';

<script src="@scripts/orders/init.ts"></script>
```

### JS5 — All custom errors must extend a BaseError class
The `BaseError` class should extend `Error` and implement `toJSON` so that `Error.prototype` properties are not lost during serialization.

Any error that extends `BaseError` and adds its own fields must override `toJSON` to include those fields, calling `super.toJSON()` so the base properties are preserved. Otherwise the subclass's fields are silently dropped during serialization.

```ts
// Bad — `orderId` is lost when serialized
class OrderError extends BaseError {
  constructor(message, public orderId) {
    super(message);
  }
}

// Good
class OrderError extends BaseError {
  constructor(message, public orderId) {
    super(message);
  }

  toJSON() {
    return { ...super.toJSON(), orderId: this.orderId };
  }
}
```

Grep: `extends Error\b`, `extends BaseError\b`

### JS6 — Check content type before parsing response body
`await res.json()` is dangerous without confirming the content type header first. The server may return HTML, plain text, or an empty body — all of which will throw an unhelpful parse error.

Grep: `await .*\.json\(\)`

### JS7 — Error messages should describe what the caller was trying to do
The message should make sense to someone reading a log without the source code open.

```ts
// Bad
throw new Error('create response missing id');

// Good
throw new Error('cannot find id in the create-order API response');
```

### JS8 — Log the error object, not just the message
Logging `err.message` discards the stack trace and the `cause` chain. Always log the full error object.

Grep: `err(or)?\.message`

```ts
// Bad
c.on('error', (err) => console.error('[queue] client error:', err.message));

// Good
c.on('error', (err) => console.error('[queue] client error:', err));
```

## Code Style

### JS9 — Always use braces for control structures
Never write single-line `if`, `for`, `while`, or other control structures. Always use braces on a new line.

Grep: `^\s*(if|for|while|else if) ?\(.*\) +[^{\s]` , `^\s*(if|for|while) ?\(.*\{.*\}`

```ts
// Bad
if (this.completed) console.log('completed');

// Bad
for (const item of items) processItem(item);

// Good
if (this.completed) {
  console.log('completed');
}

// Good
for (const item of items) {
  processItem(item);
}
```

### JS10 — No complex ternaries
A simple single-condition ternary (`flag ? a : b`) is fine. The moment the predicate gets an `&&`/`||` chain, or a branch contains another ternary, split it into `if`/`else` or pull the predicate into a named boolean.

Grep: `(&&|\|\|)[^?:]*\?[^.]` , `\? [^:]*\? `

```js
// Bad
const pageSize =
  Number.isInteger(raw) && raw >= MIN && raw <= MAX
    ? raw
    : undefined;

// Bad (nested)
const x = a ? b : c ? d : e;

// Good
let pageSize;
if (Number.isInteger(raw) && raw >= MIN && raw <= MAX) {
  pageSize = raw;
}

// Also good — named predicate
const inRange = Number.isInteger(raw) && raw >= MIN && raw <= MAX;
const pageSize = inRange ? raw : undefined;
```

### JS11 — Multi-line object literals in argument positions
Object literals passed as function arguments get one property per line, even short ones. Diffs read cleaner and each key gets horizontal space.

```js
// Bad
JSON.stringify({identifier: $identifier.trim(), retry})

// Good
JSON.stringify({
  identifier: $identifier.trim(),
  retry,
})
```

### JS12 — Sentinel filtering: assign first, then reset
When external input may carry a sentinel meaning "missing" (`"N/A"`, `-1`, `"unknown"`), assign the raw value to the real variable, then reset it to `undefined` if it matches. Don't gate the assignment behind a predicate with a throwaway temp.

```js
// Bad
const header = req.get('x-some-header');
let headerValue;
if (header && header !== 'N/A' && header !== 'unknown') {
  headerValue = header;
}

// Good
let headerValue = req.get('x-some-header');
if (headerValue === 'N/A' || headerValue === 'unknown') {
  headerValue = undefined;
}
```

### JS13 — Check Node built-ins before inventing constants
Before defining `const FOO = '...'` for a system-level value, check whether Node already names it: `dns.NOTFOUND`, `os.constants.errno.E*`, `os.constants.signals.SIG*`, `fs.constants.*`, `http.STATUS_CODES`. Only define a local constant when the value is project-specific.

### JS14 — Minimal types in shared modules
In a shared utility module, annotate with Node built-ins (`http.IncomingMessage`, `http.ServerResponse`, `Buffer`, `URL`) instead of framework types (`express.Request`) unless the framework is already a dependency of that module. Don't add a dependency just to annotate a parameter.

### JS22 — Don't create variables with poor names
A variable's name is its only documentation at the call site. Three failure modes:

1. **Truncated-word abbreviations** (`tn`, `sp`, `cur`, `ph`). Save no real space, lose all meaning. Use the word.
2. **Single-letter locals outside tight idioms.** `i`/`j` as loop indices, `e`/`err` in a catch, `_` for unused are fine. `t` for a target or `m` for a match is not.
3. **Generic placeholders** (`data`, `result`, `value`, `item`). Pick a name that says what the thing is — `payload`, `charSpan`, `textNode`.

```js
// Bad
let t = byType.get(type);
const lo = Math.max(sp.from, tn.start);
const hi = Math.min(sp.to, tn.start + tn.len);

// Good
let target = byType.get(type);
const overlapStart = Math.max(charSpan.from, textNode.start);
const overlapEnd = Math.min(charSpan.to, textNode.start + textNode.len);
```

Grep: `\b(const|let|var) [a-z]{1,2}[ ;=,)]` — candidates only; idiomatic short names (`i`, `el`, `err`) are dismissed by reading.

### JS23 — One statement per line, no compressed constructs
Never put multiple statements on one line, and never hide a side effect (assignment) inside a loop or branch condition. Code reads top to bottom, one action per line.

```js
// Bad
if (!t) { t = { el, type }; byType.set(type, t); }
for (let m, re = /\S+/g; (m = re.exec(concat)); ) words.push({ from: m.index });

// Good
let target = byType.get(type);
if (!target) {
  target = { el, type };
  byType.set(type, target);
}

for (const match of concat.matchAll(/\S+/g)) {
  words.push({ from: match.index });
}
```

Grep: `\{[^{}]*;[^{}]*;` , `while ?\(.*=[^=]` — `for(;;)` headers are dismissals.

### JS24 — Keep shared domain types in `types.ts`
Each package should keep its shared domain types in a dedicated `types.ts` file at the package root. For example, `lib/internal/mail` should define `lib/internal/mail/types.ts` for mail-specific error classes that extend `BaseError`, error codes, `AuthenticationResult`, and other types imported by multiple modules. Keep implementation details and types used by only one module in that module instead of turning `types.ts` into a catch-all.

```text
lib/internal/mail/
├── types.ts       # shared errors, codes, and result types
├── authenticate.ts
└── send.ts
```

## JSDoc

### JS15 — Every function gets a JSDoc; trivial locals don't
A function signature is a contract, so document every `@param` and `@return`, even on a two-liner. Skip JSDoc on trivial local variables whose type and purpose are obvious from the assigning expression.

### JS16 — JSDoc must be crisp, never loquacious
One short sentence for the contract, one line per `@param`/`@return`. No "this function does X" preambles, no prose walls. Add `@example` only when it surfaces an edge case the types don't convey.

### JS17 — Inline per-property descriptions inside `{{…}}` object types
For a multi-field options object, keep the object-literal type intact and put each property's description on the same line after its type. Don't split into separate `@param [opts.foo]` lines and don't describe all fields in one prose paragraph.

```js
// Good
/**
 * @param {{
 *   nonce?: string  client nonce preserved across the oauth2 hop.
 * }} [opts]
 */
```

### JS18 — Keep docs layer-agnostic
Backend JSDoc must not name frontend components, screens, or UI state (and vice versa). Describe only the contract the function itself exposes — the API should not care what callers do with the response.

## Svelte Components

### JS19 — Scoped styles
Use Svelte's `<style>` block for component-specific styles that have no Bootstrap equivalent. Prefer Bootstrap utility classes in the template first.

### JS20 — Props
Use Svelte 5 `$props()` rune with a TypeScript `interface Props`.

### JS21 — Store subscriptions
Import nanostores atoms without `$` prefix (e.g., `import { projects } from './store'`). Use `$projects` in Svelte templates for reactive subscriptions.
