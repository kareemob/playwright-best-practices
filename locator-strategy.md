# Locator Strategy: The Order You Should Follow

## Introduction

Locators are among the easiest things to learn in automation. But they're also the number one reason automation collapses.

Anyone who has done automation for a year knows the story: you wrote a hundred tests, they all passed, and after one sprint half of them are red. The application works fine; the developer just changed a `class` or moved an element inside another `div`, and everything broke.

The problem isn't that you wrote a Locator wrong. The problem is that you chose the wrong **type** of Locator. There's a correct order of choice, one Playwright itself recommends, and that's what this document is about.

## Objectives

After reading this, you should be able to:

- Rank Locator types from most stable to most fragile, and know why
- Choose the right Locator the first time instead of trying until it works
- Understand why `getByRole` is the officially recommended first choice
- Know when a CSS Selector is acceptable and when it's a warning sign

## The Question You Should Ask

Before writing any Locator, ask: **"If the developer changes the design tomorrow, will this Locator still work?"**

An example that makes everything clear. You have a button:

```html
<div class="form-actions mt-4">
  <button class="btn btn-primary btn-lg" id="submit-btn-2">
    Log in
  </button>
</div>
```

Five ways to reach it:

```ts
page.locator('.btn.btn-primary.btn-lg')            // very fragile
page.locator('#submit-btn-2')                      // fragile
page.locator('div.form-actions > button')          // very fragile
page.getByRole('button', { name: 'Log in' })       // stable ✅
page.getByTestId('login-submit')                   // stable ✅
```

The first three break when the design changes, and the design changes a lot. The last two keep working, because they're tied to the element's **function**, not its **appearance**.

> **Important**
> A stable Locator describes **what the element does for the user**, not **where it is in the code**. This sentence alone solves 80% of Locator problems.

## The Order: From Most Stable to Most Fragile

### 1. `getByRole`: Always the First Choice

```ts
await page.getByRole('button', { name: 'Log in' }).click();
await page.getByRole('textbox', { name: 'Email' }).fill('a@b.com');
await page.getByRole('heading', { name: 'Todos' }).isVisible();
await page.getByRole('link', { name: 'Home' }).click();
```

Why is it first? For three reasons:

1. **It describes function.** "A button named Log in" stays true however the CSS changes.
2. **It checks accessibility for free.** If `getByRole` can't find the button, it's usually because the element isn't a real button: a `<div onclick>` instead of a `<button>`. So your test uncovers a real accessibility problem.
3. **It's close to the user experience.** The user looks for "the button that says Log in", not for `.btn-primary`.

The most common roles: `button` · `link` · `textbox` · `checkbox` · `radio` · `heading` · `list` · `listitem` · `dialog` · `alert` · `table`.

### 2. `getByLabel`: For Fields

```ts
await page.getByLabel('Password').fill('secret');
```

Tied to the `<label>`, which makes it more stable than the placeholder, because the label rarely changes.

### 3. `getByPlaceholder` and `getByText`

```ts
await page.getByPlaceholder('Type a new todo').fill('Milk');
await page.getByText('No todos yet').isVisible();
```

Good, but text changes more often than a label: any wording edit breaks them. For long texts, use part of the text with `exact: false`.

### 4. `getByTestId`: When There's No Clear Function

```ts
await page.getByTestId('todo-item-3').click();
```

Here you add a `data-testid` in the code. This is the most stable approach there is, because the attribute exists for testing, so nobody will remove it in a design change.

> **Tip**
> If you need a `data-testid` and there isn't one, add it yourself. Don't wait for the developer to add it for you: you're part of the team and you're allowed to edit the code. Two minutes of work saves you hours of maintenance.

### 5. CSS Selector: The Last Choice

```ts
page.locator('.todo-list li:nth-child(3)');
```

Acceptable when the element has no role, no text, and no label, and you can't add a `data-testid`. But consider it technical debt: you'll pay it when the design changes.

### And XPath?

Playwright supports it, but avoid it. `//div[3]/span[2]/button` is the most fragile form possible: it breaks from any element added in between. There's no case in modern automation that needs XPath without a better alternative.

## Decision Table

| Situation | Use |
|---|---|
| Button, link, field, heading | `getByRole` |
| Field with a `<label>` | `getByLabel` |
| Field without a label but with a placeholder | `getByPlaceholder` |
| Static text on the page | `getByText` |
| Element without a clear role (card, row, container) | `getByTestId` |
| None of the above | `locator()` with CSS |
| (never) | XPath |

## Mistakes That Trip People Up

### Relying on Order

```ts
// ❌ Will break as soon as the order changes
page.locator('.todo-item').nth(2);

// ✅ Tie it to real content
page.getByRole('listitem').filter({ hasText: 'Buy milk' });
```

### A Long CSS Chain

```ts
// ❌ Every element in the chain is a breaking point
page.locator('div.container > div.row > div.col > form > button');

// ✅
page.getByRole('button', { name: 'Save' });
```

### A Generic Locator That Finds More Than One Element

```ts
// ❌ Will throw a strict mode violation
page.getByRole('button');

// ✅ Make it specific
page.getByRole('button', { name: 'Delete' });
```

This is exactly what **Strict Mode** is about: Playwright refuses to guess which element you meant, and that's correct behavior, not an annoyance.

## Summary

- Locators are the number one reason automation collapses, and the problem is usually the type of Locator, not its syntax.
- A stable Locator describes the element's function for the user, not its location in the code.
- The order: `getByRole` → `getByLabel` → `getByPlaceholder`/`getByText` → `getByTestId` → CSS, and never XPath.
- `getByRole` checks accessibility for free: if it can't find the button, the element usually isn't a real button, and that's an issue worth opening.
- `data-testid` is the most stable option there is. If it isn't there, add it yourself instead of waiting for the developer.
