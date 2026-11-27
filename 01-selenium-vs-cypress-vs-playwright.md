# Selenium vs Cypress vs Playwright: Choosing a Test Automation Tool

## Introduction

This question comes up in every team meeting: why Playwright? What's wrong with Selenium?

Most of the answers you'll hear are biased. Someone will tell you Playwright "killed" Selenium, someone else will say Cypress is easier and that's the end of it. Both are wrong.

This comparison aims to be honest, and at the end it covers when to choose each one, not who the winner is.

> **Important**
> Selenium is not being downplayed here, not even by a single line. It is still the most widely used tool in the market, and there is work that can only be done with it. The goal is to clarify the real technical differences.

## Objectives

After reading this, you should be able to:

- Explain the **architectural** difference between the three tools, not just the syntax difference
- Know where Selenium is still effectively the only option
- Understand why Cypress is easy and why its ease comes at a price
- Choose a tool for a real project and defend your choice with a technical argument

## The Architectural Difference: Where All the Differences Begin

Before any table, you need to understand how each tool talks to the browser. Every other difference is a consequence of this point.

**Selenium** talks to the browser through a standard protocol called **W3C WebDriver**. There is an intermediary, the **driver**, between your code and the browser. This intermediary is the reason for both its strength and its slowness: it's standardized, so it works with any browser and any language, but every command passes through an extra stop.

**Cypress** runs **inside the browser itself**, in the same execution loop as the application. There's no intermediary at all. This gives it speed and excellent network control, but it locks it inside the browser's box and its restrictions.

**Playwright** talks to the browser through a **direct control protocol** (like CDP), over a single persistent connection instead of a round trip per command. It sits outside the browser like Selenium, but much closer to it.

## Comparison Table

| Item | Selenium | Cypress | Playwright |
|---|---|---|---|
| Architecture | WebDriver, a standard intermediary | Inside the browser | Direct control protocol |
| Speed | Slowest | Fast | Fastest in practice |
| Languages | Java · Python · C# · Ruby · JS · Kotlin | JS/TS only | JS/TS · Python · Java · .NET |
| Auto-wait | ❌ You write it | ✅ Built in | ✅ Built in and deeper |
| Parallelism | Via Grid, separate setup | Limited for free | ✅ Built in and free |
| Browsers | Everything, even real Safari | Chrome + Firefox + experimental WebKit | Chromium · Firefox · WebKit |
| Multiple tabs and domains | ✅ | Difficult | ✅ Natural |
| Ecosystem maturity | 20+ years, the widest | Mature in frontend | Newer and growing fast |
| Real mobile | ✅ Via Appium | ❌ | ⚠️ Emulation only |
| Diagnostics | Logs and screenshots | Excellent Time Travel | Excellent Trace Viewer |

## Point by Point: What This Means in Practice

### Waiting

In Selenium you write the waiting yourself: `WebDriverWait` and `ExpectedConditions`. Very flexible, but every developer on the team writes it differently, and the first one who forgets produces a **flaky test**.

In Playwright, waiting is built into every action and every assertion. It's not smarter than the engineer, it just prevents forgetting. And that's a big difference on a team of ten people.

### Languages

Here Selenium clearly wins. If your team is Java or C#, Selenium speaks your language fully and maturely. Playwright has Java and .NET versions, but the community, the examples, and the Fixtures all revolve around TypeScript.

And that's a real reason to choose Selenium if the whole team is Java and doesn't want to move.

### Parallelism

Selenium needs **Grid**: separate infrastructure that you install and maintain. Very powerful on large projects, but it carries an administrative cost.

Cypress parallelizes through its cloud service or by manual splitting. Playwright parallelizes from the first command, for free, on your own machine.

### Mobile

> **Warning**
> Playwright does not run tests on a real mobile device. What it does is emulation: screen size, user agent, and touch. Excellent for checking **responsive** behavior, but not a substitute for testing a mobile app. For real mobile, the answer is **Appium**, and it's built on WebDriver, meaning on the same foundation as Selenium.

### Tabs and Domains

If your test opens a new tab, goes to a payment gateway on another domain, or deals with an iframe from an external origin, this is where Cypress struggles. Playwright and Selenium both handle this naturally.

## When to Choose Each One

### Choose Selenium when:

- Your team is Java, C#, or Python and there's no intention to change
- You need specific browsers and versions, or real Safari on macOS
- You already have a working Grid or existing infrastructure
- You want to connect web with real mobile through Appium
- The project will live for years and you want the widest possible ecosystem and support ✅

### Choose Cypress when:

- The team is a frontend team writing its own tests
- The app is a single-domain SPA with no tabs or multiple domains
- Developer experience and Time Travel matter most to you
- You don't want to manage infrastructure for parallelism ✅

### Choose Playwright when:

- You're starting a new web automation project
- You want stable tests with minimal manual waiting work
- You want real parallelism for free from day one
- The team is comfortable with TypeScript
- You need tabs, multiple domains, and strong diagnostics on CI ✅

## The Sentence That Sums It All Up

**Selenium is a standard, Cypress is an experience, Playwright is a balance.**

Selenium chose compatibility with everything and paid the price in speed and setup. Cypress chose developer comfort and paid the price in flexibility. Playwright came late, so it saw the mistakes of both and tried to take the middle ground.

That doesn't mean it's always the best. It means it's the reasonable default choice for a new web project in TypeScript.

> **Tip**
> If anyone asks, don't say "Playwright is better than Selenium." Say "Playwright solves the waiting and parallelism problem without extra setup, but Selenium covers more languages, browsers, and devices."

## Summary

- All the differences come back to architecture: Selenium through a WebDriver intermediary, Cypress inside the browser, Playwright through a direct control protocol.
- Selenium is the widest in languages, browsers, and adoption, and it's the right choice for Java/C# teams and for integrating with Appium.
- Cypress is excellent for frontend teams and single-domain apps, and it struggles with tabs and multiple domains.
- Playwright gives you deep auto-waiting, free built-in parallelism, and a powerful Trace Viewer, but its mobile support is emulation, not real devices.
- The decision isn't "which is best." It's the team's language, the shape of the app, and the infrastructure you have.
