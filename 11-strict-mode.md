# Strict Mode: Why the Test Fails When It Finds Two Elements

## Introduction

At some point, you'll write a Locator that looks perfectly fine, run the test, and get a strange error:

```
Error: strict mode violation: locator('button') resolved to 3 elements
```

The natural first reaction: "Fine, just take the first one!" That's exactly what Selenium does, and exactly what Playwright refuses to do.

This document is about why the refusal is the right behavior, not an annoyance.

## Objectives

After reading this, you should be able to:

- Understand what Strict Mode is and why it's enabled by default
- Read the violation message and know what to do with it
- Make the Locator specific the right way instead of silencing the error
- Know the few cases where using `first()` is acceptable

## What Happens in Other Tools

In Selenium:

```java
driver.findElement(By.tagName("button")).click();
```

If there are three buttons, Selenium silently takes the first one. No warning, no error.

The problem is that "the first" isn't guaranteed. It's the first one in DOM order, and DOM order changes when the developer adds an element or when the display order changes. So your test stays green while clicking a button other than the one you meant.

And that's the worst kind of error: **the test passes while checking the wrong thing.**

## Playwright's Position

Playwright says: if your Locator finds more than one element, you haven't described your intent precisely enough. So it throws an error instead of guessing.

```ts
await page.locator('button').click();
// ❌ strict mode violation: resolved to 3 elements
```

The message itself is useful: it counts the elements it found and shows them to you:

```
strict mode violation: locator('button') resolved to 3 elements:
  1) <button class="btn">Add</button>
  2) <button class="btn">Delete</button>
  3) <button class="btn">Save</button>
```

So the error isn't an obstacle; it's a diagnosis. It tells you exactly what's available, and you choose.

> **Important**
> Strict Mode isn't an option you turn on. It's the default behavior for every `page.locator()` and `getBy*`. And that's intentional: an error during development is much cheaper than a test that checks the wrong button for six months.

## The Solutions, in the Right Order

### 1. Specify by Name (the Best)

```ts
// ❌
await page.getByRole('button').click();

// ✅
await page.getByRole('button', { name: 'Delete' }).click();
```

This is the correct solution in 90% of cases. You already know which button you want, so say it.

### 2. Specify by Context: Chaining

When the same button exists in several places:

```ts
// Three cards, each with a "Delete" button
await page
  .getByRole('listitem')
  .filter({ hasText: 'Buy milk' })
  .getByRole('button', { name: 'Delete' })
  .click();
```

Here you said: "the Delete button inside the element that contains the text Buy milk". That's a precise and stable description; it keeps working even if the order of the cards changes.

### 3. `filter` with More Precise Conditions

```ts
await page.getByRole('listitem')
  .filter({ has: page.getByRole('checkbox', { checked: true }) })
  .getByRole('button', { name: 'Delete' })
  .click();
```

"The row that has a checked checkbox": a logical description, not a position.

### 4. `first()` / `last()` / `nth()`: The Last Resort

```ts
await page.getByRole('listitem').first().click();
```

This disables Strict Mode for this Locator. And sometimes it's completely correct:

- "Open the first search result": logically you want the first one
- "The last message in the chat": logically you want the last one

The difference between correct and incorrect use: **is "the first" part of your requirement, or just a way to silence the error?**

> **Warning**
> If you write `.first()` to make the error go away, you've gone back to Selenium's behavior exactly, just manually. The test will pass, and you won't know when it clicks the wrong button.

## A Special Case: Truly Identical Elements

Sometimes there are identical elements and that's normal, like table rows. Here the solution is to check the **group**, not one element from it:

```ts
// How many items are in the list
await expect(page.getByRole('listitem')).toHaveCount(3);

// The text of all items in order
await expect(page.getByRole('listitem')).toHaveText([
  'Buy milk',
  'Pay the bill',
  'Call Ahmad',
]);
```

Notice: `toHaveCount` and `toHaveText` with an array don't throw a strict mode violation, because they already expect multiple elements. Strict Mode applies to actions and to assertions that expect a single element.

## Strict Mode as a Diagnostic Tool

A nice use: when you're not sure how many elements match your description, write the Locator without specifying and see what the message tells you.

```ts
await page.getByRole('button').click();   // fails, and lists all the buttons
```

The failure prints the full list of buttons. Faster than opening DevTools and searching.

But there's a better tool for this: **UI Mode** and its **Locator Picker**. You click on the element and it gives you the recommended Locator ready to use.

## Summary

- Strict Mode throws an error when the Locator matches more than one element, instead of silently taking the first one like Selenium.
- This is intentional and useful: a test that clicks the wrong button while green is much worse than a test that fails clearly.
- The message lists the elements it found, so it's a ready-made diagnosis, not an obstacle.
- The solutions in order: specify by name → specify by context (chaining + filter) → and only then `first()`.
- `first()` is correct when "the first" is actually part of your requirement. If you use it to silence the error, you've gone back to Selenium's behavior, just manually.
- For groups, use `toHaveCount` and `toHaveText` with an array; Strict Mode doesn't apply to them.
