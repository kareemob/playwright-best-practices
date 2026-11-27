# Projects and Multiple Browsers: A Practical Strategy

## Introduction

Projects in the config file let you run the same tests on Chromium, Firefox, and WebKit.

And this is where the most common mistake happens: people think Projects = browsers, enable all three on every Pull Request, and a month later CI takes forty minutes and someone on the team suggests "let's just run them at night", and the whole point is lost.

Projects are much broader than that. And the browser decision on its own has a practical answer, not the one you'd expect.

## Objectives

After reading this, you should be able to:

- Use Projects for four things, not just browsers
- Build a setup Project that logs in once for everyone
- Decide which browsers to run on a PR and which at night
- Defend your decision with a time and cost calculation, not personal taste

## What a Project Actually Is

**A Project is a complete run of the tests with its own settings.**

```ts
projects: [
  {
    name: 'chromium',
    use: { ...devices['Desktop Chrome'] },
  },
]
```

Meaning: take the tests and run them under these conditions. And the conditions include anything in `use`: the browser, the `baseURL`, the language, the screen size, the session, the headers.

So anything you'd want to run your tests under twice with different conditions is a Project. The browser is just one case out of four.

## Use One: Setup Dependency (The Most Powerful)

This is the most valuable use, and it has nothing to do with browsers at all.

**The problem:** fifty tests, and each one logs in first. Fifty times filling a form and waiting for a response: minutes lost on every run.

**The solution:** a Project that logs in once and saves the session, and all the rest start ready.

```ts
projects: [
  {
    name: 'setup',
    testMatch: /.*\.setup\.ts/,
  },
  {
    name: 'chromium',
    use: {
      ...devices['Desktop Chrome'],
      storageState: 'playwright/.auth/user.json',
    },
    dependencies: ['setup'],
  },
]
```

And the setup file:

```ts
import { test as setup } from '@playwright/test';

setup('log in', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill(process.env.TEST_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_PASSWORD!);
  await page.getByRole('button', { name: 'Login' }).click();
  await page.waitForURL('/todos');
  await page.context().storageState({ path: 'playwright/.auth/user.json' });
});
```

`dependencies: ['setup']` means: don't start before setup finishes. And if setup fails, everything else is skipped, which is correct, because without a session there's no point running.

> **Tip**
> The same idea works for cleanup. Define a Project called `cleanup` and set it as the `teardown` of the setup, to delete the data the tests created. That way the test environment stays clean without external scripts.

## Use Two: Environments

The same tests on staging and production, with a single browser:

```ts
projects: [
  {
    name: 'staging',
    use: { baseURL: 'https://staging.todo.example.com' },
  },
  {
    name: 'production',
    use: { baseURL: 'https://todo.example.com' },
    grep: /@smoke/,
  },
]
```

Notice the `grep` on production: on production you run only the critical paths, not everything. A test that creates a hundred fake users on production is a real problem, not an achievement.

```
npx playwright test --project=staging
```

## Use Three: Screen Sizes

```ts
projects: [
  { name: 'desktop', use: { ...devices['Desktop Chrome'] } },
  { name: 'mobile', use: { ...devices['iPhone 13'] } },
  { name: 'tablet', use: { ...devices['iPad Pro 11'] } },
]
```

> **Warning**
> This is emulation, not real devices. Playwright changes the screen size and the user agent and enables touch; it runs on the same Chromium engine on your machine. Excellent for checking Responsive behavior and elements hidden on small screens. Not a substitute for testing on a real device if your product is a mobile app.

## Use Four: Browsers

And now the use everyone starts with:

```ts
projects: [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
  { name: 'webkit', use: { ...devices['Desktop Safari'] } },
]
```

Two lines and you have coverage of three browsers. Very easy, and that's exactly the trap.

### The Calculation Nobody Does

Let's calculate honestly. You have eighty tests, and a single run takes six minutes.

| Setup | CI time | Runs per day | Daily cost |
|---|---|---|---|
| Chromium only | 6 minutes | 20 PRs | 2 hours |
| Three browsers | 18 minutes | 20 PRs | 6 hours |

Three times the time and three times the cost. So what did you buy in return?

In practical experience with modern web apps, most failures on the other browsers aren't real Bugs. They're timing differences, fonts loading at different speeds, or minor WebKit behavior with animations. Meaning you're paying three times the time to fix your tests, not to find Bugs.

And there's a bigger problem than time: **the feedback loop.** A developer waiting 18 minutes will open another tab and move on to another task. The difference between 6 and 18 minutes isn't time; it's whether they stay focused on the PR or not.

> **Important**
> This doesn't mean ignoring the other browsers. It means running them at the right time. Real differences between engines exist, but they're rare and don't happen every hour, so they're not worth checking on every commit.

### The Practical Strategy

**On every Pull Request: Chromium only.**

```
npx playwright test --project=chromium
```

Fast, feedback within minutes, and it catches 90% of breakages.

**Every night: all browsers.**

```
npx playwright test
```

Time has no value at two in the morning. And if something fails, you see it in the morning.

**Before every release: everything + devices.**

All browsers and all screen sizes. This is where the minutes are worth it.

### Implementation in CI

```yaml
jobs:
  pr-tests:
    if: github.event_name == 'pull_request'
    steps:
      - run: npx playwright test --project=chromium

  nightly-tests:
    if: github.event_name == 'schedule'
    steps:
      - run: npx playwright test
```

**When do you break this rule?** When you have a real reason:

- Your product targets Safari heavily → WebKit on every PR
- You have a history of recurring Bugs in a specific browser → add it
- You have twenty tests that finish in a minute → run everything, there's no problem to begin with

The decision is a calculation, not a principle. When you reach tens of minutes, go back to the table.

## Uses Table

| Dimension | Example | When |
|---|---|---|
| Setup | `setup` + `dependencies` | Always: highest return |
| Environment | staging / production | When you test more than one environment |
| Device | desktop / mobile | When the design is Responsive |
| Browser | chromium / firefox / webkit | Chromium always, the rest at night |
| Categorization | `grep: /@smoke/` | On production and fast PRs |

## Summary

- A Project is a run with its own settings, and the browser is one of four uses, not the most important.
- `setup` with `dependencies` is the highest return in the whole file: logging in once instead of fifty times.
- Environment Projects let you run the same tests on staging and production, and with `grep` you limit production to the critical paths.
- Mobile Projects are emulation, not real devices: excellent for Responsive, not a substitute for testing a mobile app.
- Three browsers on every PR multiplies CI time by three, and most of the extra failures are timing differences, not Bugs.
- The strategy: Chromium on every PR, all browsers at night, and everything before a release, unless you have a measured reason that says otherwise.
