# Test Annotations: Complete Reference

## Introduction

Tags, Annotations, `skip`, `only`, `fixme`, and `slow` are each simple on their own. This document gathers them in one place: a reference to come back to, not a new explanation.

The difference between `skip`, `fixme`, and `fail` gets lost quickly, and the conditional syntax is often written wrong. Keep this file open next to you for the first month, and after that it'll be second nature.

## Objectives

After reading this, you should be able to:

- Choose between `skip`, `fixme`, and `fail` in a second instead of stopping to think
- Write the conditional syntax correctly the first time
- Categorize your tests with Tags and run a subset of them from the terminal
- Control the behavior of an entire file with `describe.configure` and `test.use`

## Reference Table

| Syntax | What it does | When to use it |
|---|---|---|
| `test.skip()` | Doesn't run it at all | The feature isn't available in this environment |
| `test.skip(condition, reason)` | Conditional skip | A specific browser or environment |
| `test.only()` | Runs only this one | Local debugging only |
| `test.fixme()` | Skips + marks it as "needs fixing" | The test itself is broken |
| `test.fail()` | Passes when it fails | A known Bug that isn't fixed yet |
| `test.slow()` | Triples the timeout | A test that is slow by nature |
| `test.setTimeout(ms)` | Custom timeout | When `slow` isn't enough |
| `{ tag: ['@smoke'] }` | Categorization for selective runs | Every test |
| `{ annotation: {...} }` | Information in the report | Linking to a ticket or bug |
| `test.describe.configure()` | Behavior of a whole group | Serial order, retries, timeout |
| `test.use()` | Change settings for a file or group | Locale, viewport, session |

## The Four That Get Mixed Up: `skip` · `fixme` · `fail` · `only`

These are the foundation of everything, and the difference between them isn't in the behavior. It's in the **message**.

### `test.skip`: "This shouldn't run here"

```ts
test.skip('upload a large file', async ({ page }) => {
  // The feature isn't enabled on the staging environment
});
```

The test is fine; the environment isn't suitable. It shows up in the report as `skipped` and nobody worries.

### `test.fixme`: "It's broken and I know it"

```ts
test.fixme('delete a todo from the archive', async ({ page }) => {
  // The Locator changed after the new design
});
```

Same result (it doesn't run), completely different message: this test carries debt and someone needs to come back to it.

> **Important**
> The difference isn't technical, it's communication with the team. `skip` says "everything is fine", and `fixme` says "there's unfinished work". Using `skip` in place of `fixme` is how problems quietly disappear until they blow up.

### `test.fail`: "It's supposed to fail"

```ts
test.fail('search with Arabic characters', async ({ page }) => {
  await page.getByPlaceholder('Search').fill('حليب');
  await expect(page.getByTestId('results')).toHaveCount(1);
});
```

This is the smartest one and the least used. The test actually runs, and it **passes when it fails**. Meaning there's a known Bug, and you've documented it in code.

More importantly: the day the developer fixes the Bug, the test will fail and you'll notice to remove `fail`. Compare that to `skip`, which stays asleep for a year.

### `test.only`: A Dangerous Weapon

```ts
test.only('the test I am working on right now', async ({ page }) => {});
```

Excellent locally, a disaster if it gets merged: CI runs a single test, comes back green, and you feel reassured.

> **Warning**
> Put `forbidOnly: !!process.env.CI` in the config file. One line that stops CI if it finds `only` anywhere. This is not optional advice.

## Conditional Syntax

All of these have three forms, and this is where the mistakes happen.

### Form 1: At Definition

```ts
test.skip('test name', async ({ page }) => { /* ... */ });
```

Always skips, with no condition.

### Form 2: Inside the Test

```ts
test('download a file', async ({ page, browserName }) => {
  test.skip(browserName === 'webkit', 'Download is not supported in WebKit');
  // rest of the test
});
```

The reason is practically mandatory: it shows up in the report. `test.skip(true)` with no reason leaves whoever comes after you guessing.

### Form 3: In a `describe` for All Tests

```ts
test.describe('mobile tests', () => {
  test.skip(({ isMobile }) => !isMobile, 'Mobile only');

  test('side menu opens', async ({ page }) => {});
});
```

Here it takes a function that receives the fixtures.

> **Warning**
> Common mistake: writing `test.skip()` after an `await` inside the test. It must be at the very start of the test, before any action. Playwright can't cancel a test that has already reached its middle.

The same three forms apply to `fixme`, `fail`, and `slow` exactly.

## Tags

The categorization that lets you run only what you want:

```ts
test('login with a valid user', { tag: ['@smoke', '@auth'] }, async ({ page }) => {
  // ...
});

test.describe('payment tests', { tag: '@slow' }, () => {
  // every test inside gets the tag
});
```

And running them:

```
npx playwright test --grep @smoke
npx playwright test --grep-invert @slow
npx playwright test --grep "@smoke.*@auth"
```

> **Tip**
> Three Tags are enough for most teams: `@smoke` for critical paths, `@regression` for the rest, and `@slow` for the ones that take time. More than five Tags and someone has to remember them, and nobody will.

## Annotations

Information attached to the test that appears in the HTML report:

```ts
test('delete a todo', {
  annotation: [
    { type: 'issue', description: 'https://github.com/example/todo/issues/42' },
    { type: 'owner', description: 'QA Team' },
  ],
}, async ({ page }) => {
  // ...
});
```

And you can add them at runtime through `testInfo`:

```ts
test('add a todo', async ({ page }, testInfo) => {
  testInfo.annotations.push({ type: 'env', description: process.env.ENV ?? 'local' });
});
```

The difference from Tags: **a Tag is for selective running, an Annotation is for reading in the report.** Don't use a Tag to store a ticket link.

## Controlling the Group

### `test.describe.configure`

```ts
test.describe.configure({ mode: 'serial' });     // in order, stops at the first failure
test.describe.configure({ mode: 'parallel' });   // in parallel even if the file isn't
test.describe.configure({ retries: 2 });         // retries for this group
test.describe.configure({ timeout: 60_000 });    // timeout for the group
```

`mode: 'serial'` solves one real case: tests that depend on each other in order. For example, creating an account and then logging in with it.

> **Warning**
> `serial` breaks the principle of atomic tests. If the first one fails, the rest don't run and you lose information. Use it when there's no other solution, not because it's easier.

### `test.use`

Changes settings for the file or for the group:

```ts
test.use({ locale: 'ar-JO', timezoneId: 'Asia/Amman' });

test.describe('mobile UI', () => {
  test.use({ viewport: { width: 390, height: 844 } });

  test('menu collapses', async ({ page }) => {});
});
```

And its most useful application is skipping login:

```ts
test.use({ storageState: 'playwright/.auth/user.json' });
```

Every test in the file starts already logged in.

## Summary

- `skip` says "the environment isn't suitable", `fixme` says "the test is broken and needs to be revisited". The difference is a message to the team, not technical behavior.
- `test.fail` documents a known Bug in code, and alerts you the day it gets fixed, unlike `skip`, which sleeps for a year.
- `test.only` must be paired with `forbidOnly: !!process.env.CI` in the config; otherwise one day it gets merged and CI comes back green on a single test.
- The conditional syntax must be at the start of the test, before any `await`, and with a written reason.
- Tags are for running (`--grep @smoke`), Annotations are for the report. Three Tags are enough for most teams.
- `describe.configure({ mode: 'serial' })` is a last resort, not a convenience, and `test.use` is the cleanest way to change locale, viewport, or session for an entire file.
