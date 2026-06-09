# **QA Automation (BDD \+ Playwright)**

After yesterday’s session, I spoke with a Senior Automation QA Engineer at Systems Limited to better understand how we can improve and automate our QA process. Based on his suggestions and further discussions with Gemini, I’ve put together a roadmap to help us gradually move from manual QA toward AI-assisted QA automation.

# **1\. Test Suite Organization**

All automation code will be managed inside a dedicated GitLab repository e.g `cx-automation-playwright`.

This gives us:

* Proper code backup and version control  
* Easier collaboration through merge requests and reviews  
* A clean foundation for future CI/CD and cloud execution

The repository structure will look like this:

```
qa-automation-playwright/
├── .gitlab-ci.yml
├── .cursorrules
├── playwright.config.ts
├── package.json
├── features/
│   └── routing.feature
└── tests/
    ├── pages/
    │   ├── AgentDeskPage.ts
    │   └── WebChatWidget.ts
    └── steps/
        └── routingSteps.ts
```

## **Folder Overview**

### **`features/`**

This is where we write our test scenarios in simple English using Gherkin syntax (`Given / When / Then`).

Example:

Given agent "Ali Hassan" is logged in

No automation code is written here.

### **`tests/pages/` (Page Object Model)**

This folder contains page selectors and reusable page actions.

The goal is to keep all selectors in one place. If something changes in the UI, we only update it once instead of fixing multiple test files.

### **`tests/steps/`**

This layer connects our plain-English feature steps with the actual Playwright code inside Page Objects.

# **2\. Branching Strategy**

To stay aligned with the development team, we will follow the same branching convention used by the development team. See Branching documentation:  
[https://expertflow-docs.atlassian.net/wiki/x/6gHwHg](https://expertflow-docs.atlassian.net/wiki/x/6gHwHg)

# **3\. BDD Guidelines & Selector Strategy**

ExpertFlow CX frontend apps are built on Angular, many CSS classes are dynamically generated and change after every deployment. Because of this, using raw CSS selectors can make tests unstable very quickly.

To keep our automation reliable, we will follow this selector priority:

## **Preferred Selector Strategy**

### **First Choice**

```ts
page.getByTestId('...')
```

This uses stable `data-testid` attributes from the application. At the moment, this pattern is not consistently implemented in ExpertFlow, so this needs discussion with the development team.

### **Second Choice (Default for now)**

```ts
page.getByRole('button', { name: '...' })
```

This uses accessible ARIA roles and is usually stable across UI updates.

## **Avoid Completely**

* Raw CSS classes  
* Long XPath selectors  
* Index-based selectors like `nth-child`

# **Clean Coding Rule**

BDD step files should stay clean and readable.They should never contain direct locator or interaction logic.  
**Incorrect**

```ts
await page.getByTestId('accept-btn').click();
```

## **Correct**

```ts
await agentDeskPage.acceptConversation();
```

This keeps the framework easier to maintain and scale.

# **4\. Organizing Test Cycles: Smoke vs Regression**

Instead of splitting tests into multiple folders, we will organize execution using tags inside feature files.

## **Smoke Tests (`@smoke`)**

These cover critical business flows, for example:

* Supervisor login  
* Webchat routing to queue

These tests will run automatically on every merge request.

## **Regression Tests (`@regression`)**

These cover:

* Edge cases  
* Multi-user scenarios  
* Boundary conditions  
* Negative testing

These will run nightly through scheduled pipelines.

## 

## 

## 

## **Run Commands**

### **Smoke Suite**

```shell
npx playwright test --grep @smoke
```

### **Regression Suite**

```shell
npx playwright test --grep @regression
```

# **5\. AI-Assisted Workflow**

We will use two AI tools together:

* Claude → for scenario design and business understanding  
* Cursor → for local code generation and implementation

## **Step 1: Generating BDD Scenarios with Claude**

I’ll be working on setting up a dedicated Claude Project for the QA team.

A Claude Project is basically a private AI workspace where we can upload:

* ExpertFlow documentation  
* Business workflows  
* Existing templates  
* QA standards

This helps Claude understand our product and workflows better.

Instead of giving generic responses, it can generate Gherkin `.feature` files tailored specifically for ExpertFlow when we paste Jira tickets into it.

## 

## 

## **Step 2: Generating Playwright Code with Cursor**

Once the feature file is ready, we can use Cursor to help generate automation code locally.

## **Important Question: Do we need to execute the flow manually first?**

Yes, initially we will use Playwright Codegen. We will run:

```shell
npx playwright codegen <staging-url>
```

A browser window opens, and we perform the flow manually one time. Playwright records the interactions and generates raw automation code automatically. This keeps the learning process simple for the team in the beginning. At a later stage we will see how we can automate this as well.

## **Why Cursor?**

The Playwright recorder generates raw code, but it does not organize the framework properly. Cursor helps us convert that raw code into:

* Page Objects  
* Clean reusable methods  
* Structured step definitions

Example prompt inside Cursor:

*`Convert this raw script into reusable Page Object methods using our AgentDeskPage.`*

# **Coding Guardrails**

I’ll work with Nabeel to prepare a shared `.cursorrules` file. This file will define our coding standards and guide Cursor to generate:

* Consistent code  
* Stable selectors  
* Proper framework structure

