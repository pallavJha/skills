## TypeScript

### Use proper types
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

### No implicit any
Do not suppress type errors with `@ts-ignore` or `as any`.

### Abbreviations in names are fully uppercased
Acronyms and abbreviations in class, interface, variable, and function names must be fully uppercased. Do not title-case them.

```ts
// Bad
export interface SmtpMeta {
export interface SpfResult {
export interface DkimSignature {
export function startSse(): EventSource {

// Good
export interface SMTPMeta {
export interface SPFResult {
export interface DKIMSignature {
export function startSSE(): EventSource {
```

### Use path aliases instead of deep relative imports
Long relative paths are hard to read. Create path aliases and use those.

```js
// Bad
import { getEmailForSession } from '../../../lib/email/repository.js';
import { listMessageIds, getMessage } from '../../../lib/email/messages.js';

<script src="../../scripts/email/init.ts"></script>

// Good
import { getEmailForSession } from '@lib/email/repository.js';
import { listMessageIds, getMessage } from '@lib/email/messages.js';

<script src="@scripts/email/init.ts"></script>
```

### All custom errors must extend a BaseError class
The `BaseError` class should extend `Error` and implement `toJSON` so that `Error.prototype` properties are not lost during serialization.

### Check content type before parsing response body
`await res.json()` is dangerous without confirming the content type header first. The server may return HTML, plain text, or an empty body — all of which will throw an unhelpful parse error.

### Error messages should describe what the caller was trying to do
The message should make sense to someone reading a log without the source code open.

```ts
// Bad
throw new Error('allocate response missing email');

// Good
throw new Error('cannot find email in the email allocate API response');
```

## Code Style

### Always use braces for control structures
Never write single-line `if`, `for`, `while`, or other control structures. Always use braces on a new line.

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

## Svelte Components

### Scoped styles
Use Svelte's `<style>` block for component-specific styles that have no Bootstrap equivalent. Prefer Bootstrap utility classes in the template first.

### Props
Use Svelte 5 `$props()` rune with a TypeScript `interface Props`.

### Store subscriptions
Import nanostores atoms without `$` prefix (e.g., `import { projects } from './store'`). Use `$projects` in Svelte templates for reactive subscriptions.
