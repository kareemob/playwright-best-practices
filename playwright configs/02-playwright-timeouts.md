# Playwright Timeouts: Which One Wins

## Introduction

Playwright has five different timeouts. Each one covers something different, and some of them override others. The result: you raise a number in one place and the test keeps failing with the same message, because you raised the wrong number.

This document puts them in order.

## Objectives

After reading this, you should be able to:

- List the types of timeout and know what each one covers
- Read a failure message and know exactly which timeout expired
- Know which setting overrides which
- Choose the right number instead of raising all the numbers

## The Five

| Timeout | Covers | Default |
|---|---|---|
| Test timeout | The whole test, from the first line to the last | 30 seconds |
| Expect timeout | A single assertion | 5 seconds |
| Action timeout | A single action (`click`, `fill`) | None, inherits from the test |
| Navigation timeout | `goto` and `waitForURL` | None, inherits from the test |
| Global timeout | The entire run | None |

The first point, and the one that explains most of the confusion: **`actionTimeout` and `navigationTimeout` have no default value.** Meaning that by default, a single `click` can take the test's entire 30 seconds.

## The Order: From Outside In

Think of them as nested boxes:

```
Global timeout  (the entire run)
 └── Test timeout  (a single test)
      ├── Action timeout      (each click / fill)
      ├── Navigation timeout  (each goto)
      └── Expect timeout      (each assertion)
```

**The rule: the smallest one stops first.**

So if `timeout: 30_000` and `actionTimeout: 10_000`, the click fails after 10 seconds, not 30. And the reverse: if `timeout: 5_000` and `actionTimeout: 10_000`, the whole test dies after 5 seconds and the action never gets to finish.

> **Warning**
> The most common wrong setting: `expect.timeout` larger than `timeout`. The assertion can never finish because the test dies before it. Keep the test timeout clearly larger than the sum of what's inside it.

## Read the Message: It Tells You Which One

Each timeout has a different message. Learn to tell them apart:

### Test timeout

```
Test timeout of 30000ms exceeded.
```

The whole test took too long. Usually there are too many steps, or a step that's hanging.

### Expect timeout

```
Timed out 5000ms waiting for expect(locator).toBeVisible()
Locator: getByRole('button', { name: 'Save' })
Expected: visible
Received: hidden
```

This is the most useful message of them all: it tells you the Locator, what was expected, and what actually happened. `Received: hidden` means the element exists but is hidden.

### Action timeout

```
locator.click: Timeout 10000ms exceeded.
Call log:
  - waiting for getByRole('button', { name: 'Save' })
  -   locator resolved to <button disabled>Save</button>
  - element is not enabled
```

The **Call log** is the treasure. Here it tells you literally: I found the button, but it's disabled. So the problem isn't time at all; raising the timeout won't solve anything, because the button will stay disabled.

### Navigation timeout

```
page.goto: Timeout 30000ms exceeded.
navigating to "http://localhost:3000/todos", waiting until "load"
```

The page didn't finish loading. Usually the server isn't running, or there's a hanging request.

## How to Set Them

### In the config: for everything

```ts
export default defineConfig({
  timeout: 30_000,
  expect: { timeout: 5_000 },
  use: {
    actionTimeout: 10_000,
    navigationTimeout: 30_000,
  },
});
```

### For a file or a group

```ts
test.describe('slow tests', () => {
  test.describe.configure({ timeout: 60_000 });
  // …
});
```

### For a single test

```ts
test('upload a large file', async ({ page }) => {
  test.setTimeout(120_000);
  // …
});

// Or: multiply the available time
test('a slow test', async ({ page }) => {
  test.slow();          // ×3
});
```

`test.slow()` is cleaner than a fixed number, because it respects the base setting and multiplies it.

### For a single assertion

```ts
await expect(page.getByText('Saved')).toBeVisible({ timeout: 15_000 });
```

This is the preferred approach when there's a single slow operation. Much better than raising the global `expect.timeout`, because you raised the time only for the case that needs it, and the rest of the tests still fail fast when they break.

## Common Mistakes

**Raising all the numbers when a test fails.** It makes the suite slow and doesn't solve the cause. Read the Call log first; usually the problem isn't time.

**`expect.timeout` larger than `timeout`.** The assertion never finishes.

**Using `waitForTimeout` instead of setting the timeout.** `waitForTimeout(5000)` sleeps five seconds every time. `toBeVisible({ timeout: 5000 })` continues as soon as the element appears. The second is faster and more stable.

**A very small `actionTimeout`.** If you set it to two seconds, any slight slowness breaks tests that used to work. Start at 10 seconds and lower it if you need to.

## Numbers That Work

A reasonable starting point for a typical web project:

```ts
timeout: 30_000,                    // single test
expect: { timeout: 5_000 },         // assertion
use: {
  actionTimeout: 10_000,            // click / fill
  navigationTimeout: 30_000,        // goto
},
```

And on CI you can raise them a little, since shared machines are slower:

```ts
timeout: process.env.CI ? 60_000 : 30_000,
```

> **Tip**
> A long timeout isn't insurance, it's postponement. If a test takes 55 seconds out of 60, it's on the edge of failing and will fail the first day the server slows down. The big number hides the problem until it blows up on CI. Read the Trace and see where the time was actually lost.

## Summary

- There are five timeouts: test, expect, action, navigation, global. `actionTimeout` and `navigationTimeout` have no default; they inherit from the test.
- The smallest one stops first. The common mistake: `expect.timeout` larger than `timeout`, so the assertion never finishes.
- Each one has a distinctive message. Learn to read it instead of guessing.
- The Call log in an action timeout is the treasure: "element is not enabled" means the problem isn't time, and raising the number won't solve anything.
- For a single slow case, use `toBeVisible({ timeout })`; it's better than raising the global setting.
- `test.slow()` is cleaner than a fixed number because it multiplies the base setting.
- A long timeout is postponement, not insurance. A test that takes 55 of 60 seconds is a test about to fail.
