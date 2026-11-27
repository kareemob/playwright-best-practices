# Running Playwright on GitHub Actions

## Introduction

Running tests on your own machine means opening the terminal, typing `npx playwright test`, and looking at the result.

That's automation halfway. The real value starts when the tests run without you: with every Pull Request, before any code reaches production, and while you're asleep.

This document is about that last step: turning your project into something that runs on its own on GitHub Actions.

## Objectives

After reading this, you should be able to:

- Write a workflow file that runs your tests with every Pull Request
- Save the report and the Trace and download them when a test fails
- Store passwords securely instead of writing them in the code
- Split the tests across more than one machine to speed them up

## The Simplest Working File

In the repository, create `.github/workflows/playwright.yml`:

```yaml
name: Playwright Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci

      - run: npx playwright install --with-deps

      - run: npx playwright test

      - uses: actions/upload-artifact@v4
        if: ${{ !cancelled() }}
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

That's all. Push it, and on the first Pull Request you'll see the tests run on their own.

Let's walk through the lines that trip people up.

## `npx playwright install --with-deps`

The most forgotten line, and the most common reason the first run fails.

The CI machine is empty: no browsers, and none of the system libraries the browsers need on Linux. `--with-deps` installs both.

Without it you'll see:

```
Host system is missing dependencies to run browsers
```

> **Tip**
> If you only run on chromium, `npx playwright install --with-deps chromium` is much faster: it downloads one browser instead of three, and saves a minute or two on every run.

## `if: ${{ !cancelled() }}` on the Artifact

This is the detail that makes the difference.

By default, any step after a failed step doesn't run. So if the tests fail (and that's exactly the moment you want the report), uploading the report doesn't happen.

`if: ${{ !cancelled() }}` says: run it even if the previous step failed, but not if the workflow was cancelled manually.

Without this line, you only get reports for successful runs, which are the ones you don't need.

## Secrets

You never write a password in the file. GitHub gives you a secure place:

```yaml
      - run: npx playwright test
        env:
          BASE_URL: ${{ secrets.BASE_URL }}
          USER_EMAIL: ${{ secrets.USER_EMAIL }}
          USER_PASSWORD: ${{ secrets.USER_PASSWORD }}
```

You add them from: **Settings → Secrets and variables → Actions → New repository secret**.

And in the code you read them normally:

```ts
await page.getByLabel('Email').fill(process.env.USER_EMAIL!);
```

> **Warning**
> Never write credentials in a file that gets pushed to Git, even if the repository is private. Anyone who joins the team sees the whole history, and deleting the line later doesn't remove it from the old commits.

## Splitting Tests Across Machines: Sharding

When you have hundreds of tests, running them on one machine takes a long time. Playwright supports splitting them:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      # … same steps …
      - run: npx playwright test --shard=${{ matrix.shard }}/4
```

Four machines, each takes a quarter of the tests. Twenty minutes becomes five.

`fail-fast: false` is important: without it, one shard failing cancels the rest, and you lose the results of three quarters of the tests.

### Merging Reports

Each shard produces its own report. To merge them:

```yaml
  merge-reports:
    if: ${{ !cancelled() }}
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true
      - run: npx playwright merge-reports --reporter html ./all-blob-reports
      - uses: actions/upload-artifact@v4
        with:
          name: html-report
          path: playwright-report
```

For this you need the `blob` reporter in the config when sharding:

```ts
reporter: process.env.CI ? 'blob' : 'html',
```

## Multiple Browsers

```yaml
    strategy:
      matrix:
        project: [chromium, firefox, webkit]
    steps:
      - run: npx playwright test --project=${{ matrix.project }}
```

> **Important**
> Don't run three browsers with every Pull Request from day one. Three times the time and three times the flakiness. Start with chromium on the PR, and run all three once at night with `schedule`:
>
> ```yaml
> on:
>   schedule:
>     - cron: '0 2 * * *'
> ```

## Reading the Result

After the run finishes:

1. The **Actions** tab shows you green or red for each commit.
2. The **Summary** has an overview, and under **Artifacts** you'll find `playwright-report`.
3. Download it, unzip it, and open `index.html`: the same report you see locally, with the Traces.
4. Open the Trace with `npx playwright show-trace trace.zip` and see every step exactly.

And if you add the `github` reporter, failures appear as a comment on the line itself inside the Pull Request:

```ts
reporter: process.env.CI ? [['blob'], ['github']] : 'html',
```

## Checklist Before Relying on It

- [ ] `npx playwright install --with-deps` is present
- [ ] `if: ${{ !cancelled() }}` on the artifact upload
- [ ] Secrets in `secrets`, not in the file
- [ ] `forbidOnly: !!process.env.CI` in the config
- [ ] `retries: 2` on CI only
- [ ] `trace: 'on-first-retry'`
- [ ] `timeout-minutes` on the job so it doesn't hang forever
- [ ] `fail-fast: false` if you're using sharding

## Summary

- Automation that only runs on your machine is half automation. The value starts when it runs with every Pull Request without you.
- `npx playwright install --with-deps` is the most forgotten line: the CI machine has no browsers and no system libraries.
- `if: ${{ !cancelled() }}` on the report upload is essential; otherwise you only get a report when the tests pass, which is the moment you don't need it.
- Secrets always go in GitHub Secrets; deleting a line from a file doesn't remove it from Git history.
- Sharding splits the tests across parallel machines, with `fail-fast: false`, the `blob` reporter, and a merge at the end.
- Chromium on the PR, and all three browsers on a nightly schedule. Three times the time on every PR isn't a winning trade.
