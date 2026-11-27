# Auto-waiting and Web-First Assertions: Why There's No `sleep` in Playwright

## Introduction

If you come from the Selenium world, there's probably something that bothers you when you look at a Playwright test: where are the waits?

In Selenium, the first thing you learn after the basics is waiting: implicit and explicit waits, `WebDriverWait`, and `ExpectedConditions`. Without them, your code breaks every day.

In Playwright, a typical test has no wait lines at all. And it works.

That's not because Playwright is "faster". It's because there are two mechanisms that work on their own: **Auto-waiting** and **Web-First Assertions**. Understanding them is the difference between someone who writes stable tests and someone who sprinkles `waitForTimeout` every time a test fails.

## Objectives

After reading this, you should be able to:

- Explain what Playwright does before every action without you asking it to
- Tell the difference between a Web-First Assertion and a regular Assertion
- Know why `waitForTimeout` is the last resort, not the first
- Diagnose a test that fails on timing and know where to start

## The Problem It Solves

The browser isn't instant. When you click a button, several things happen that take time:

- A request to the server comes back after 200 milliseconds
- An element gets added to the page after the response arrives
- An animation takes 300 milliseconds until the button settles in place

Code runs in microseconds. So without waiting, you try to click a button that hasn't arrived yet, or is still moving.

The traditional solution: you write the waiting yourself. Playwright's solution: the tool does it on its own.

## Level One: Auto-waiting

Before any action (`click`, `fill`, `check`, `selectOption`), Playwright verifies a chain of conditions. It only executes when all of them are met:

| Condition | Meaning |
|---|---|
| Attached | The element actually exists in the DOM |
| Visible | It has a size, and isn't `display: none` or `visibility: hidden` |
| Stable | It stopped moving; the animation finished |
| Enabled | It isn't `disabled` |
| Receives events | No other element is on top of it, blocking it |

So when you write:

```ts
await page.getByRole('button', { name: 'Login' }).click();
```

You're actually saying: "Wait until this button exists, is visible, stops moving, becomes clickable, and has nothing on top of it, then click it." All of that in one line.

> **Important**
> The **Receives events** condition is the one that saves you the most, and it's the one you won't find in other tools. If there's a transparent Modal over the button, a Cookie banner, or a Loader, Selenium will click and fail with `ElementClickInterceptedException`. Playwright will wait until whatever is on top disappears.

The waiting is tied to `actionTimeout` in the config, which by default inherits from the global timeout.

## Level Two: Web-First Assertions

Auto-waiting covers actions. But what about verification?

This is the difference that trips people up:

```ts
// ❌ Regular Assertion: checks once and that's it
const text = await page.getByTestId('counter').textContent();
expect(text).toBe('1');

// ✅ Web-First Assertion: retries until it passes or the time runs out
await expect(page.getByTestId('counter')).toHaveText('1');
```

The core difference:

- The **first** reads the text at this exact moment. If the number hasn't updated yet, it fails immediately.
- The **second** keeps re-reading every ~100 milliseconds until it becomes `1` or the `expect` timeout runs out.

**How to tell them apart at a glance:** a Web-First Assertion starts with `await expect(...)` and takes a **Locator** inside it. A regular Assertion takes a **value**, meaning you did an `await` before it to fetch the value.

```ts
await expect(locator).toBeVisible();          // ✅ locator -> retries
expect(await locator.isVisible()).toBe(true); // ❌ value   -> one moment
```

The second line works sometimes and fails sometimes. And that is exactly what a **Flaky Test** is.

### Assertions That Retry

`toBeVisible` · `toBeHidden` · `toBeEnabled` · `toBeDisabled` · `toBeChecked` · `toHaveText` · `toContainText` · `toHaveValue` · `toHaveCount` · `toHaveAttribute` · `toHaveClass` · `toHaveURL` · `toHaveTitle`

They all take a Locator and they all retry. Use them instead of fetching the value and checking it yourself.

## So When Do You Need an Explicit Wait?

Auto-waiting covers "the element is ready". It doesn't cover "the page finished its work". There are cases where you need to be explicit:

```ts
// After a click that navigates to another page
await page.getByRole('button', { name: 'Login' }).click();
await page.waitForURL('**/todos');

// When waiting for a specific API response
const responsePromise = page.waitForResponse('**/api/todos');
await page.getByRole('button', { name: 'Save' }).click();
await responsePromise;
```

These are **conditional waits**: they wait for a specific event, not for time.

## And `waitForTimeout`?

```ts
// ❌ Don't do this
await page.waitForTimeout(3000);
```

This sleeps for three seconds regardless of anything. The problems:

- **Always slow:** even when the page is ready after 200 milliseconds, it still sleeps 3 seconds. Multiply that by a hundred tests.
- **And it still fails:** the day the server slows down to 4 seconds, it fails. So you paid for the slowness and didn't buy stability.

> **Warning**
> `waitForTimeout` is acceptable in one case: during diagnosis, when you're trying to understand what's happening. In code that runs on CI, its presence is a sign that there's a real condition you haven't found yet. Look for the condition instead of sleeping.

## The Full Picture

```ts
test('adding a todo shows it in the list', async ({ page }) => {
  await page.goto('/todos');

  // auto-waiting: waits for the field to exist, be visible, and be stable
  await page.getByPlaceholder('New todo').fill('Buy milk');
  await page.getByRole('button', { name: 'Add' }).click();

  // web-first: retries until the element appears
  await expect(page.getByText('Buy milk')).toBeVisible();
  await expect(page.getByTestId('count')).toHaveText('1');
});
```

Not a single explicit wait line, and the test is stable. Not because of luck, but because every line waits for the right condition on its own.

## Summary

- Playwright performs Auto-waiting before every action: attached, visible, stable, enabled, and nothing on top blocking it.
- The **receives events** condition is what solves the Modals and Loaders problem that hurts in other tools.
- A Web-First Assertion (`await expect(locator).toBeVisible()`) retries; an Assertion on a value (`expect(await locator.isVisible())`) checks one moment and becomes Flaky.
- The visual rule: `expect` with a Locator inside = retries. With a value inside = doesn't.
- Explicit waits for events (`waitForURL`, `waitForResponse`) are acceptable and useful. `waitForTimeout` is slow and still fails; its presence in final code means there's a real condition you haven't found yet.
