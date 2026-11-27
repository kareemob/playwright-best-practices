# Playwright Command Line: Common Commands

The most commonly used commands when working with Playwright day to day.

## `npx playwright test`

Runs all the test cases in all the test files. By default, it executes all the projects in **headless** mode.

## `npx playwright test --ui`

Opens **UI Mode**: an interactive window where you can browse the test files, pick which tests to run, watch each step as it executes, and inspect the DOM snapshot, actions, network calls, and console logs for every step. Tests are not run automatically; you choose what to run from the window.

## `npx playwright test --project=chromium`

Runs the tests only on Desktop Chrome. All the projects are defined and managed in the `playwright.config.ts` file.

## `npx playwright test example`

Runs the tests in a specific file. The argument is matched against file names, so `example` runs any test file whose name contains "example" (for example `example.spec.ts`).

## `npx playwright test --debug`

Runs the tests in debug mode.

## `npx playwright codegen`

Auto-generates tests with Codegen.

## `npx playwright test --headed`

Runs the tests in **headed** mode, meaning the browser window is visible while the tests run.

## `npx playwright show-report`

Opens the Playwright HTML report.

## `npx playwright test --grep "@smoke"`

Runs all tests with a specific tag.

## `npx playwright test --grep "@smoke|@auth"`

Runs all tests that have the `@smoke` tag **or** the `@auth` tag.

## `npx playwright test --grep-invert "@smoke"`

Runs all tests **except** those with a specific tag.

## Quick Reference

| Command | What it does |
|---|---|
| `npx playwright test` | Run all tests, all projects, headless |
| `npx playwright test --ui` | Open the interactive UI Mode |
| `npx playwright test --project=chromium` | Run tests on Desktop Chrome only |
| `npx playwright test example` | Run tests in files matching "example" |
| `npx playwright test --debug` | Run tests in debug mode |
| `npx playwright codegen` | Generate tests by recording actions |
| `npx playwright test --headed` | Run tests with the browser visible |
| `npx playwright show-report` | Open the HTML report |
| `npx playwright test --grep "@smoke"` | Run tests with a specific tag |
| `npx playwright test --grep "@smoke\|@auth"` | Run tests with either tag |
| `npx playwright test --grep-invert "@smoke"` | Run all tests except a specific tag |
