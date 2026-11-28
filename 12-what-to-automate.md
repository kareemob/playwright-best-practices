# What to Automate and What Not To

## Introduction

This is the first real decision. A beginner automates everything they see. An engineer automates whatever hurts when it fails.

## Deciding Flow by Flow

Example: a Todo application.

| Flow | Automate it? | Why |
|---|---|---|
| Login with a valid account | ✅ | Everything in the application sits behind it |
| Login with an invalid account | ✅ | The error message is part of security |
| Creating a new account | ✅ | The user's first experience; when it fails, you lose users |
| Adding a Todo | ✅ | The core function of the application |
| Deleting a Todo | ✅ | Data loss: more dangerous than failing to add |
| Every Validation message | ❌ | Unit Tests cover them faster and cheaper |
| Button colors and spacing | ❌ | Not the job of E2E; that's Visual Testing work |
| "Share on Facebook" button | ❌ | Third party, you don't control it, and it breaks Tests randomly |

> **Important**
> The rule that will serve you your whole career: **automate the Flow that, if it broke, someone would call you about.** If nobody would call, it's probably not a priority.

## Validation: A Clear Example

Validation is an excellent example of the idea. "The email must contain @" is a single line of code, and a Unit Test checks it in a fraction of a second.

## The Order: Why Login Comes First

> **Tip**
> Notice that the order goes from **least dependent to most dependent**. This isn't the user's order; it's the **build order**. The user registers an account before logging in, but we build Login first because it's the simplest and the most used by the rest of the Tests.

## What "Done" Means

Tests passing on your machine isn't a standard. This is the real standard: four points, and all of them must be met:

- **They pass in parallel.** Run with `fullyParallel: true` and they're all green (see `playwright configs/01-playwright-config-map.md`).
- **They pass twice in a row.** Without manually cleaning the database between runs.
- **They pass on CI.** A clean machine, a slower network, no open browser (see `playwright configs/05-playwright-github-actions.md`).
- **The failure is understandable from the name.** Without opening a Trace (see `07-organizing-test-suite.md`, Naming).

> **Warning**
> The second point is the one that sinks most projects. The Test passes the first time because the database was empty, and fails the second time because it filled up. If it doesn't pass twice, it's not a Test; it's a coincidence. Tests need to be atomic and clean up after themselves (see `08-fixtures-instead-of-hooks.md`, where every test gets its own user and cleanup is tied to the fixture).

And there's a fifth, non-technical standard: **someone else can read the Test and understand what it checks without asking you.** If it needs an explanation, the code isn't ready.
