# Playwright Best Practices

A practical, opinionated guide to writing Playwright tests that stay stable, fast, and easy to maintain.

Each article starts from a problem you hit in real automation work. It explains *why* the recommended approach works, shows ❌/✅ code comparisons, and ends with a summary of rules you can apply right away. All examples are in TypeScript.

## Who this is for

- QA and automation engineers starting out with Playwright
- Engineers moving over from Selenium or Cypress
- Teams that want shared rules for locators, waits, fixtures, config, and CI

## Articles

### Fundamentals
| Article | What you'll learn |
|---|---|
| [Selenium vs Cypress vs Playwright](01-selenium-vs-cypress-vs-playwright.md) | How the three tools differ in architecture, and when to choose each one |
| [Automation Project Structure](02-automation-project-structure.md) | Where every file goes: pages, components, fixtures, API clients, and test data |
| [Playwright CLI Commands](03-playwright-cli-commands.md) | The `npx playwright` commands you'll use every day |

### Writing Tests
| Article | What you'll learn |
|---|---|
| [Locator Strategy](10-locator-strategy.md) | The order to choose locators in, from `getByRole` down to CSS, and why XPath never makes the list |
| [Strict Mode](11-strict-mode.md) | Why a locator that matches two elements fails, and how to fix it without `first()` |
| [Auto-waiting and Web-First Assertions](05-auto-waiting-web-first-assertions.md) | Why Playwright tests need no `sleep`, and how retrying assertions work |
| [Waiting and Load States](09-waiting-and-load-states.md) | Load states, why to avoid `networkidle`, and `waitForURL` / `waitForResponse` |
| [Fixtures Instead of Hooks](08-fixtures-instead-of-hooks.md) | Building fixtures that compose, with worker scope and auto fixtures |
| [Organizing the Test Suite](07-organizing-test-suite.md) | When to use `describe`, hooks, and `test.step`, and how big a spec file should get |
| [Test Annotations Reference](06-test-annotations-reference.md) | `skip`, `fixme`, `fail`, `only`, `slow`, tags, and annotations |

### Debugging
| Article | What you'll learn |
|---|---|
| [UI Mode, Trace Viewer, and Inspector](04-ui-mode-trace-viewer-inspector.md) | Which debugging tool to use for which moment |

### Configuration & CI
| Article | What you'll learn |
|---|---|
| [The `playwright.config` Map](playwright%20configs/01-playwright-config-map.md) | The seven groups of config options and the layers they belong to |
| [Complete `playwright.config` Reference](playwright%20configs/04-playwright-config-complete-reference.md) | A config file you can copy, plus how local and CI settings differ |
| [Playwright Timeouts](playwright%20configs/02-playwright-timeouts.md) | The five timeouts, which one wins, and how to read their error messages |
| [Projects and Multiple Browsers](playwright%20configs/03-projects-multi-browser-strategy.md) | Using projects for login setup, environments, screen sizes, and browsers |
| [Running Playwright on GitHub Actions](playwright%20configs/05-playwright-github-actions.md) | A working CI workflow, secrets, sharding, and when to run each browser |

### Actions
| Article | What you'll learn |
|---|---|
| [Actionability Checks](playwright%20actions/01-actionability-checks.md) | What Playwright verifies before every action, how to read the Call log, and why `force: true` is rarely the answer |
| [Dialogs, iFrames, and New Tabs](playwright%20actions/02-dialogs-iframes-new-tabs.md) | The three reasons a correct locator finds nothing, and how to handle each one |
| [Complete Actions Reference](playwright%20actions/03-actions-reference.md) | The syntax and options for every action, plus the mistakes that make tests flaky |

## The rules in one minute

- **Locators:** `getByRole` → `getByLabel` → `getByPlaceholder`/`getByText` → `getByTestId` → CSS. Never XPath.
- **Assertions:** `await expect(locator).toBeVisible()` retries until it passes. `expect(await locator.isVisible())` checks once and causes flaky tests.
- **Waits:** keep `waitForTimeout` and `networkidle` out of finished tests. Wait for the result you actually expect.
- **Setup:** setup that only some tests need goes in a fixture. Setup that every test in a file needs goes in a hook.
- **Config:** use `trace: 'on-first-retry'`, retries on CI only, and `forbidOnly: !!process.env.CI`.
- **CI:** run Chromium on every pull request and all browsers nightly.

## Suggested reading order

If you're new to Playwright, read the Fundamentals section first, then Locator Strategy, Auto-waiting, Waiting and Load States, and Fixtures. Read the Configuration & CI articles once you have tests running.
