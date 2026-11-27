# A Complete `playwright.config`: Ready-to-Copy Reference

## Introduction

This gathers every `playwright.config` option into a single file.

Not a new explanation: a ready file you copy and start from, with every line explained next to it. This is what you actually need when you open a new project at ten in the morning and don't have time to remember the name of every option.

## Objectives

After reading this, you should be able to:

- Copy a ready config file and adjust it for your project in minutes
- Know which option affects speed and which affects stability
- Tell the difference between local development settings and CI settings
- Find any option in seconds instead of searching the documentation

## The Complete File

```ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  // ---------- Where the tests are ----------
  testDir: './tests',
  outputDir: './test-results',        // failure screenshots and videos

  // ---------- Parallelism ----------
  fullyParallel: true,                // all tests in parallel, not just files
  workers: process.env.CI ? 2 : undefined,   // undefined = half of the machine's CPU cores
  retries: process.env.CI ? 2 : 0,    // retries on CI only
  forbidOnly: !!process.env.CI,       // prevents merging code that contains test.only

  // ---------- Timeouts ----------
  timeout: 30_000,                    // for the whole single test
  expect: { timeout: 5_000 },         // for each assertion

  // ---------- Reports ----------
  reporter: process.env.CI
    ? [['html', { open: 'never' }], ['github']]
    : [['html', { open: 'on-failure' }], ['list']],

  // ---------- Browser settings for all tests ----------
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',

    // Evidence on failure: the three most important lines in the file
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',

    actionTimeout: 10_000,
    navigationTimeout: 30_000,

    // Environment
    locale: 'ar-JO',
    timezoneId: 'Asia/Amman',
    colorScheme: 'light',
    viewport: { width: 1280, height: 720 },

    // Network
    ignoreHTTPSErrors: true,
    extraHTTPHeaders: { 'x-test-run': 'true' },
  },

  // ---------- Projects ----------
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
    {
      name: 'mobile',
      use: { ...devices['iPhone 13'], storageState: 'playwright/.auth/user.json' },
      dependencies: ['setup'],
    },
  ],

  // ---------- Start the application before the tests ----------
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
  },
});
```

## Explaining the Important Options

### Evidence on Failure: The Most Important Three

```ts
trace: 'on-first-retry',
screenshot: 'only-on-failure',
video: 'retain-on-failure',
```

These save you the most. When a test fails on CI at three in the morning, the Trace gives you the full page at every step: the DOM, the network, and the console. You open it with `npx playwright show-trace`.

`trace` values:

| Value | When |
|---|---|
| `'off'` | You don't need it |
| `'on'` | Every test: slow and takes space |
| `'retain-on-failure'` | Records everything and deletes the passing ones |
| `'on-first-retry'` | Recommended: records only when retrying |

`'on-first-retry'` is the right balance: you don't pay the recording cost on a normal run, and you get it exactly when you need it.

### Retries: A Double-Edged Sword

```ts
retries: process.env.CI ? 2 : 0,
```

On CI it makes sense: the network drops, and shared machines slow down. Locally it's zero on purpose, so you see the failure directly instead of it being hidden.

> **Warning**
> Retries hide Flakiness; they don't fix it. A test that passes on the second attempt is a test with a problem. Playwright marks it `flaky` in the report. Watch that number, and if it grows, investigate the flaky tests instead of raising `retries`.

### `workers` and `fullyParallel`

```ts
fullyParallel: true,
workers: process.env.CI ? 2 : undefined,
```

`fullyParallel: true` runs all tests in parallel, even those in the same file. Without it, only the files run in parallel.

On CI we set `workers` because CI machines are usually weaker; too many workers causes slowness and flakiness. Locally, `undefined` lets Playwright use half the cores.

### `webServer`

```ts
webServer: {
  command: 'npm run dev',
  url: 'http://localhost:3000',
  reuseExistingServer: !process.env.CI,
},
```

It starts the application before the tests and waits until it's ready. `reuseExistingServer: !process.env.CI` means: locally, use the running server if one exists; on CI, always start a new one.

### `projects`

The common use isn't browsers; it's the setup dependency:

```ts
{ name: 'setup', testMatch: /.*\.setup\.ts/ },
{ name: 'chromium', dependencies: ['setup'], use: { storageState: '…' } },
```

That way login happens once, and every test starts already logged in through `storageState`.

## Settings Difference: Local vs CI

| Option | Locally | On CI |
|---|---|---|
| `retries` | 0: see the failure | 2: the network is unstable |
| `workers` | `undefined` | 2 or depending on the machine |
| `trace` | `'on-first-retry'` | `'on-first-retry'` |
| `reporter` | list + html | github + html |
| `forbidOnly` | `false` | `true` |
| `reuseExistingServer` | `true` | `false` |

`forbidOnly` is the most important in the table: it prevents merging code that contains `test.only`. That mistake happens, and it makes CI run a single test and come back green while you think everything is fine.

## Common Mistakes

**A small `timeout` with a large `expect.timeout`.** The `timeout` covers the whole test. If you set it to 5 seconds and `expect.timeout` to 10, the assertion never finishes. Keep the `timeout` much larger.

**Forgetting `baseURL`.** Without it, you write the full URL in every `goto`. With it, you write `await page.goto('/todos')` and change the environment with a single variable.

**`headless: false` in the file.** It slows down CI for no benefit. Use `--headed` from the CLI when diagnosing.

**A small `viewport`.** Elements get hidden behind responsive breakpoints. 1280×720 is a safe starting point.

## Summary

- This file is a ready starting point: copy and adjust it, don't build it from scratch every time.
- The three most important lines are the failure evidence: `trace`, `screenshot`, and `video`. And `trace: 'on-first-retry'` is the right balance.
- `retries` on CI only, and watch the flaky number in the report: retries hide the problem, they don't solve it.
- `fullyParallel: true` parallelizes all tests, not just files; set `workers` on CI because its machines are weaker.
- `forbidOnly: true` on CI prevents the most dangerous mistake: `test.only` gets merged, a single test runs, and CI comes back green.
- Keep the `timeout` much larger than `expect.timeout`, otherwise the assertion won't have time to finish.
