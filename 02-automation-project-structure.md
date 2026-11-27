# Automation Project Structure: Where Every File Goes

## Introduction

A new Playwright project starts with a `tests` folder containing two files, and that's it. That's how every project begins.

The problem shows up three months later. You now have sixty tests, the Locators are duplicated across ten files, and every time a button changes you're opening files whose names you don't remember. An automation project doesn't collapse because of the code, it collapses because of the mess.

This document is about structure. These aren't cosmetic rules: every folder here prevents a specific kind of pain you'll feel later.

## Objectives

After reading this, you should be able to:

- Build a folder structure for a real automation project from day one
- Know where every file goes and why it goes there specifically
- Tell the difference between a Page, a Component, a Client, and a Model
- Protect the project's secrets from being committed to Git

## The Full Structure

This is what a mature Playwright project looks like for a Todo application:

```
playwright-todo/
├── tests/
│   ├── e2e/
│   │   ├── login.spec.ts
│   │   ├── register.spec.ts
│   │   └── todos.spec.ts
│   └── api/
│       ├── auth.spec.ts
│       └── todos.spec.ts
├── pages/
│   ├── LoginPage.ts
│   ├── RegisterPage.ts
│   └── TodosPage.ts
├── components/
│   ├── Header.ts
│   └── TodoItem.ts
├── fixtures/
│   └── test-fixtures.ts
├── clients/
│   ├── AuthClient.ts
│   └── TodoClient.ts
├── models/
│   ├── User.ts
│   └── Todo.ts
├── data/
│   └── users.ts
├── playwright.config.ts
├── .env
├── .env.example
├── .gitignore
└── package.json
```

Looks like a lot? Take it folder by folder. Each one answers a single question.

## `tests/`: What Describes the Behavior

The spec files go here, and only here. A file in this folder should be readable and understandable by a product manager.

```ts
test('user can add a new todo', async ({ loginPage, todosPage }) => {
  await loginPage.goto();
  await loginPage.login('user@example.com', 'Test1234');
  await todosPage.addTodo('Buy milk');
  await expect(todosPage.todoByText('Buy milk')).toBeVisible();
});
```

No Locators, no `page.click`, no waiting. If you see a CSS Selector in a spec file, that's a sign something is in the wrong place.

Splitting into `e2e/` and `api/` helps when you want to run only one type: `npx playwright test tests/api`.

## `pages/`: The Whole Page

Every page in the application gets a Class. Inside it are the page's Locators and actions.

```ts
export class LoginPage {
  constructor(private page: Page) {}

  private emailInput = () => this.page.getByLabel('Email');
  private passwordInput = () => this.page.getByLabel('Password');
  private submitButton = () => this.page.getByRole('button', { name: 'Login' });

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput().fill(email);
    await this.passwordInput().fill(password);
    await this.submitButton().click();
  }
}
```

The rule: the page knows where the buttons are, the test knows what it wants to test.

## `components/`: What Repeats Across Pages

This is the folder most people forget, and it's the one that saves the most.

The Header exists on every page. A single Todo row repeats fifty times on the same page. These aren't pages, they're repeating pieces.

```ts
export class TodoItem {
  constructor(private root: Locator) {}

  async delete() {
    await this.root.getByRole('button', { name: 'Delete' }).click();
  }

  async toggle() {
    await this.root.getByRole('checkbox').check();
  }
}
```

Notice that the Component takes a `Locator`, not a `page`. That's because it's part of a page, not a page. This exact point is what separates someone who understands POM from someone who imitates it.

## `fixtures/`: Setup Before the Test

A Fixture prepares something and hands it to the test ready to use. Instead of writing `new LoginPage(page)` in every file:

```ts
export const test = base.extend<{ loginPage: LoginPage; todosPage: TodosPage }>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  todosPage: async ({ page }, use) => {
    await use(new TodosPage(page));
  },
});
```

Then in the test: `async ({ loginPage })` and that's it. This is where Fixtures become a real replacement for Hooks.

## `clients/` and `models/`: The API World

These belong to the API testing layer, but their place exists from day one.

The **Client** knows how to talk to the API: the URLs, the Headers, the Token.

```ts
export class TodoClient {
  constructor(private request: APIRequestContext) {}

  async create(title: string, token: string) {
    return this.request.post('/api/todos', {
      data: { title },
      headers: { Authorization: `Bearer ${token}` },
    });
  }
}
```

The **Model** describes the shape of the data:

```ts
export interface Todo {
  id: string;
  title: string;
  completed: boolean;
}
```

The difference in one sentence: **the Client acts, the Model describes.**

Why care? Because the API can be used to prepare data in a second instead of going through the UI. Registering a user via the API is five times faster than filling in a form.

## `data/`: Static Test Data

Fixed users, long texts, edge cases:

```ts
export const validUser = {
  email: 'user@example.com',
  password: 'Test1234',
};
```

> **Warning**
> User data that a test creates does not go here. Keep it inside the test itself so the test stays atomic and independent. The `data/` folder is for shared constants only, not for each test's state.

## Root Files

**`playwright.config.ts`**: the brain of the project. The `baseURL`, the timeouts, the Projects, the Reporters.

**`.env`**: secrets and URLs that change per environment:

```
BASE_URL=https://todo.example.com
API_URL=https://todo.example.com/api
TEST_USER_PASSWORD=Test1234
```

**`.env.example`**: the same keys without values, and this one gets committed. That way anyone new on the team knows what they need to fill in.

**`.gitignore`**:

```
node_modules/
test-results/
playwright-report/
playwright/.auth/
.env
```

> **Important**
> `playwright/.auth/` is where the `storageState` is saved, meaning a logged-in user's session. If it gets committed, you've pushed a real Token to Git. This is one of the most common leaks in automation projects, and a single line prevents it.

## Decision Rule

When you're unsure where to put a file, ask:

| Question | Location |
|---|---|
| Does it describe user behavior? | `tests/` |
| Does it define the Locators of a whole page? | `pages/` |
| Does it repeat across more than one page, or more than once on a page? | `components/` |
| Does it prepare something before the test? | `fixtures/` |
| Does it talk to the API? | `clients/` |
| Does it describe the shape of data? | `models/` |
| Is it a shared constant value? | `data/` |
| Is it a secret or a URL that changes per environment? | `.env` |

## Summary

- Structure isn't cosmetic: every folder prevents a specific kind of pain that shows up two months later.
- A spec file should be readable by a non-technical person. A CSS Selector inside a spec is a warning sign.
- A Page takes `page` and a Component takes `Locator`. That's the clearest sign you understand the Page Object Model.
- `clients/` and `models/` are the API world, and they're what let you prepare data in a second instead of going through the UI.
- `.env` is ignored and `.env.example` is committed, and `playwright/.auth/` must be in `.gitignore` because it contains a real session.
- Build the structure from day one. Reorganizing it after sixty tests is much harder than creating it empty.
