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
