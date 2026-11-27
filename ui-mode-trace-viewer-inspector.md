# UI Mode, Trace Viewer, and Inspector

## Introduction

From the command line you can run a test and read its result in the terminal. But the terminal tells you it **failed**; it doesn't tell you **why**.

And that's exactly where the longest hours of your work go. A test fails on CI, and you're sitting there trying to reproduce it on your machine, adding `console.log`, running it ten times, and you still don't understand.

Playwright gives you three tools that shorten that hour to a minute: **UI Mode**, **Trace Viewer**, and **Inspector**. The three look similar, but each one has a completely different job.

## Objectives

After reading this, you should be able to:

- Choose the right tool of the three depending on the situation you're in
- Read a full Trace file and identify the line that failed and why
- Extract a ready-made Locator from the page without opening DevTools
- Diagnose a test that failed on CI without reproducing it locally

## Quick Overview of the Three

| Tool | Command | When |
|---|---|---|
| UI Mode | `npx playwright test --ui` | While writing and developing |
| Trace Viewer | `npx playwright show-trace trace.zip` | After a failure, especially on CI |
| Inspector | `npx playwright test --debug` | When you need to pause mid-execution |

The rule in one sentence: **you write in UI Mode, you investigate in Trace, you pause in Inspector.**

## UI Mode: Home for Daily Work

```
npx playwright test --ui
```

It opens a window with all your tests on the left and the browser in the middle. Click a test and it runs in front of you, step by step.

What makes it different from just `--headed`:

1. **Watch Mode.** Click the eye icon next to the test. From then on, every time you save the file, the test runs by itself. You write a line and see its result immediately, without going back to the terminal even once.

2. **Time travel.** Every action in the list has a saved DOM snapshot. Hover over the steps and see what the page looked like at each moment. Not an image: a real DOM you can inspect.

3. **Partial runs.** Changed a line in the middle? No need to run from the start.

4. **Filtering.** By name, by Tag, by project, or failures only.

> **Tip**
> If you take one thing from this, make it this: write all your tests in UI Mode. People run from the terminal because that's how they learned, and they waste hours on something UI Mode solves in seconds.

## The Locator Picker: The Hidden Tool

Inside UI Mode there's a **Pick locator** button. Click it, then hover over any element on the page.

Playwright will write the suggested Locator for you, and it respects the recommended priority order: `getByRole` first, then `getByLabel`, and only when the options run out does it fall back to CSS.

```ts
// What the Picker gives you
page.getByRole('button', { name: 'Add Todo' })
```

Copy it with one click. Not because you're lazy, but because the tool reads the **Accessibility Tree** more precisely than your eye can.

> **Warning**
> The Picker is a suggestion, not a verdict. If it suggests `.css-1x7fjq`, that means the element has no clear role or name, and that's a problem in the application, not in the tool. That's where you go and add a `data-testid` yourself.

The same tool exists standalone:

```
npx playwright codegen https://todo.example.com
```

It opens a browser and records your actions as code. Useful for exploration, but don't ship its code as is: codegen generates long tests with no structure.

## Trace Viewer: The Black Box

A Trace is a compressed file containing everything that happened in the test. You enable it from the config:

```ts
use: {
  trace: 'on-first-retry',
}
```

And when a test fails, you open it:

```
npx playwright show-trace test-results/todos-add-todo/trace.zip
```

Or drag the file onto `trace.playwright.dev`; it runs in the browser without installing anything.

### What's Inside the Trace

**The Timeline at the top.** A time bar with sequential screenshots. The first thing you do: look for the moment the page went wrong. You usually spot it with your eyes before reading anything.

**The Actions list on the left.** Every step and its duration. The failed step is red. A step that took 30 seconds is probably your problem, even if it passed.

**The DOM Snapshot in the middle.** This is the real power: an inspectable snapshot of the page at that moment. Open DevTools inside it and check whether the element exists, and what its class actually is.

Click any action and you'll see three states: **Before**, **Action**, and **After**. The Action state highlights the element Playwright tried to interact with, which immediately reveals whether it grabbed the wrong element.

**Network tab.** Every request, its status, and its duration. If `POST /api/todos` returned 500, the problem isn't in the test.

**Console tab.** JavaScript errors from the application itself.

**Source tab.** Your code, with the failing line highlighted.

### A Practical Reading Method

1. Look at the screenshots at the top: where did the page go off track?
2. Click the red step and read the error message
3. Open **Before**: was the page even ready?
4. Open **Network**: did a request fail or get delayed?
5. Open **DOM Snapshot**: does the element exist, but with a different name?

Five steps, two minutes, and you know the cause. Without reproducing anything locally.

## Inspector: For Pausing in the Middle

```
npx playwright test login.spec.ts --debug
```

It opens two windows: the browser, and a control panel with a step-by-step run button. The test pauses at the first line and waits for you.

You can pause at a specific line instead of the beginning:

```ts
await page.pause();
```

This is the cleanest approach: put it where you suspect the problem, and run normally.

When do you actually need the Inspector? When you want to try Locators against the current state of the page: after logging in, after opening a Modal, after filling a form. That's where you pause and use the Picker on the state you've reached.

> **Info**
> `--debug` adjusts the timeouts automatically so they don't expire while you're stopped thinking. So don't be surprised if the test doesn't fail at the expected time while you're in debug mode.

## Which One Should I Choose?

| Situation | Tool |
|---|---|
| Writing a new test | UI Mode |
| Looking for a Locator | Picker inside UI Mode |
| Test failed on CI | Trace Viewer |
| Test fails one time in ten | Trace with `on-first-retry` |
| Want to pause after Login and experiment | `page.pause()` |
| Want to see network requests | Trace → Network |

## Summary

- You write in UI Mode, you investigate in Trace Viewer, you pause in Inspector. Three tools for three different moments.
- UI Mode with Watch Mode shortens the "write, run, see" cycle to seconds. Write all your tests in it.
- The Locator Picker suggests a Locator that respects the correct priority order. If it suggests a strange CSS selector, the problem is in the application, not the tool.
- A Trace contains an inspectable DOM, not images, along with Network, Console, and Source. The Action state shows the element Playwright actually grabbed.
- `trace: 'on-first-retry'` is the right setting: you don't pay the recording cost normally, and you get it when you need it.
- `page.pause()` is cleaner than `--debug` because it stops you where you suspect the problem, not from the first line.
