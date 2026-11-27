# Actionability Checks: What Playwright Verifies Before Every Action

## Introduction

`click`, `hover`, `fill`, `check`, `selectOption`, file uploads, and drag and drop all work without a single line of waiting.

The general reason is Auto-waiting (see `../05-auto-waiting-web-first-assertions.md`). This document opens the box and shows exactly what Playwright checks before every action, because when an action fails, the failure message tells you which check stopped it, and that information saves you half an hour of diagnosis.

## Objectives

After reading this, you should be able to:

- List the Actionability checks and know what each one means
- Know which check applies to which action
- Read the Call log and identify which check stopped the action
- Know when to use `force`, and why it's rarely the solution

## The Five Checks

Before any action, Playwright retries until all the required conditions are met:

| Check | Meaning |
|---|---|
| Visible | Has a non-zero size, and isn't `visibility: hidden` |
| Stable | Hasn't moved for two consecutive frames; the animation finished |
| Enabled | Isn't `disabled` and isn't inside a `<fieldset disabled>` |
| Editable | Isn't `readonly` (for fields) |
| Receives events | The click point actually reaches it; nothing is on top of it |

Not every action checks all five. Each one checks what makes sense for it:

| Action | Visible | Stable | Enabled | Editable | Receives events |
|---|---|---|---|---|---|
| `click` `dblclick` `hover` | ✅ | ✅ | ✅ | — | ✅ |
| `fill` | ✅ | — | ✅ | ✅ | — |
| `check` `uncheck` | ✅ | ✅ | ✅ | — | ✅ |
| `selectOption` | ✅ | ✅ | ✅ | — | — |
| `setInputFiles` | — | — | ✅ | — | — |
| `press` `pressSequentially` | ✅ | — | ✅ | — | — |
| `focus` `blur` | — | — | — | — | — |
| `textContent` `getAttribute` | — | — | — | — | — |

Notice two important rows:

1. **`fill` doesn't check Stable or Receives events**, because it writes into the field directly; it doesn't simulate a click at a specific point. That's why `fill` is inherently more stable than `click`.
2. **Reading (`textContent`) checks nothing**, so it returns the text of a hidden element without complaint. That's why you should use `expect(...).toHaveText()` instead of reading and comparing.

## Receives Events: The Check That Saves You the Most

This is the most important one, and the one you won't find in other tools.

Playwright calculates the center point of the element, and asks the browser: which element is actually at this point? If the answer is a different element (a Loader, a transparent Modal, a Cookie banner, a Tooltip), it waits.

In Selenium, the same situation gives you:

```
ElementClickInterceptedException: Element is not clickable at point (450, 300)
```

And you have to write a manual wait every time. Playwright waits on its own.

### The Message When It Doesn't Resolve

```
locator.click: Timeout 10000ms exceeded.
Call log:
  - waiting for getByRole('button', { name: 'Save' })
  -   locator resolved to <button>Save</button>
  - attempting click action
  -   waiting for element to be visible, enabled and stable
  -   element is visible, enabled and stable
  -   scrolling into view if needed
  -   done scrolling
  -   performing click action
  -   <div class="overlay">…</div> intercepts pointer events
```

The last line tells you literally: `<div class="overlay">` is standing in front of the button.

Now you don't raise the timeout; you go see why the overlay didn't disappear. Usually an animation longer than expected, or a hanging network request.

## Read the Call Log: It's the Whole Diagnosis

The Call log shows you exactly where it stopped:

| Last line in the log | Meaning | Solution |
|---|---|---|
| `waiting for locator` | It didn't find the element at all | The Locator is wrong (see `../10-locator-strategy.md`) |
| `element is not visible` | Exists but is hidden | Hasn't appeared yet, or `display:none` |
| `element is not enabled` | Disabled | Wait for the condition that enables it |
| `element is not stable` | Still moving | Long animation |
| `intercepts pointer events` | Something on top of it | Wait for what's on top to disappear |
| `element is not editable` | `readonly` | Is the field locked on purpose? |

> **Tip**
> In the Trace Viewer, click the failed action and look at the DOM snapshot at that moment. You'll usually see the overlay or the Loader with your own eyes in two seconds, instead of reading a log and imagining it (see `../04-ui-mode-trace-viewer-inspector.md`).

## `force: true`: And When to Use It

```ts
await page.getByRole('button', { name: 'Save' }).click({ force: true });
```

`force` skips all the checks and clicks directly.

And here it's worth being blunt: **in 95% of cases, `force` hides a real problem.**

Think about what it means: there's something on top of the button, and you're clicking anyway. Meaning you're testing something a real user can't do, because if they tried, they'd click the overlay, not the button. So your test turned green while testing an impossible scenario.

> **Warning**
> Every `force: true` in your code is an unanswered question. Write a comment next to it explaining why; otherwise, three months later, nobody will know whether it's a necessity or laziness.

### The Few Acceptable Cases

- An element intentionally hidden with CSS but functional, like an `<input type="file">` hidden behind a styled button
- A third-party library that places an invisible layer in its design
- Temporary diagnosis, to confirm the problem really is the blocking

### Alternatives Before `force`

```ts
// Wait for the blocker to disappear
await expect(page.getByTestId('loading-overlay')).toBeHidden();
await page.getByRole('button', { name: 'Save' }).click();

// Close whatever is on top
await page.getByRole('button', { name: 'OK' }).click();   // cookie banner
await page.getByRole('button', { name: 'Save' }).click();

// For a hidden field, use fill, not click
await page.getByLabel('Name').fill('Ahmad');
```

> **Tip**
> The solutions above work when you know when the blocker appears. But sometimes it appears at random times: an ad, a survey, or a banner that pops up at any moment. For that case, Playwright has `page.addLocatorHandler()`: you register a blocker once, and Playwright deals with it on its own every time it appears.

## Useful Options Other Than `force`

```ts
// Click at a specific point inside the element
await locator.click({ position: { x: 10, y: 10 } });

// With a modifier key
await locator.click({ modifiers: ['Control'] });

// Right click
await locator.click({ button: 'right' });

// Try without executing: only checks actionability
await locator.click({ trial: true });
```

`trial: true` is handy for diagnosis: it runs all the checks and returns without clicking. Useful when you want to know whether the element is ready without changing the application's state.

## Summary

- Before every action, Playwright checks: visible, stable, enabled, editable, receives events, and retries until they're met.
- Not every action checks everything. `fill` doesn't check stable or receives events, which is why it's inherently more stable than `click`.
- Reading (`textContent`) checks nothing: it silently returns the text of a hidden element. Use `toHaveText` instead.
- Receives events is the check that solves the Loaders and Modals problem, the one that hurts in other tools.
- The Call log is the diagnosis: the last line tells you which check stopped it, and `intercepts pointer events` names the blocking element.
- `force: true` hides a problem in 95% of cases: it tests a scenario a real user can't perform. Try waiting for the blocker or closing it first.
- `trial: true` runs the checks without executing: a handy diagnostic tool.
