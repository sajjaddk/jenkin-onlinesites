# Jenkins QA Pipeline

API and UI test automation against public practice sites, written in **three tools** (Newman, Cypress and Playwright) and run by a **Jenkins pipeline on a home Ubuntu Server** that watches this GitHub repository.

> Test targets are sites built for practice: [Restful-Booker](https://restful-booker.herokuapp.com) (API) and [SauceDemo](https://www.saucedemo.com) (UI). They belong to their owners. These tests are light functional checks only, with no load or attack testing. No employer code or data is used.

## How it fits together
```
 GitHub (this repo) <------ polls every ~5 min ------ Jenkins on Ubuntu Server (home network)
                                                         |
                  +--------------------------------------+---------------------------------+
                  |                                      |                                 |
        API: Newman (Postman)                  UI: Cypress (POM)              API + UI: Playwright (POM)
          -> Restful-Booker                       -> SauceDemo                  -> Restful-Booker + SauceDemo
                  |                                      |                                 |
                  +-------------- JUnit XML + HTML reports + Allure results ---------------+
                                                         |
                                  Jenkins: Test Result, API report, Playwright report,
                                  Allure report, archived artifacts, failure screenshots/traces
```
Jenkins is **not** exposed to the internet. It polls GitHub instead of receiving webhooks.

## What is tested
| Suite | Tool | Target | Tests |
|---|---|---|---|
| API | Postman collection run with Newman | Restful-Booker | 13 requests, 37 assertions: auth, create/read/list/filter, update, partial update, forbidden update, invalid payload, delete, deleted-is-gone |
| UI | Cypress with Page Object Model | SauceDemo | 14 tests: valid/invalid/locked-out login, required fields, protected page, logout, cart badge, sorting, full checkout, checkout validation |
| API | Playwright (`request` and fixtures) | Restful-Booker | 26 tests: health, auth (valid and bad credentials), create/get/list/filter, PUT (cookie and Basic auth), PATCH, DELETE, 403/404/405 cases, invalid bodies, special characters; each test creates and cleans up its own data |
| UI | Playwright with Page Object Model | SauceDemo | 26 tests plus a one-time login setup: login failures and protected page, logout and session end, product list, name and price sorting, cart badge, product page, cart actions, checkout validation, totals (item total + 8% tax = total), full purchase |

Why the same site is covered by more than one tool: it shows the same scenarios in the tools employers ask for, and the Playwright suite adds things the others don't, such as a saved login session, API fixtures with automatic clean-up, traces on failure, and calculated-total checks.

Reporting: JUnit (Jenkins trends), the Newman and Playwright HTML reports, and one **Allure** report for everything, with features, severity labels and steps.

Techniques: positive and negative testing, boundary and validation checks, authorisation checks, state-based flows (create then update then delete), end-to-end user journeys, test independence with fixtures and clean-up, data sharing between requests with variables, and calculated-value checks.

## Pipeline (`Jenkinsfile`)
1. **Install**: `npm ci`
2. **Lint**: ESLint on the Cypress and Playwright code
3. **Tests (in parallel)**: Newman, Cypress and Playwright (Playwright downloads Chromium on the first run)
4. **Post**: publish JUnit results, the API and Playwright HTML reports and the combined **Allure report**, archive reports, screenshots and traces, clean the workspace

Features: pipeline as code, SCM polling, a `SUITE` parameter (`all`, `newman`, `cypress`, `playwright`) to run part of the pipeline, a build timeout, build history limits, retries for flaky public sites, and no concurrent builds.



## Run the tests without Jenkins
Requires Node.js 20 or newer.
```bash
npm install
npx playwright install chromium   # first time only

npm run test:api       # Newman, report in reports/api/index.html
npm run test:ui        # Cypress headless, JUnit in reports/ui/
npm run test:pw        # Playwright API + UI, HTML report in reports/playwright/html
npm run test:pw:api    # Playwright API tests only
npm run test:pw:e2e    # Playwright UI tests only
npm run pw:ui          # Playwright interactive UI mode
npm run pw:report      # open the last Playwright HTML report
npm run cy:open        # Cypress interactive mode
npm run report         # build the Allure report from the last run (reports/allure-report)
npm run report:open    # open it in the browser
npm run lint
```

## Screenshots
_Added after the first Jenkins runs, in `docs/screenshots/`._

## Project layout
```
Jenkinsfile                      pipeline definition
api-tests/                       Postman collection and environment (Newman)
cypress/e2e/                     Cypress UI tests
cypress/support/pages/           Cypress page objects
playwright/api/                  Playwright API tests and fixtures
playwright/e2e/                  Playwright UI tests
playwright/pages/                Playwright page objects
playwright/setup/                one-time login that saves the session
playwright.config.js             Playwright projects, reporters, retries
docs/JENKINS_SETUP.md            server installation guide
```
