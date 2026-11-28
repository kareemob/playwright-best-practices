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
| Reordering todos by dragging | ❌ | Not yet: `dragTo` costs more than it's worth here (see `playwright actions/03-actions-reference.md`) |

> **Important**
> The rule that will serve you your whole career: **automate the Flow that, if it broke, someone would call you about.** If nobody would call, it's probably not a priority.

## Validation: A Clear Example

Validation is an excellent example of the idea. "The email must contain @" is a single line of code, and a Unit Test checks it in a fraction of a second.
