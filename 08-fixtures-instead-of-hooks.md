# Fixtures Instead of Hooks: The Modern Way in Playwright

## Introduction

Hooks (`beforeEach`, `afterEach`, `beforeAll`, and `afterAll`) work perfectly well, and they exist in almost every testing tool.

But Playwright has something else, and it's the way Playwright itself works internally. Every time you write `async ({ page }) => {}`, that `page` between the braces is a **Fixture**. It didn't come from nowhere: Playwright created a browser, a context, and a new page for you, handed it to you, and cleans it up after you.

This document is about how to create your own Fixtures, and why they're better than Hooks in most cases.

## Objectives

After reading this, you should be able to:

- Explain what a Fixture is and how it differs from a Hook
- Write your own Fixture and use it in your tests
- Know when a Fixture is better and when a Hook remains the right choice
- Build a clean test base with no duplicated setup

## The Problem with Hooks

Let's look at the problem before the solution:

```ts
// ❌ Duplicated setup, and variables hanging outside
test.describe('Todos', () => {
  let user: User;
  let todosPage: TodosPage;

  test.beforeEach(async ({ page, request }) => {
    user = await authClient.registerRandomUser(request);
    todosPage = new TodosPage(page);
    await todosPage.goto();
  });

  test('add a todo', async () => {
    await todosPage.add('Milk');
  });
});
```

It works. But it has three problems:

1. **`user` and `todosPage` are declared outside and filled inside.** TypeScript can't guarantee they were filled; in the editor's view their type could be `undefined`.
2. **You can't reuse it.** Want the same setup in another file? Copy the whole `beforeEach`.
3. **It runs for every test in the `describe`**, even the tests that don't need it.

## The Fixture: The Idea

A Fixture is something that gets **prepared on demand** and **handed to the test as a parameter**.

The differences from a Hook:

| | Hook | Fixture |
|---|---|---|
| Runs | For every test in the scope | Only for the tests that request it |
| Reaches you | Through an outside variable | Directly as a parameter |
| Reuse | Copy and paste | `import` |
| Type | Manual | Inferred automatically |
| Composition | Nested | A fixture uses a fixture |

## Your First Fixture

```ts
// fixtures.ts
import { test as base } from '@playwright/test';
import { TodosPage } from './pages/TodosPage';

type MyFixtures = {
  todosPage: TodosPage;
};

export const test = base.extend<MyFixtures>({
  todosPage: async ({ page }, use) => {
    const todosPage = new TodosPage(page);
    await todosPage.goto();
    await use(todosPage);          // ← the test runs here
  },
});

export { expect } from '@playwright/test';
```

The shape looks strange the first time, so let's break it down:

- **`base.extend<MyFixtures>`**: we take Playwright's `test` and add to it.
- **The function takes `({ page }, use)`**: you can request other fixtures (like `page`), and `use` is how you hand over the value.
- **Everything before `use` is setup. Everything after it is cleanup.** The test itself runs at the moment of `use`.

And the usage:

```ts
import { test, expect } from './fixtures';

test('add a todo', async ({ todosPage }) => {
  await todosPage.add('Milk');
  await expect(todosPage.item('Milk')).toBeVisible();
});
```

Notice: no `beforeEach`, no outside variable, and the type is known. Most importantly, a test that doesn't request `todosPage` doesn't pay the cost of preparing it.

## A Fixture That Uses a Fixture

This is where the power shows:

```ts
type MyFixtures = {
  user: User;
  todosPage: TodosPage;
};

export const test = base.extend<MyFixtures>({
  // A new user for every test, via the API
  user: async ({ request }, use) => {
    const user = await authClient.registerRandomUser(request);
    await use(user);
    await authClient.deleteUser(request, user);      // cleanup
  },

  // Depends on user above
  todosPage: async ({ page, user }, use) => {
    await loginPage.loginAs(page, user);
    const todosPage = new TodosPage(page);
    await todosPage.goto();
    await use(todosPage);
  },
});
```

And now:

```ts
test('add a todo', async ({ todosPage }) => {
  await todosPage.add('Milk');
});
```

One line in the test, and behind it: a new user via the API, login, opening the page, and deleting the user when finished. Playwright resolves the dependency chain on its own.

> **Important**
> See how this applies the **Atomic** and **Clean** principles: every test gets its own user, and cleanup is tied to the fixture, not to an `afterEach` that might be forgotten. Isolation becomes part of the structure, not personal discipline.

## Scope: Once per Worker

By default, a fixture runs for every test. Sometimes you want it once for the whole run:

```ts
export const test = base.extend<{}, { adminToken: string }>({
  adminToken: [async ({}, use) => {
    const token = await authClient.loginAsAdmin();
    await use(token);
  }, { scope: 'worker' }],
});
```

`scope: 'worker'` means: once per worker, not per test. Useful for something expensive that doesn't change, like an admin token.

> **Warning**
> Anything you share between tests with worker scope breaks Atomic if the tests modify it. Use it for read-only data or for something truly constant.

## The Auto Fixture

If you want something to run for every test without requesting it:

```ts
export const test = base.extend<{ logFailures: void }>({
  logFailures: [async ({}, use, testInfo) => {
    await use();
    if (testInfo.status !== testInfo.expectedStatus) {
      console.log(`Failed: ${testInfo.title}`);
    }
  }, { auto: true }],
});
```

`auto: true` makes it always run. This is the closest thing to `beforeEach`, but reusable through `import`.

## When Should You Still Use Hooks?

Fixtures aren't an absolute replacement. A Hook is clearer when:

- **The setup is specific to one file** and you won't reuse it; `beforeEach` is shorter and more explicit.
- **You want an explicit order** of steps within the same file.
- **The team isn't used to Fixtures yet**; code that's clear to the team matters more than elegant code.

**The practical rule:** setup repeated across more than one file → Fixture. Setup specific to one file → Hook.

## Summary

- The `page` you use in every test is a Fixture; Playwright prepares it and cleans it up for you.
- A Fixture runs only for the tests that request it, arrives as a parameter, and its type is known, unlike a Hook with an outside variable.
- The shape: `base.extend`, with setup before `use` and cleanup after it.
- Fixtures compose: a fixture requests a fixture, and Playwright resolves the chain. One line in the test hides a complete setup behind it.
- This is how isolation becomes part of the structure: a new user and cleanup tied to the fixture, not to personal discipline.
- `scope: 'worker'` for something expensive and constant, and `auto: true` for something that must always run.
- Setup repeated across files → Fixture. Setup specific to one file → Hook.
