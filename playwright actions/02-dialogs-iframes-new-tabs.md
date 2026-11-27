# Dialogs, iFrames, and New Tabs

## Introduction

Most actions deal with elements inside the page. But real applications have three situations that fall outside that:

1. **A confirmation window from the browser**: "Are you sure you want to delete?"
2. **An element inside an iframe**: a payment gateway, a text editor, a map
3. **A new tab** that opens when you click a link

All three stop beginners, because they write a perfectly correct Locator and get "element not found". The reason is that they're looking in the wrong place.

## Objectives

After reading this, you should be able to:

- Handle `alert`, `confirm`, and `prompt`
- Reach any element inside an iframe
- Handle a new tab or window
- Diagnose "element not found" and know which of the three is the cause

## First: Dialogs

These are the browser's own windows, not HTML:

```js
alert('Saved');
confirm('Are you sure?');
prompt('Enter your name');
```

You can't find them with a Locator, because they're not part of the page.

### The Default Behavior

**Playwright dismisses them automatically.** Meaning `confirm` returns `false` and the page continues.

This is intentional: without it, any dialog would freeze the test until the timeout ends.

> **Important**
> That's why you might find a test that "clicks delete" and the item doesn't get deleted. The dialog appeared and Playwright dismissed it, so no deletion happened. The solution isn't waiting; the solution is handling the dialog explicitly.

### Explicit Handling

```ts
// Accept
page.on('dialog', (dialog) => dialog.accept());
await page.getByRole('button', { name: 'Delete' }).click();

// Dismiss
page.on('dialog', (dialog) => dialog.dismiss());

// Accept with text (for prompt)
page.on('dialog', (dialog) => dialog.accept('Ahmad'));
```

### Check Its Content

```ts
page.on('dialog', async (dialog) => {
  expect(dialog.type()).toBe('confirm');
  expect(dialog.message()).toBe('Are you sure you want to delete?');
  await dialog.accept();
});

await page.getByRole('button', { name: 'Delete' }).click();
await expect(page.getByText('Buy milk')).toBeHidden();
```

> **Warning**
> Register the listener **before** the click, not after it. The dialog appears immediately with the click; if you register after it, you miss the event and the test hangs.

### Only Once

```ts
page.once('dialog', (dialog) => dialog.accept());
```

`once` removes the listener after its first use. Useful when you expect only one dialog, so it doesn't affect the rest of the test.

## Second: iframes

An iframe is a whole page inside a page. Regular Locators can't see inside it, because it's actually a separate document.

```ts
// ❌ Finds nothing
await page.getByLabel('Card number').fill('4242424242424242');

// ✅
await page
  .frameLocator('#payment-frame')
  .getByLabel('Card number')
  .fill('4242424242424242');
```

### Ways to Select the Frame

```ts
page.frameLocator('#payment-frame')                    // by id
page.frameLocator('iframe[name="checkout"]')           // by name
page.frameLocator('iframe[src*="stripe"]')             // by part of the URL
page.frameLocator('iframe').first()                    // the first one
```

### A Frame Inside a Frame

```ts
await page
  .frameLocator('#outer')
  .frameLocator('#inner')
  .getByRole('button', { name: 'Confirm' })
  .click();
```

> **Tip**
> `frameLocator` keeps all the advantages of a regular Locator: auto-waiting, strict mode, and `filter`. So once you're inside the frame, everything else works as it is (see `../10-locator-strategy.md` and `../11-strict-mode.md`).

### How Do You Know the Problem Is an iframe?

Open DevTools, right-click the element → **Inspect**. If you find an `<iframe>` between the element and `<body>` in the DOM tree, that's it.

The same goes for the Trace Viewer: the DOM snapshot shows you the full structure (see `../04-ui-mode-trace-viewer-inspector.md`).

## Third: New Tabs and Windows

When you click a link with `target="_blank"`, a new tab opens. Your `page` still points to the old one.

```ts
// ❌ Still on the old page
await page.getByRole('link', { name: 'Terms' }).click();
await expect(page.getByRole('heading', { name: 'Terms' })).toBeVisible();
```

### The Correct Form

```ts
const pagePromise = page.context().waitForEvent('page');
await page.getByRole('link', { name: 'Terms' }).click();
const newPage = await pagePromise;

await newPage.waitForLoadState();
await expect(newPage.getByRole('heading', { name: 'Terms' })).toBeVisible();
await newPage.close();
```

**Order matters:** prepare the Promise **before** the click. If you wait after it, you might miss the event. Same logic as the dialog, and as `waitForResponse` (see `../09-waiting-and-load-states.md`).

### More Than One Tab

```ts
const pages = page.context().pages();
console.log(`Number of tabs: ${pages.length}`);
await pages[1].bringToFront();
```

### Avoid It When You Can

Sometimes you don't need the tab at all:

```ts
// Check that the link is correct instead of opening it
const link = page.getByRole('link', { name: 'Terms' });
await expect(link).toHaveAttribute('href', '/terms');
await expect(link).toHaveAttribute('target', '_blank');
```

If what you're checking is that the link goes to the right place, this is faster and more stable than opening a tab.

## Diagnosis Table

"The Locator is correct but the element isn't found": go through these:

| Symptom | Likely cause | Solution |
|---|---|---|
| The click happened but nothing changed | A dialog was dismissed | `page.on('dialog', …)` |
| The element is visible in the browser but not found | Inside an iframe | `frameLocator` |
| The test is on the old page after the click | A new tab | `waitForEvent('page')` |
| Works headed and fails headless | Timing or screen size | Check the `viewport` and the Trace |

## Summary

- Dialogs aren't part of the page, so you can't find them with a Locator. Playwright dismisses them automatically, and that's a common cause of "I clicked delete and nothing was deleted".
- Use `page.on('dialog', …)` or `page.once(...)`, and register the listener before the click.
- An iframe is a separate document: use `frameLocator`, and after that all the Locator advantages work normally.
- A new tab needs `waitForEvent('page')`, with the Promise prepared before the click.
- If all you're checking is the link's destination, check the `href` instead of opening a tab.
- All three give the same symptom, "element not found", so go through the diagnosis table instead of raising the wait time.
