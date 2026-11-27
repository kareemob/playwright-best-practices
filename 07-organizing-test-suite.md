# Organizing the Test Suite: When to Use `describe`, Hooks, and Steps

## Introduction

Playwright gives you a full set of tools for writing tests: `describe`, Hooks, Steps, Fixtures, Tags, and Annotations.

The next problem isn't "how do I use them". It's **when to use each one**.

Each of these solves a real problem, and each becomes a burden when used in the wrong place. `describe` inside `describe` inside `describe`. A Hook that prepares something half the tests don't need. `test.step` on every line. Mess in automation rarely comes from a lack of tools; it comes from using them without a decision.

This document is about decisions. Four questions, and a clear answer for each.

## Objectives

After reading this, you should be able to:

- Decide when to group with `describe` and when to leave it out
- Choose between a Hook and a Fixture with a single question
- Know when `test.step` is worth it and when it's noise
- Determine a reasonable spec file size and when to split it

## Question One: When to Use `describe`?

`test.describe` does three things: it groups visually in the report, it defines the scope of Hooks, and it allows `configure` and `use` on a group.

**Use it when:**

- You have more than three tests in the same file on the same feature
- You want a Hook or `use` to apply to only part of the file
- You want a Tag on a whole group

**Don't use it when:**

- The file has two or three tests on the same feature anyway; the file name is enough
- You're adding it only "because that's what people do"

```ts
// ❌ Useless describe: the file is already named login.spec.ts
test.describe('Login', () => {
  test('successful login', async ({ page }) => {});
});

// ✅ No extra wrapping
test('successful login', async ({ page }) => {});
```

### The Nesting Rule

**One level. Two at most if there's a reason.**

```ts
// ❌ Three levels: nobody will find anything
test.describe('Todos', () => {
  test.describe('Adding', () => {
    test.describe('Invalid cases', () => {

// ✅ A separate file is clearer
// tests/e2e/todos-validation.spec.ts
```

If you feel you need a third level, you need a new file, not a new `describe`.

## Question Two: Hook or Fixture?

This is the most important decision here, and the question that settles it:

**Does every test in this scope need this setup?**

- Yes, all of them → a Hook is acceptable
- No, only some → Fixture

Why? Because a Hook runs **by force**. A Fixture runs **only when the test asks for it**.

```ts
// ❌ Hook: every test logs in, even the login screen test itself
test.beforeEach(async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('Test1234');
  await page.getByRole('button', { name: 'Login' }).click();
});

// ✅ Fixture: runs only for the tests that ask for it
export const test = base.extend<{ authedPage: Page }>({
  authedPage: async ({ page }, use) => {
    await login(page);
    await use(page);
  },
});

test('add a todo', async ({ authedPage }) => { /* logged in */ });
test('error message with wrong password', async ({ page }) => { /* not logged in */ });
```

The second difference: **readability**. With a Fixture, the test's signature tells you what it needs. With a Hook, you have to scroll up and see what happened before it.

### Where Hooks Are Still Right

| Situation | Tool |
|---|---|
| Setup that literally every test in the file needs | `beforeEach` |
| General cleanup (clearing data, closing a connection) | `afterEach` / `afterAll` |
| Heavy one-time setup (seeding a database) | `beforeAll` |
| Setup that some tests need | Fixture |
| An object you build in every test (Page Object) | Fixture |
| Data specific to each test | Fixture |

> **Warning**
> `beforeAll` shares state between tests, and that breaks isolation. A test that changes something prepared by `beforeAll` corrupts the ones after it, and the failure shows up in an innocent test. If you use it, keep it read-only.

## Question Three: When to Use `test.step`?

`test.step` groups lines under a title that appears in the report and the Trace.

**It's worth it when:**

- The test has more than one logical phase: setup, action, verification on a long flow
- The test is read by a non-technical person in the report
- You want to know which phase failed without opening the code

```ts
// ✅ Three clear phases on a long flow
test('purchase a subscription', async ({ page }) => {
  await test.step('Log in', async () => { /* ... */ });
  await test.step('Choose a plan', async () => { /* ... */ });
  await test.step('Complete payment', async () => { /* ... */ });
});

// ❌ A step for every line: noise, not organization
test('add a todo', async ({ page }) => {
  await test.step('Type the title', async () => {
    await page.getByPlaceholder('New todo').fill('Milk');
  });
  await test.step('Click add', async () => {
    await page.getByRole('button', { name: 'Add' }).click();
  });
});
```

**The practical rule:** a step should cover more than two lines, and its title should be in the user's language, not the code's language. "Complete payment" is a step; "Click the Submit button" isn't.

And a five-line test doesn't need Steps at all; its structure is clear at a glance.

## Question Four: How Big Should a Spec File Get?

There's no official number, but there are practical indicators:

| Indicator | Reasonable limit |
|---|---|
| Number of tests in the file | 5 to 12 |
| Lines in the file | Under 200 after Page Objects |
| `describe` levels | 1, and 2 at most |
| File run time | Under two minutes |

The real signal isn't the number. It's when you start scrolling to find a test.

### How to Split

**Split by feature, not by page:**

```
tests/e2e/
├── login.spec.ts            # successful and failed login
├── register.spec.ts         # account creation and verification
├── todos-create.spec.ts     # adding todos
├── todos-delete.spec.ts     # deleting todos
└── todos-validation.spec.ts # invalid cases
```

Notice: todos was split into three files because the operations are different, not because the page is big.

### Naming

- **Files:** kebab-case ending in `.spec.ts`, e.g. `todos-create.spec.ts`
- **Page Objects:** PascalCase, e.g. `TodosPage.ts`
- **Fixtures:** `test-fixtures.ts`
- **Test titles:** a full sentence in the user's language: "user can delete a todo", not "delete test 2"

> **Tip**
> A quick test for a test title: if it appeared on its own in a failure report, would you know what broke in the product? If the answer is no, the title is weak. "todo test 3" says nothing; "deleting a todo removes it from the list immediately" says everything.

## The Decisions in One Table

| Situation | Decision |
|---|---|
| Two tests on the same feature | No `describe` |
| Six tests on the same feature | One `describe` |
| I want a third `describe` level | New file |
| Setup that everyone needs | `beforeEach` |
| Setup that some need | Fixture |
| Page Object | Always a Fixture |
| A three-phase flow | `test.step` |
| A five-line test | No Steps |
| File exceeded 12 tests | Split it by feature |

## Summary

- `describe` is for grouping when there are more than three tests on the same feature. One level, and two at most.
- If you need a third level, you need a file, not a `describe`.
- The Hook vs Fixture question is one: does every test need this setup? All of them → Hook. Some → Fixture.
- A Fixture runs on demand, and makes the test's signature say what it needs without scrolling up.
- `test.step` is worth it on long flows with more than one phase, with its title in the user's language. A step per line is noise.
- A reasonable file is 5 to 12 tests, split by feature, not by page. And a test title should explain the breakage on its own in a failure report.
