# /e2e — End-to-End Test Generator

Generate Playwright or Detox e2e tests for the specified feature or user flow.

## Steps

1. **Identify target**
   - If argument provided: generate tests for that feature/page/flow
   - Otherwise: examine recent changes and identify the most critical user flow

2. **Understand the feature**
   - Read relevant component/screen files
   - Identify the user journey: entry → interactions → expected outcome
   - List all happy path steps and key error states

3. **Detect test framework** in use:
   ```bash
   cat package.json | grep -E "playwright|detox|cypress"
   ```
   - Web → Playwright (preferred)
   - React Native → Detox
   - Fallback → Playwright

4. **Generate test file:**

   ### Playwright (Web)
   ```typescript
   import { test, expect } from '@playwright/test'

   test.describe('<Feature Name>', () => {
     test.beforeEach(async ({ page }) => {
       await page.goto('/path')
     })

     test('happy path: <description>', async ({ page }) => {
       // Arrange
       // Act
       // Assert
     })

     test('error case: <description>', async ({ page }) => {
       // ...
     })
   })
   ```

   ### Detox (React Native)
   ```typescript
   describe('<Screen Name>', () => {
     beforeAll(async () => {
       await device.reloadReactNative()
     })

     it('should <behavior>', async () => {
       await expect(element(by.id('testId'))).toBeVisible()
       await element(by.id('button')).tap()
       await expect(element(by.text('Success'))).toBeVisible()
     })
   })
   ```

5. **Output test file path** and run command:
   ```bash
   npx playwright test tests/e2e/<feature>.spec.ts
   # or
   npx detox test --configuration ios.sim.debug
   ```

## Rules

- Use `data-testid` attributes for selectors (not CSS classes or text)
- Note which `data-testid` attributes need to be added to source components
- Cover: happy path, empty state, error state, loading state
- Keep tests independent — each test resets state
