# The `playwright.config` Map

## Introduction

Once you have working tests, the next file to understand is the one that controls all of them: `playwright.config.ts`.

This file isn't just "settings". It's the difference between a set of tests that run on your machine, and an automation project that runs on CI in parallel, produces a report, records a Trace when it fails, and runs against four environments with a single variable.

Let's draw the map before diving into the details.

## Objectives

After reading this, you should be able to:

- Read a full `playwright.config` file and know what each section of it does
- Tell the difference between global options, `use` options, and `projects`
- Know which group of options affects speed, which affects stability, and which affects diagnostics
- Know where to look for any setting without opening the documentation

## First: The Overall Structure

The file consists of three layers, and understanding the layers matters more than memorizing the options:

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // 1️⃣ Global options: about the test runner itself
  testDir: './tests',
  fullyParallel: true,
  retries: 2,
  timeout: 30_000,
  reporter: 'html',

  // 2️⃣ use: about the browser and the page
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    locale: 'ar-JO',
  },

  // 3️⃣ projects: multiple versions of the run
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],
});
```

> **Important**
> The difference between global options and **`use`** confuses a lot of people, and it's simple: global options tell Playwright **how to manage the run** (how many processes, how many retries, what report). `use` tells it **how to prepare the browser and the page** (which URL, which language, which screen size). A practical sign: anything that can change from one project to another goes in `use`.

## Group One: Test Discovery

The question: where are the tests, and which of them will run?

```ts
testDir: './tests',
testMatch: /.*\.spec\.ts/,
testIgnore: '**/wip/**',
outputDir: './test-results',
```

These are the simplest options in the file. `testDir` sets the folder, and `outputDir` is where failure artifacts are saved: screenshots, videos, and Traces.

**Affects:** organization. A change here doesn't break anything, but it saves time.

## Group Two: Parallelism and Retries

The question: how fast should it go, and what happens when it fails?

```ts
fullyParallel: true,
workers: process.env.CI ? 2 : undefined,
retries: process.env.CI ? 2 : 0,
forbidOnly: !!process.env.CI,
maxFailures: process.env.CI ? 10 : 0,
```

This is the biggest speed gain in the entire file. `fullyParallel` parallelizes all tests, even those in the same file, not just the files.

**Affects:** speed and stability, and the two pull against each other. Too many `workers` on a weak machine produces Flakiness.

> **Warning**
> `retries` hide the problem; they don't solve it. A test that passes on the second attempt is a test with a defect, and Playwright marks it `flaky` in the report. Watch that number instead of raising `retries` every time something goes red.

## Group Three: Timeouts

The question: how long do you wait before giving up?

```ts
timeout: 30_000,              // the whole single test
expect: { timeout: 5_000 },   // each assertion on its own
use: {
  actionTimeout: 10_000,      // each click / fill
  navigationTimeout: 30_000,  // each goto
},
```

Four different timeouts, with a priority order between them that decides which one wins over which. This point trips up a lot of people and deserves its own deep dive.

**The short rule:** the global `timeout` must be much larger than everything inside it. If the test is 5 seconds and the assertion is 10, the assertion will never get the chance to finish.

**Affects:** stability and how long it takes to detect a failure.

## Group Four: Reports

The question: how will you see the result?

```ts
reporter: process.env.CI
  ? [['html', { open: 'never' }], ['github']]
  : [['html', { open: 'on-failure' }], ['list']],
```

`list` for the terminal during development, `html` for the full report, `github` so errors appear directly inside the Pull Request. There's also `json` and `junit` for external systems.

You can run more than one at the same time, and that's what you do in practice.

**Affects:** how fast you understand a failure, not how fast the run is.

## Group Five: Browser Environment (The Biggest Group)

All of these live inside `use`, because they simulate a real user:

```ts
use: {
  baseURL: 'https://todo.example.com',

  // Identity and location
  locale: 'ar-JO',
  timezoneId: 'Asia/Amman',
  geolocation: { latitude: 31.95, longitude: 35.91 },
  permissions: ['geolocation', 'notifications'],

  // Display
  viewport: { width: 1280, height: 720 },
  colorScheme: 'dark',

  // Network
  ignoreHTTPSErrors: true,
  extraHTTPHeaders: { 'x-test-run': 'true' },
  httpCredentials: { username: 'qa', password: 'secret' },

  // Evidence on failure
  trace: 'on-first-retry',
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
}
```

**`baseURL`** is the first one you'll use. Without it, you write the full URL in every `goto`; with it, you write `page.goto('/todos')` and switch environments with a single variable.

And **`trace`**, **`screenshot`**, and **`video`** are the three most important lines in the entire file. They're what let you understand a failure that happened on CI at three in the morning without reproducing it.

> **Tip**
> `locale` and `timezoneId` aren't a luxury for an Arabic app. An app that displays dates will behave differently under `ar-JO` than under `en-US`, and a test that checks "today" breaks when CI runs on UTC time. Pin them from day one.

## Group Six: Projects

The question: how many versions of the same tests do you want to run, and under what conditions?

```ts
projects: [
  { name: 'setup', testMatch: /.*\.setup\.ts/ },
  {
    name: 'chromium',
    use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
    dependencies: ['setup'],
  },
  { name: 'mobile', use: { ...devices['iPhone 13'] } },
],
```

Most people think Projects are only for browsers. Not true: their most powerful use is **`dependencies`**: one project logs in once, and all the rest start already logged in.

**Affects:** coverage, speed, and the structure of the whole project.

## Group Seven: Starting the Application

```ts
webServer: {
  command: 'npm run dev',
  url: 'http://localhost:3000',
  reuseExistingServer: !process.env.CI,
  timeout: 120_000,
},
```

Playwright starts your application and waits until it's ready before the first test. Without it, someone has to start the server manually, and that gets forgotten and breaks CI.

## The Map

| Group | Options | Affects |
|---|---|---|
| Discovery | `testDir` · `testMatch` · `outputDir` | Organization |
| Parallelism | `fullyParallel` · `workers` · `retries` · `forbidOnly` | Speed and stability |
| Timeouts | `timeout` · `expect` · `actionTimeout` | Stability |
| Reports | `reporter` | Understanding failures |
| Environment | `baseURL` · `locale` · `viewport` · `trace` … | Realism and diagnostics |
| Projects | `projects` · `dependencies` | Coverage and structure |
| Startup | `webServer` | Full automation |

Remember this table, not the options. You'll find any option in a second once you know which group to look in.

## Summary

- `playwright.config.ts` is the brain of the project: the difference between tests on your machine and a real automation project.
- The file has three layers: global options to manage the run, `use` to prepare the browser, `projects` for versions of the run.
- Any option that can change from one project to another belongs in `use`. That's the easiest way to tell them apart.
- The parallelism group gives you the biggest speed gain, and the timeouts group the biggest stability gain.
- `trace`, `screenshot`, and `video` are the three most important lines in the file because they explain a CI failure without reproducing it.
- Projects aren't only for browsers: their most powerful use is `dependencies` to log in once for everyone.
