## Markup formatting

### HTML1 — Don't break attributes one-per-line with a dangling bracket
When a tag has too many attributes for one line, wrap them across a few lines and align the continuation to the first attribute — keep several attributes per line. Never put one attribute per line, and never drop the closing `>` (or `/>`) onto its own line. Indentation of the children already shows where the tag ends; a bracket sitting alone on a line is noise, and one-attribute-per-line stretches a single element over a dozen lines.

Grep: `^\s*/?>\s*$` — a line that is only `>` or `/>`.

```html
<!-- Bad — every attribute on its own line, `>` dangling on its own line -->
<button
  type="button"
  on:click={onContinue}
  disabled={!canContinue || busy}
  aria-label="Continue"
  aria-disabled={!canContinue || busy}
  data-testid="start-continue"
>
  <span
    class="rounded border"
  >
    {busy ? 'Checking...' : 'Continue'}
  </span>

  {#if busy}
    <span
      aria-hidden="true"
    />
  {/if}
</button>

<!-- Good — attributes wrap aligned to the first one, brackets stay on the line -->
<button type="button" on:click={onContinue}
        disabled={!canContinue || busy} aria-label="Continue"
        aria-disabled={!canContinue || busy} data-testid="start-continue">
  <span class="rounded border">
    {busy ? 'Checking...' : 'Continue'}
  </span>

  {#if busy}
    <span aria-hidden="true" />
  {/if}
</button>
```
