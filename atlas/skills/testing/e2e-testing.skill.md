---
name: e2e-testing
description: >
  Rules and patterns for end-to-end tests using Playwright that verify complete user
  flows through a real browser against a running application. Covers the Page Object
  Model, locator priority rules, test isolation with browser contexts, and flakiness
  prevention. Required by QA agents writing UI or E2E tests and Dev Agents responsible
  for frontend features with user-facing flows.
applies-to: [dev-agent, qa-agent, qa-test-subagent]
stack: none
layer: Testing
version: 1.0
---

# E2E Testing (Playwright)

<context>
This skill applies when writing end-to-end tests that drive a real browser through
complete user flows (login → action → verify outcome). E2E tests are slower than
integration tests and should be reserved for critical paths. Each test MUST run
against the same application stack as production (with test-safe data only).
Source: Playwright Best Practices (playwright.dev/docs/best-practices).
</context>

<rules>
## Locator Priority (Playwright)
- Locators MUST use user-visible attributes in this priority order: role → label → placeholder → text → test ID.
- Locators MUST NOT use CSS classes, element IDs tied to implementation, or XPath — these couple tests to markup.
- `data-testid` attributes MAY be used when no role or label uniquely identifies the element.
- Locators MUST be resilient to DOM structure changes — always prefer semantic queries over structural ones.

## Test Isolation
- Each test MUST run in an isolated browser context (new page per test or new context per test file).
- Tests MUST NOT share cookies, localStorage, or authenticated sessions across test cases.
- Test state MUST be set up via API calls (direct HTTP to the application), not by navigating through the UI.
- Each test MUST clean up data it creates, or use ephemeral data scoped to the test run.

## Assertions
- Assertions MUST use Playwright's built-in async assertions (`expect(locator).toBeVisible()`) — these auto-retry until the condition is met or timeout.
- MUST NOT use `page.waitForTimeout()` as a substitute for assertions — hard sleeps produce flaky tests.
- Each test MUST assert on the final observable outcome (data saved, redirect occurred, notification shown), not intermediate states.
- Assertions MUST verify user-observable behavior, not internal implementation (DOM class names, React state).

## Page Object Model
- UI interactions for a reusable page or component MUST be encapsulated in a Page Object.
- Page Objects MUST expose semantic methods (`loginPage.login(email, password)`) not raw locators.
- Page Objects MUST NOT contain assertions — return data from navigation methods and assert in the test.
- Page Object constructors MUST accept a `Page` instance; they MUST NOT create pages themselves.

## Test Data
- E2E tests MUST use dedicated test accounts or dynamically-created accounts, never real user accounts.
- Test data MUST be created via the application's own API (not direct DB inserts), unless the API does not support it.
- Sensitive data (passwords, tokens used in tests) MUST be stored in environment variables or a secrets manager, not hardcoded.

## Parallelism
- Tests MUST be parallelizable: no test may depend on state left by another test in a different worker.
- Tests within a single file MAY run serially if they share a single browser context, but files MUST run in parallel.
</rules>

<patterns>
## Page Object Model (TypeScript)

```typescript
// pages/LoginPage.ts
import { Page, Locator } from "@playwright/test";

export class LoginPage {
    private readonly emailInput: Locator;
    private readonly passwordInput: Locator;
    private readonly submitButton: Locator;

    constructor(private readonly page: Page) {
        this.emailInput    = page.getByRole("textbox", { name: "Email" });
        this.passwordInput = page.getByRole("textbox", { name: "Password" });
        this.submitButton  = page.getByRole("button",  { name: "Sign in" });
    }

    async goto() {
        await this.page.goto("/login");
    }

    async login(email: string, password: string) {
        await this.emailInput.fill(email);
        await this.passwordInput.fill(password);
        await this.submitButton.click();
    }
}
```

## Test with API setup + Page Object (TypeScript)

```typescript
import { test, expect } from "@playwright/test";
import { LoginPage } from "../pages/LoginPage";

test.describe("Order Placement", () => {
    let authToken: string;

    test.beforeEach(async ({ request }) => {
        // Set up test data via API — not via UI navigation
        const res = await request.post("/api/test/users", {
            data: { email: "test@e2e.local", password: "TestPass!1" }
        });
        const { token } = await res.json();
        authToken = token;
    });

    test("user can place an order and see it in order history", async ({ page }) => {
        // Arrange — authenticate via API to skip login UI in every test
        await page.context().addCookies([{ name: "access_token", value: authToken, url: "http://localhost:3000" }]);

        // Act
        await page.goto("/products");
        await page.getByRole("button", { name: "Add to cart", exact: false }).first().click();
        await page.getByRole("link", { name: "Checkout" }).click();
        await page.getByRole("button", { name: "Confirm order" }).click();

        // Assert — user-observable outcome
        await expect(page.getByRole("heading", { name: "Order confirmed" })).toBeVisible();
        await expect(page.getByTestId("order-id")).not.toBeEmpty();
    });
});
```

## API-based auth setup (fixture)

```typescript
// fixtures/auth.ts
import { test as base } from "@playwright/test";

export const test = base.extend({
    authenticatedPage: async ({ page, request }, use) => {
        const res = await request.post("/api/auth/login", {
            data: { email: process.env.E2E_USER_EMAIL, password: process.env.E2E_USER_PASSWORD }
        });
        const { access_token } = await res.json();
        await page.context().addCookies([{ name: "access_token", value: access_token, url: process.env.BASE_URL! }]);
        await use(page);
    }
});
```

## Resilient locators (preferred vs. fragile)

```typescript
// PREFERRED — semantic, resilient to DOM changes
await page.getByRole("button", { name: "Submit order" }).click();
await page.getByLabel("Email address").fill("user@test.com");
await page.getByTestId("order-total").textContent();

// FRAGILE — coupled to implementation details
await page.locator(".btn.btn-primary.submit-btn").click();     // CSS class
await page.locator("#input-email-73").fill("user@test.com"); // auto-generated ID
await page.locator("div > form > div:nth-child(3) > input").fill("..."); // XPath-style
```
</patterns>

<anti-patterns>
- **`page.waitForTimeout(2000)`**: hard sleep — test passes if the app is slow, fails if it's fast, and masks real timing issues; use `expect(locator).toBeVisible()` with auto-retry.
- **CSS class locators**: `page.locator(".submit-btn")` — classes change during styling refactors, breaking tests that have nothing to do with behavior changes.
- **UI-driven test setup**: logging in through the UI before every test — multiplies test duration; set up auth state via API.
- **Tests depending on order**: test B requires test A to have created a user — parallel execution breaks the suite; each test must be self-contained.
- **Assertions on intermediate states**: asserting a loading spinner appeared — brittle and timing-dependent; assert the final user-visible outcome.
- **Hardcoded test credentials in source**: `email: "admin@test.com"`, `password: "password123"` — exposed in git history; use environment variables.
- **Page Objects with assertions**: `expect(this.successBanner).toBeVisible()` inside a Page Object method — assertions belong in tests, not in page objects.
- **Single browser context for all tests**: cookies and local storage bleed between tests; each test needs an isolated context.
</anti-patterns>

<checklist>
- [ ] All locators use role, label, placeholder, or `data-testid` — no CSS classes or XPath.
- [ ] Each test runs in an isolated browser context (fresh cookies, fresh localStorage).
- [ ] Auth state set up via API call, not by navigating the login UI.
- [ ] No `page.waitForTimeout()` — all waits use Playwright auto-retry assertions.
- [ ] Assertions verify the final user-observable outcome, not intermediate states.
- [ ] Page Objects expose semantic methods; assertions are not in Page Objects.
- [ ] No hardcoded credentials — stored in environment variables.
- [ ] Tests are parallelizable: no cross-test state dependency.
</checklist>
