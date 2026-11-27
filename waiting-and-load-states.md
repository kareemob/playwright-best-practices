# Waiting and Load States

## Introduction

Auto-waiting means Playwright waits for the element to be ready before every action, on its own.

So why talk about waiting again?

Because Auto-waiting answers one question: "Is this element ready?" There's a second question it doesn't answer: "**Has the page finished its work?**"

That question comes up specifically with navigation. When you call `goto`, click a button that takes you to another page, or wait for an API response to come back before you verify something. That's where you need to be explicit about waiting, and this document is about the right way to do it, and about the way that looks right but is wrong.

## Objectives

After reading this, you should be able to:

- Tell the four load states apart and know when to use each
- Use `waitForURL`, `waitForLoadState`, and `waitForResponse` in the right place
- Understand why `networkidle` isn't recommended for modern applications
- Know when you don't need an explicit wait at all, which is the most common case

## First: The Difference Between the Two Types

| Type | Answers | Who does it |
|---|---|---|
| Auto-waiting | Is the element ready for interaction? | Playwright on its own |
| Load states | Has the page finished loading? | You declare it |

The first is automatic with every `click`, `fill`, and `expect`. The second you request when navigation is part of the test.

And the golden rule: **most of the time you don't need the second.**

## The Four Load States

`page.goto()` takes a `waitUntil` option that defines when the page is considered ready and the call returns:

```ts
await page.goto('/todos', { waitUntil: 'load' });
```

| Value | Returns when | When to use it |
|---|---|---|
| `'commit'` | The server response arrived and loading started | The fastest: when you want to check something that happens during loading |
| `'domcontentloaded'` | The HTML was parsed and built | Pages that rely on JS for content |
| `'load'` | All resources loaded (images, styles) | The default |
| `'networkidle'` | No network requests for half a second | ⚠️ Not recommended |

### `'load'`: The Default, and Right Most of the Time

```ts
await page.goto('/todos');   // = waitUntil: 'load'
```

It waits for the browser's `load` event: the HTML, the images, and the CSS have all arrived. Always start here, and only change it when you have a reason.

### `'domcontentloaded'`: When Images Don't Matter to You

```ts
await page.goto('/dashboard', { waitUntil: 'domcontentloaded' });
```

It waits only for the HTML to be built, without waiting for images and fonts. Noticeably faster on image-heavy pages.

Useful when the application is a SPA that renders its content with JavaScript anyway: the images you'd be waiting for usually aren't what you're checking.

### `'commit'`: The Fastest

```ts
await page.goto('/report', { waitUntil: 'commit' });
await expect(page.getByText('Loading')).toBeVisible();
```

It returns as soon as the server response arrives. Its only real use: when you want to check something that appears during loading, such as a loading screen, a skeleton, or a progress bar. If you waited for `load`, they'd be gone.

### `networkidle`: The Neat Trap

```ts
// ❌ Playwright itself says not to use it
await page.goto('/todos', { waitUntil: 'networkidle' });
```

It waits until there have been no network requests for 500 milliseconds. It looks ideal: "wait until everything is done". And that's exactly its problem.

**Reason one: modern applications never go quiet.** An open WebSocket, polling every second, lazy-loaded images, analytics tracking. Any of these prevents the network from becoming "idle", so the wait continues until the timeout ends and the test fails while everything is working fine.

**Reason two: slow on purpose.** Even when it succeeds, you paid at least an extra half second on every navigation. Multiply that by a hundred tests.

**Reason three, and the most important: it waits for the wrong thing.** You don't care about the network; you care that the todo list appears. So wait for exactly that:

```ts
// ❌ Indirect and fragile waiting
await page.goto('/todos', { waitUntil: 'networkidle' });
expect(await page.getByTestId('todo-item').count()).toBe(3);

// ✅ Waiting for the condition you actually care about
await page.goto('/todos');
await expect(page.getByTestId('todo-item')).toHaveCount(3);
```

> **Important**
> The second line isn't a shortcut for the first; it's more correct. The Web-First Assertion retries until the count is three, regardless of the network state. Wait for the result you're checking, not the means of reaching it.

## Useful Explicit Waits

### `waitForURL`: After Navigation

```ts
await page.getByRole('button', { name: 'Login' }).click();
await page.waitForURL('**/todos');
await expect(page.getByRole('heading', { name: 'My Todos' })).toBeVisible();
```

It waits until the URL matches the pattern. Especially useful when Login takes time, or when there's a multi-step redirect.

And you can give it the same `waitUntil` option:

```ts
await page.waitForURL('**/todos', { waitUntil: 'domcontentloaded' });
```

> **Tip**
> `expect(page).toHaveURL('**/todos')` does almost the same thing and throws a clearer error message because it's an assertion. Use `waitForURL` when you're waiting in order to continue, and `toHaveURL` when you're verifying that the navigation happened correctly.

### `waitForLoadState`: After an Action That Changed the Page

```ts
await page.getByRole('link', { name: 'Reports' }).click();
await page.waitForLoadState('domcontentloaded');
```

The same states as `goto`, but after the navigation instead of before it. You use it when the navigation happened through a click, not through `goto`.

### `waitForResponse`: The Most Precise Tool

```ts
const responsePromise = page.waitForResponse('**/api/todos');
await page.getByRole('button', { name: 'Add' }).click();
const response = await responsePromise;

expect(response.status()).toBe(201);
```

**Pay attention to the order:** you define the wait **before** the click. If you reverse it, the response will have arrived before you start waiting, and it hangs until the timeout.

And you can filter with a function:

```ts
const responsePromise = page.waitForResponse(
  (res) => res.url().includes('/api/todos') && res.request().method() === 'POST'
);
```

This is the most powerful wait there is, because it waits for an exact, specific event: not time, and not a general state.

## Decision Tree

| Situation | Solution |
|---|---|
| I opened a page and want to check an element | Normal `goto` + `expect().toBeVisible()` |
| I want to check a loading screen | `waitUntil: 'commit'` |
| A heavy page with images I don't care about | `waitUntil: 'domcontentloaded'` |
| I clicked a button and moved to a page | `waitForURL('**/todos')` |
| I want to confirm the navigation is correct | `expect(page).toHaveURL(...)` |
| I want to check the API response | `waitForResponse` before the click |
| I want to wait for the network to calm down | No: wait for the element instead |
| I don't know what to wait for | Look for the condition, and don't sleep |

## The Practical Summary in One Example

```ts
test('adding a todo shows it in the list', async ({ page }) => {
  await page.goto('/todos');                        // load: the default

  const responsePromise = page.waitForResponse('**/api/todos');
  await page.getByPlaceholder('New todo').fill('Buy milk');
  await page.getByRole('button', { name: 'Add' }).click();
  const response = await responsePromise;

  expect(response.status()).toBe(201);
  await expect(page.getByText('Buy milk')).toBeVisible();  // auto-retry
});
```

No `networkidle`, no `waitForTimeout`. Every wait here is tied to a real condition.

## Summary

- Auto-waiting answers "Is the element ready?", and Load states answer "Has the page finished?". The first is automatic; the second you request.
- `goto` states: `commit` for checking loading screens, `domcontentloaded` for heavy pages, and **`load`** is the default and right most of the time.
- `networkidle` isn't recommended: modern applications never go quiet, it adds half a second to every navigation, and it waits for the means, not the result.
- Instead of `networkidle`, wait for what you're actually checking: `await expect(locator).toHaveCount(3)` retries on its own.
- `waitForURL` for navigation after a click, and `toHaveURL` when you're verifying, not waiting.
- `waitForResponse` is defined before the action, not after it, and it's the most precise wait there is because it's tied to a specific event.
