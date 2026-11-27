# Playwright Best Practices — Knowledge Base

This folder is a **shared knowledge base** on Playwright best practices, written by the owner and maintained together with Claude.

## How Claude should use it

1. **It's the source of truth.** When answering a Playwright question or writing Playwright code with the owner, check the relevant article first and follow its rules. Cite it as `file.md` (section) so the owner can trace the reasoning.
2. **Keep it consistent.** If an answer or a new article would contradict an existing one, point out the conflict. Don't quietly pick a side. Then update the older article or flag it.
3. **Grow it, don't duplicate it.** Before writing a new article, check whether the topic belongs in an existing one. Add cross-links rather than re-explaining.
4. **Keep the index current.** Whenever an article is added, renamed, or gets a new key rule, update the index and "Core rules" below, and the article tables in `README.md`. Links in the README use `%20` for folder names with a space (`playwright%20configs/`, `playwright%20actions/`).
5. **Verify against current Playwright.** If an article looks outdated against current Playwright docs or behavior, say so and propose an edit.

## Index

### Fundamentals
| File | Topic |
|---|---|
| `01-selenium-vs-cypress-vs-playwright.md` | Architecture differences and when to choose each tool |
| `02-automation-project-structure.md` | Folder layout: `tests/`, `pages/`, `components/`, `fixtures/`, `clients/`, `models/`, `data/` |
| `03-playwright-cli-commands.md` | Everyday `npx playwright` commands and quick reference |

### Writing tests
| File | Topic |
|---|---|
| `10-locator-strategy.md` | Locator priority order and common locator mistakes |
| `11-strict-mode.md` | Strict mode violations, how to narrow a locator, when `first()` is acceptable |
| `05-auto-waiting-web-first-assertions.md` | Auto-waiting checks, retrying vs value assertions, why no `sleep` |
| `09-waiting-and-load-states.md` | Load states, `networkidle`, `waitForURL`, `waitForResponse`, decision tree |
| `08-fixtures-instead-of-hooks.md` | `base.extend`, fixture composition, `scope: 'worker'`, `auto: true` |
| `07-organizing-test-suite.md` | `describe` depth, hook vs fixture, `test.step`, spec file size, test titles |
| `06-test-annotations-reference.md` | `skip`/`fixme`/`fail`/`only`/`slow`, tags vs annotations, `serial`, `test.use` |

### Debugging
| File | Topic |
|---|---|
| `04-ui-mode-trace-viewer-inspector.md` | UI Mode, Locator Picker, Trace Viewer, Inspector, `page.pause()` |

### Configuration & CI (`playwright configs/`)
| File | Topic |
|---|---|
| `playwright configs/01-playwright-config-map.md` | The 7 option groups of `playwright.config.ts` and its three layers |
| `playwright configs/04-playwright-config-complete-reference.md` | A complete config you can copy, plus local vs CI settings |
| `playwright configs/02-playwright-timeouts.md` | The 5 timeouts, which one wins, how to read the error messages |
| `playwright configs/03-projects-multi-browser-strategy.md` | Projects for setup dependency, environments, viewports, browsers |
| `playwright configs/05-playwright-github-actions.md` | GitHub Actions workflow, secrets, sharding, multi-browser schedule |

### Actions (`playwright actions/`)
| File | Topic |
|---|---|
| `playwright actions/01-actionability-checks.md` | The 5 actionability checks, which action runs which, reading the Call log, `force` vs alternatives, `trial: true` |
| `playwright actions/02-dialogs-iframes-new-tabs.md` | Dialogs (auto-dismissed, `page.on`/`once`), `frameLocator`, new tabs with `waitForEvent('page')`, diagnosis table |
| `playwright actions/03-actions-reference.md` | Syntax and options for every action, key names, `selectOption`, `setInputFiles`, `dragTo` fallback, shared options |

## Core rules (quick reference, details in the articles)

**Locators** (`10-locator-strategy.md`)
- Order: `getByRole` → `getByLabel` → `getByPlaceholder`/`getByText` → `getByTestId` → CSS. Never XPath.
- Describe what the element *does for the user*, not where it sits in the DOM. Use `.filter({ hasText })` instead of `.nth()`.
- If a `data-testid` is missing, add it to the app code yourself.
- A strict mode violation means the locator is vague. Narrow it by name, then by context (chaining + `filter`). Use `first()`/`nth()` only when "the first" is the actual requirement (`11-strict-mode.md`).

**Waiting & assertions** (`05-auto-waiting-web-first-assertions.md`, `09-waiting-and-load-states.md`)
- `await expect(locator).toBeVisible()` retries. `expect(await locator.isVisible())` checks one moment only and becomes flaky.
- No `waitForTimeout` in final code. No `networkidle`. Wait for the real result instead (`toHaveCount`, `toHaveURL`).
- `waitForURL` after a click that navigates. Set up `waitForResponse` *before* the action that triggers it.

**Structure & organization** (`02-automation-project-structure.md`, `07-organizing-test-suite.md`, `08-fixtures-instead-of-hooks.md`)
- No CSS selectors inside spec files. A Page Object takes `page`, a Component takes a `Locator`.
- Setup that some tests need goes in a fixture. Setup that every test in one file needs goes in a hook.
- Use at most 1–2 levels of `describe`. A spec file holds 5–12 tests, split by feature. A test title should explain the failure on its own.
- Use `test.step` only for long multi-phase flows, with titles in the user's language.
- `.env` and `playwright/.auth/` go in `.gitignore`. `.env.example` is committed.

**Actions** (`playwright actions/`)
- Before an action, Playwright checks visible, stable, enabled, editable and receives events. `fill` skips stable and receives events, so it is more stable than `click`.
- Read the last line of the Call log before raising a timeout. `intercepts pointer events` names the blocking element.
- Avoid `force: true`. Wait for the blocker to disappear or close it first. If `force` is truly needed, add a comment that explains why.
- Playwright dismisses dialogs automatically. Register `page.on('dialog')` or `page.once('dialog')` before the click that triggers it.
- Elements inside an iframe need `frameLocator`. A new tab needs `context().waitForEvent('page')`, with the promise prepared before the click. If you only need the link target, check the `href` instead.
- Use `fill` for typing, `check`/`uncheck` for checkboxes (they verify the result), and `pressSequentially` only for fields that react to each keystroke. `selectOption` works only on a native `<select>`, and `setInputFiles` must target the `input[type=file]` itself.

**Annotations** (`06-test-annotations-reference.md`)
- `skip` means the environment isn't suitable. `fixme` means the test is broken. `fail` marks a known bug.
- `test.only` must always be paired with `forbidOnly: !!process.env.CI`.
- Conditional annotations go at the start of the test, before any `await`, with a written reason. About 3 tags is enough.

**Config & timeouts** (`playwright configs/`)
- `trace: 'on-first-retry'`. Turn on `retries` only on CI, and watch the flaky count.
- `fullyParallel: true`, and limit `workers` on CI.
- `timeout` must be much larger than `expect.timeout`. For one slow case, prefer `{ timeout }` on that call or `test.slow()` over raising the global value.
- Read the Call log before raising a timeout. "element is not enabled" is not a timing problem.
- Log in once with a `setup` project plus `dependencies`. Run Chromium on every PR and all browsers nightly.
- On CI, use `install --with-deps`, upload the report with `if: ${{ !cancelled() }}`, and keep secrets in GitHub Secrets.

## Article template (for new articles)

Every article follows this shape. Keep it.

```
# Title: Subtitle

## Introduction      the real-world pain, in plain words
## Objectives        "After reading this, you should be able to:" + 3–5 bullets
## ...body...        concrete code, ❌/✅ comparisons, decision tables
## Summary           5–7 bullets, each a standalone rule
```

Style:
- English, practical, and opinionated. Explain *why*, not only *what*.
- Use TypeScript code blocks.
- Use `> **Important**` / `> **Tip**` callouts.
- Use decision tables for "when to use what" questions.
- File names are kebab-case `.md`, prefixed with a two-digit order number (`01-`, `02-`, ...). Numbering is separate for the main folder and for each subfolder. Config and CI topics go in `playwright configs/`. Action topics go in `playwright actions/`.
