# Complete Actions Reference

## Introduction

`click`, `dblclick`, `hover`, `blur`, `fill`, `press`, Checkboxes, `selectOption`, file uploads, and Drag and Drop: this document gathers them all in one place.

It's a reference. Not something you read once and forget: it's the page you come back to every time you write an Action and don't remember the syntax or the right Option.

Bookmark it.

## Objectives

After reading this, you should be able to:

- Find the syntax of any Action in seconds without opening the documentation
- Know which Options are available for each Action and when to use them
- Distinguish the Actions that need special handling from the rest
- Avoid the common mistakes that make a Test Flaky

## Quick Table

| Action | Syntax | When |
|---|---|---|
| `click` | `locator.click()` | Normal click: the most used |
| `dblclick` | `locator.dblclick()` | Two consecutive clicks |
| `hover` | `locator.hover()` | Showing a menu or Tooltip |
| `fill` | `locator.fill('text')` | Filling a field: the fastest and most stable |
| `clear` | `locator.clear()` | Emptying a field |
| `press` | `locator.press('Enter')` | A single Keyboard key |
| `pressSequentially` | `locator.pressSequentially('text')` | Typing character by character |
| `check` / `uncheck` | `locator.check()` | Checkbox or Radio |
| `selectOption` | `locator.selectOption('jo')` | Native `<select>` only |
| `setInputFiles` | `locator.setInputFiles('cv.pdf')` | Uploading files |
| `dragTo` | `source.dragTo(target)` | Drag and drop |
| `focus` / `blur` | `locator.focus()` | Focusing and leaving focus |
| `scrollIntoViewIfNeeded` | `locator.scrollIntoViewIfNeeded()` | Rarely: see below |

## Clicking: `click` and Its Siblings

```ts
await page.getByRole('button', { name: 'Save' }).click();
await page.getByRole('button', { name: 'Save' }).dblclick();
await page.getByRole('link', { name: 'Profile' }).hover();
```

### `click` Options

```ts
await locator.click({ button: 'right' });              // right mouse button
await locator.click({ button: 'middle' });             // middle button
await locator.click({ clickCount: 3 });                // triple click: selects the line
await locator.click({ modifiers: ['Control'] });       // Ctrl + click
await locator.click({ modifiers: ['Shift', 'Alt'] });  // more than one key
await locator.click({ position: { x: 10, y: 5 } });    // a specific point inside the element
await locator.click({ timeout: 10_000 });              // a custom timeout for this line
await locator.click({ trial: true });                  // check without actually clicking
await locator.click({ force: true });                  // ❌ skip the checks
```

> **Tip**
> `trial: true` is the best Option nobody uses. Playwright performs all the Actionability checks and then doesn't click. Useful when you want to confirm the button is actually ready without changing the application's state (see `01-actionability-checks.md`).

`position` is useful with Canvas and maps.

## Typing: `fill` and `clear`

```ts
await page.getByLabel('Email').fill('ahmad@test.com');
await page.getByLabel('Email').clear();                 // same as fill('')
```

`fill` empties the field and fills it in one go, sending a single `input` event. This is always the default approach.

### `press`: A Single Key

```ts
await page.getByLabel('Search').press('Enter');
await page.getByLabel('Name').press('Tab');
await page.keyboard.press('Escape');
```

Key name syntax:

| Key | Syntax |
|---|---|
| Enter | `Enter` |
| Navigation | `Tab` · `Shift+Tab` |
| Cancel | `Escape` |
| Arrows | `ArrowDown` · `ArrowUp` · `ArrowLeft` · `ArrowRight` |
| Delete | `Backspace` · `Delete` |
| Select all | `Control+A` (or `Meta+A` on macOS) |
| Pages | `PageDown` · `Home` · `End` |
| Single character | `a` · `A` · `$` |

### `pressSequentially`: Character by Character

```ts
await page.getByPlaceholder('Search for a city')
          .pressSequentially('Amman', { delay: 100 });
```

This sends `keydown`, `keypress`, and `keyup` for each character. Use it only when the field reacts to every keystroke, like a search field that opens a suggestions list.

> **Info**
> `pressSequentially` replaced the old `type`. `type` still works but is deprecated, because its name suggested it was the natural way to type, and the natural way is `fill`.

## Selection: Checkbox, Radio, and Select

```ts
await page.getByLabel('Remember me').check();
await page.getByLabel('Remember me').uncheck();
await page.getByRole('radio', { name: 'Male' }).check();
```

`check` verifies the result afterwards. If the element stays unchecked, the Action fails; it doesn't pass silently like `click`.

### `selectOption`

```ts
await select.selectOption('jo');                        // by value
await select.selectOption({ label: 'Jordan' });         // by visible text
await select.selectOption({ index: 2 });                // by position
await select.selectOption(['jo', 'eg', 'sa']);          // multiple selection
await select.selectOption([]);                          // clear all selections
```

> **Warning**
> `selectOption` only works with a native `<select>`. Most dropdowns on modern sites are `<div>`s styled with CSS, and with those you have to `click` the trigger and then `click` the option. If `selectOption` gives you the error "Element is not a select element", that's why.

## File Uploads: `setInputFiles`

```ts
await page.getByLabel('Resume').setInputFiles('tests/files/cv.pdf');
await page.getByLabel('Photos').setInputFiles(['a.png', 'b.png']);   // multiple
await page.getByLabel('Resume').setInputFiles([]);                    // clear
```

And you can upload a file from memory without it existing on disk:

```ts
await page.getByLabel('File').setInputFiles({
  name: 'todos.csv',
  mimeType: 'text/csv',
  buffer: Buffer.from('Buy milk,false'),
});
```

> **Warning**
> `setInputFiles` must target the `<input type="file">` itself, not the styled button you see. The button is usually a `<button>` or `<label>`, and the real input is hidden behind it. If you can't find it with a regular Locator, look for it with `page.locator('input[type=file]')`.

## Drag and Drop: `dragTo`

```ts
await page.getByText('First task').dragTo(page.getByTestId('done-column'));
```

And if the application doesn't respond, walk through the steps manually:

```ts
const source = page.getByText('First task');
const target = page.getByTestId('done-column');

await source.hover();
await page.mouse.down();
await target.hover();
await target.hover();          // twice, so the movement registers
await page.mouse.up();
```

> **Warning**
> `dragTo` is the Action most prone to Flakiness in the entire library, because every Drag and Drop library listens to different events. If it becomes unstable for you, the manual approach above works in most cases.

## Focus and Scrolling

```ts
await page.getByLabel('Email').focus();
await page.getByLabel('Email').blur();          // useful for triggering validation
await page.getByText('Terms').scrollIntoViewIfNeeded();
```

> **Important**
> You almost never need `scrollIntoViewIfNeeded`. Every Action scrolls automatically before interacting with the element, as part of the Actionability checks. Use it only with something that happens on scroll itself, like Lazy Loading.

## Shared Options

You can pass these to most Actions:

| Option | What it does | Warning |
|---|---|---|
| `timeout` | A custom timeout for this line (ms) | Zero means no timeout: dangerous |
| `force` | Disables all Actionability checks | Last resort, with a comment explaining why |
| `noWaitAfter` | Doesn't wait for Navigation after the Action | When the click starts a file download |
| `trial` | Checks readiness without executing | Excellent for diagnosis |

> **Warning**
> `force: true` isn't a solution; it hides the problem. When you need it, it means the element isn't actually ready: covered by an overlay, or still moving with a CSS transition. The Test will pass for you and fail on CI. Review `../05-auto-waiting-web-first-assertions.md` and `01-actionability-checks.md` before writing it.

## Common Mistakes

**❌ Using `pressSequentially` for every field.** It slows down the Test for no benefit. `fill` is the default.

**❌ `click` on an element instead of `check`.** `check` verifies the result, and `click` doesn't. If the click didn't work, `click` still passes successfully.

**❌ `force: true` to silence an error.** You're disabling the very protection you chose Playwright for.

**❌ `waitForTimeout` before the Action.** Auto-waiting does the work, and a fixed wait slows things down without solving anything.

**✅ The right default:** `fill` for typing, `click` for clicking, `check` for selecting, and no Options at all. Use Options only when you have a reason you can write in a comment.

## Summary

- `fill` is the default for typing, and `pressSequentially` only when the field reacts to every character, like search and suggestion fields.
- `check` and `uncheck` are safer than `click` on Checkboxes, because they verify the result after clicking.
- `selectOption` works with a native `<select>` only. Styled dropdowns need two clicks: on the trigger and on the option.
- `setInputFiles` must target the `input[type=file]` itself, not the visible button hiding it.
- `dragTo` is the most prone to Flakiness. When it fails, walk through the steps manually with the mouse, hovering the target twice.
- `force: true` disables the Actionability checks: use it as a last resort with a comment explaining why, because it hides the problem instead of solving it.
- You rarely need `scrollIntoViewIfNeeded`, because every Action scrolls automatically anyway.
