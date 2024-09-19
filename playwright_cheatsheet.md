# Playwright

### Suggested patterns
> Try observing Codegen patterns to get a sense of how you'll actually need to write your awaits.

Playwright can be finicky about how you wait for things (perhaps why Cypress auto-waits). A good pattern:

```ts
const myElement = page.locator(someSelector); // note the lack of `await` here; you don't need the await until you're actually trying to assert or perform an action.

await expect(myElement).not.toBeVisible()
// click on shit or something
await expect(myElement).toBeVisible()
```

## Getting Started

* POM vs `page` object

Good reading to get started (once you have a basic understanding of the above):
* https://playwright.dev/docs/test-pom
* https://playwright.dev/docs/selectors
  * https://playwright.dev/docs/selectors#quick-guide
  * https://playwright.dev/docs/selectors#best-practices
* https://playwright.dev/docs/debug
* https://playwright.dev/docs/cli#inspect-selectors

## ROBUST TEST CHECKLIST

When writing any test, because Playwright is so finicky, it's wise to test robustly before releasing the test, so that you don't have to debug weird failures in the future. Follow this checklist:

* Tests should pass against both your local environment AND your staging environment.
* Tests should pass when `--repeat-each=6` is run.

## Setting up/Version
`npm i @playwright/test`
`npm ls @playwright/test`


## Config
> Over time I've found that command-line options are generally more useful than messing with the config.
```ts
const config: PlaywrightTestConfig = {
    workers: os.cpus().length - 2, // change how many cpus are used by the tests; one per thread.
    projects: [
        {
            name: 'Chrome',
            use: {
                browserName: 'chromium',
                // headless: true,
                headless: false, // set headless to false to launch a browser and watch the tests
                launchOptions: {
                    slowMo: 50, // slow down the test so you can follow along
                },
            },
        },
    ],
}
```

* `testConfig.workers`
    By default Playwright uses half the number of threads available on your machine. You can adjust it up or down via: 
    ```js
    const config: PlaywrightTestConfig = {
        workers: 3, // or, based on your OS - `os.cpus().length - 1`, for example. (and import os from 'os')
    };
    ```
    **However,** so far I've found that tests actually run slower when I assign more workers.

### Per-test config

#### Timeouts
> https://playwright.dev/docs/api/class-test#test-set-timeout

Modify the timeout for all tests in a `describe` block:
```ts
test.describe('some test', () => {
    test.setTimeout(10000) // default is 30000ms

    test('some test', ...)
    ... 
})
```

#### Device/Viewport/Browser

```ts
// Run tests in this file with portrait-like viewport.
test.use({
  viewport: { width: 600, height: 900 },
});

```

## Running Tests

https://playwright.dev/docs/1.19/test-cli

### CLI Options

I *think* you can pass all these in to the CLI: 
> https://playwright.dev/docs/api/class-testproject#test-project-repeat-each

I know these work:

* `--repeat-each=3`
    Useful for debugging flakey tests.
* `--debug`
    Headed, step-through test with console.
* `--headed`

## Page Object Model
A set of abstractions lain over a page to make running multiple tests on the page quicker. (So, instead of writing "go here, click this, click this other thing, type this", you can just create a class like "register", then call it in any test where you want to do that.)
> https://playwright.dev/docs/test-pom/

## Syntax

### Test Setup

* `beforeAll()`
    Note that `page` is not availabe in `beforeAll`. Things that require `page` generally must be done on a per-test basis.

### Selectors

#### `getBy...`
v.127 introduced `getBy` selectors, as in `RTL`, and *recommends them over page.locator*.

> https://playwright.dev/docs/api/class-framelocator#frame-locator-get-by-role

#### `$`, `$$` - Element Handles

Though use of element handles is strongly discouraged in Playwright, in favor of `getByRole` queries or the Locator api, in some cases they seem to be necessary, or at least expedient, such as in conditionals. The syntax is: 

`page.$(selector)`
`$$` returns an array of all elements matching that selector.

```ts
await page.$('id="close-modal"')
```

#### Single vs Multiple Elements

Locators are "strict", which means they will throw an error if more than one element matches the selector, *unless* you call a function that expects multiple, e.g. 

```ts
await page.locator('input').click(); // will fail if there are more than one

await page.locator('input').first().click(); // will not fail if there are more than one

await page.locator('input').count();
```

#### `getBy`
RTL-style `getBy` queries were released in late 2022, and are now recommended over the Locator queries below.

The one key difference I've found between RTL and Playwright's `getBy` queries is that RTL uses strict text match by default, whereas Playwright uses case-insensitive partial match. So in Playwright there's no need to pass regex to the `name` field, for example, in 

```ts
getByRole('button', {name: 'best colleges'}) // will match Best Colleges in Idaho
```

#### Text Selectors

```ts
await page.click('text=my text');
// shorthand
await page.click('"my text"');
```

*Shorthand*
Surround with quotes.
`await page.click('"Log in"')` = `text='Log in'`.

* **Basic**
    `text='some text'`
    Case-insensitive, matches substrings.
* **Regex**
    `text=/Log\s?in/i`

*Note* that, in many circumstances, I've found the only way I could successfully test text content was with queries like: 

```ts
expect(await page.textContent('.my-selector')).toContain('some text string')
```

##### Pseudo-Classes

* `:has-text()`
    Use with CSS selectors to match any of that CSS selector with the specified text. 
    `button:has-text("Log in")`
* `:text()`
    Matches the smallest element containing the specified text.
    `#nav-bar :text("Log in")` - `text=Log in`, but inside the `#nav-bar` element.
* `:text-is()`
    "", but strict text node match.
* `:text-matches()`
    With regex.
* `:visible`
    Only visible elements. This has two forms: 
    `'input:visible'` = only visible inputs.
    Or, you can combine syntaxes with the `(some selector) >> visible=true` syntax: 
    `'a:has-text("Massachusetts Institute of Technology") >> visible=true'`

*Note* that `button` and `submit` input elements are matched by their `value` rather than text content.

#### CSS Selectors

```ts
await page.locator('css=input').count()
await page.locator('css=[placeholder="Enter password"]').fill('password');
// shorthand
await page.locator('input').count()
await page.locator('[placeholder="Enter password"]').fill('password');
```

#### By Test ID
Playwright has shorthand for `id`, `data-testid`, `data-test-id`, and `data-test`. **Note** that the shorthand has to match the actual attribute - e.g., `id=whatever` wont work if you used `data-testid` on the element.
```ts
await page.locator('id=informationalcta'); // matches id="informationalcta"
await page.locator('data-testid=informationalcta') // matches data-testid="informationalcta"
```

You can test by id with either the `id` function, or css:
```ts
// queries data-test-id attribute with css
await page.locator('css=[data-test-id=directions]').click();
await page.locator('[data-test-id=directions]').click(); // short-form

// queries data-test-id with id
await page.locator('data-test-id=directions').click();
```


#### React Selectors

#### Combining Selectors

You can combine standard CSS selectors (anything that's a valid target for a CSS rule) with text selectors, with a couple syntaxes:
```ts
page.locator(button:has-text("Log in"));
page.locator(button >> text="Log in"); // identical

page.locator(a >> text="Partner Links");
```

#### By Attribute

```ts
page.locator(a[href="whatever/url"]);
```

#### Selecting Specific Elements

##### Select Elements

```ts
      await page
        .getByTestId('your_id')
        .type('Aw');
      await page
        .getByText(
          'Awesome pigs'
        )
        .click();
```
### Assertions

#### General


* *Note* that assertions must be `await`ed, unless the element has already resolved (i.e., been awaited already).
    ```ts
    await expect(page.locator('text=whatever')).toBeVisible();
    ```

* Playwright has its own convenience assertions: 
    ```ts
    const logOutVis = await page.isVisible('text=log in')
    expect(logOutVis).toBeTruthy()
    ```
    though I'm not entirely sure how they're more convenient...
    > https://playwright.dev/docs/assertions

#### Asserting for...

### Waiting

Much of Playwright has auto-waiting built-in; basically, anything where it *knows* it needs to wait for something like navigation or loading (`page.goto()`, for example, knows it always needs to wait for a page load). Even things like `page.click()` has auto-waiting for navigation built in. (But you still need to `await` it.)

However, there are (common) situations in which auto-waiting fails, such as when a page waits for a network request/response before navigation, or when a page is lazy-loaded. In these cases, you may need to wait explicitly.


* `page.waitForNavigation()`
    Useful for waiting for asynchronous navigation, such as after a network request/response.
    ```ts
    await page.waitForNavigation();
    ```
    Note that often, you'll actually need to use this form: 
    ```ts
    await Promise.all([
            this.page.waitForNavigation(), // set up the wait first, so that it's in place before the action that causes the navigation; think of it as a listener.
            this.page.click(this.selectors.buttons.saveButton),
          ]);
    ```
    You can pass in a specific url, if the navigation will pass through multiple urls:
    ```ts
    await page.waitForNavigation({ url: '/whatever/url'})
    ```

* `page.waitFor()`
    Useful when elements are lazy loaded.
    ```ts
    await page.click('text=Login')
    await page.locator('#username').waitFor(); // wait for the element to appear

    // pass options
    await page.locator('#username').waitFor({state: 'visible'})
    ```

#### Waiting for multiple things
    ```ts
    await Promise.all([
            this.page.waitForNavigation(), // set up the wait first, so that it's in place before the action that causes the navigation; think of it as a listener.
            this.page.click(this.selectors.buttons.saveButton),
          ]);
    ```

### User Actions

* **Check a checkbox**
    `await page.check(selector)`
    *Note* that to uncheck it, you use a separate
    `await page.uncheck(selector()`.
* **Debugging**
    Playwright sucks with checkboxes. If a test is failing because "checking checkbox failed to change state", try changing the order in the test in which it's checked, or try preceding it with a `click()` on the same element.

## Tools

### Open
Use `npx playwright open` to open a webpage with Playwright running, and pass various options.

```
npx playwright open example.com

# Open page in WebKit
npx playwright wk example.com

# Emulate iPhone 11.
npx playwright open --device="iPhone 11" wikipedia.org
```

### Codegen
`npx playwright codegen <my-site>`
e.g.
`npx playwright codegen http://localhost:3000`

### Inspector
> https://playwright.dev/docs/inspector#using-browser-developer-tools

Playwright offers an in-browser inspector for use during debugging. It does the same thing as `codegen`, but instead you use `page.pause()` in your code, and click "explore" in the headed browser. (This is only available in the browser instance that Playwright is running.)

* It can be helpful to insert `page.waitForEvent('close')` after a potential error, if you want time to investigate it in the browser.

#### Run queries in the console
> https://playwright.dev/docs/cli#inspect-selectors
While you have `open` or `codegen` running, you can run Playwright queries in the browser console: 

```ts
playwright.inspect('text=Log in')
playwright.locator('.auth-form', { hasText: 'Log in' });

// generate a selector
playwright.selector($0)
```

### `--debug`
Opens codegen, but with the test code loaded in it, and a JS debugger, so that you can step through the code. Very useful.
`npx playwright test --debug myTest.test.ts`

### Page.pause()
Will stop the test and launch the inspector. 

Note that *you must* prefix with `await`. 

```ts
await page.pause()
```

### Snapshots
> https://playwright.dev/docs/test-snapshots

Playwright snapshots are different than Jest's in that they're actual pixel-comparisons, as opposed to Jest's comparison of rendered html.

Uses this engine: https://github.com/mapbox/pixelmatch

#### Notes

* Make sure you don't have any sort of dynamic content on a page (ads, modals that only appear sometimes, etc.); that'll make the test fail/brittle.

## API

### `page`
`page` provides methods to interact with a single tab in a browser.

#### `textContent()`
`page.textContent(selector[, options])`
Returns an element handle for an element matching the selector.

Takes an options object, with: 
* `strict<boolean>` - requires it to resolve with one element only, fails on multiple. 
* `timeout<number>` - in ms; defaults to 15000.

#### `url()`
    Returns current url.


## Authentication

### Authentication tokens/cookies
In tests, a browser context is created for each test, and so, if you log in, via something like 

```ts
const { ADMIN_EMAIL, ADMIN_PASSWORD } = process.env

await request.post(`${process.env.REACT_APP_ENV_URL}/api/login`, {
    data: { email: ADMIN_EMAIL, extendTTL: true, password: ADMIN_PASSWORD },
})
```
then the `set-cookie` header on the response will set the cookie in the context within that test; and any subsequent requests to the endpoint *within that test* will have access to the authentication token. (And, as with a normal browser, that token will be appended to each request to the token's origin.)

### Sharing authentication between contexts

The documentation around this is awful. 
> https://playwright.dev/docs/test-api-testing#using-request-context
> https://playwright.dev/docs/api/class-fixtures#fixtures-request

But, it's not hard to do. You need to: 

```ts
    import { test, request } from '@playwright/test' // note that we're importing request HERE, not using the `request` object available via test, e.g. `test('my test', ({ request }) => {...})`. That's because the latter `request` is an APIRequestContext, which doesn't have methods for *creating* new contexts, etc. The `request` we're importing here is an APIRequest, which, via `newContext()`, returns an APIRequestContext.


    //...
    test('api accepts expected payloads', async () => {
        const { ADMIN_EMAIL, ADMIN_PASSWORD } = process.env

        // we have to manually create a context so that we can store the authentication token received from `/api/login`.
        const request = await baseRequest.newContext({}) // create the APIRequestContext via the APIRequest

        // sign in via a login endpoint; the `set-cookie` header will set the token in this context.
        await request.post(`${process.env.REACT_APP_ENV_URL}/api/login`, {
            data: { email: ADMIN_EMAIL, extendTTL: true, password: ADMIN_PASSWORD },
        })

        // store the token received in a context. The file name is arbitrary.
        await request.storageState({ path: 'state.json' })
        await requestAndValidateAPIEndpoint(request, mockData)
    })

    test('accepts payload without color and label', async ({ browser }) => {
        // we create a new browser context, and import the context file from the previous test. Note that, although APIRequest also has the `newContext` method, using it will not set the proper context; you need to access the `browser` fixture (available as a `test` parameter), and create a new context with that, importing the same context file you saved above.
        const context = await browser.newContext({ storageState: 'state.json' })

        const { color, label, ...request2 } = mockData
        await requestAndValidateAPIEndpoint(context.request, request2)
    })


```

## Stubbing Network Responses

> https://playwright.dev/docs/network#modify-responses

```ts
await page.route('**/*', route => {
            /** Intercept the submit request so as not to pollute the scholarship in the DB, which must remain without data for the tests to work. This does mean that we're not testing the contract with the api, which should perhaps occur elsewhere. */
            if (route.request().method() === 'PUT') {
                /** Validate data being sent to DB. Note that, if these fails, the error message will be something unhelpful like "waiting for navigation until 'load'" */
                const request = JSON.parse(route.request().postData() as string) as ITransformedData

                expect(request.template).toEqual('half-width-header')
                expect(request.scholarshipFragment).toEqual('qa-test-e2e-test')
                return route.fulfill({
                    status: 200,
                    body: '{ success: true }',
                })
            }

            if (route.request().method() === 'GET') {
                return route.fulfill({
                    status: 404,
                    body: '{ error: "GUID does not correspond to a known landing page: some-long-guid"}', // NOTE that the body needs to be a string, not an actual object
                })
            }

            return route.continue()
        })
```

## How To...

### Run tests in parallel

```ts
test.describe.parallel('my tests', () => {
    test(...);
    test(...);
} 
```
Each test will run in its own thread. Adjust `testConfig.workers` to adjust the number of threads available.

### Test that an element is present, or not
without asserting anything further.

```ts
await page.locator(selector).waitFor({state: 'visible'}) // or 'attached'
```

or 

```ts
(page.$('#close-modal')) !== null) 
```

### Narrow when multiple elements are returned
Instead of `(text="my text")`, try something like `(a:has-text("My Text"))`.

### Use OR in an expect statement
> https://stackoverflow.com/questions/44654210/logical-or-for-expected-results-in-jest
Reversing is one option - 

```ts
expect([
            'Deletion successful',
            "Deletion failed. Error status from server: text does not exist for page: 'https://www.mysite.com/some-page'",
        ]).toContain(await page.textContent('text=Deletion'))
```

### Detect when code is being run by a test

Set a `userAgent` in the config: 

```ts
// playwright.config.ts
const config: PlaywrightTestConfig = {
    timeout: 30000,
    forbidOnly: !!process.env.CI,
    projects: [
        {
            name: 'Chrome',
            use: {
                browserName: 'chromium',
                headless: true,
                userAgent: 'playwright',
            },
        },
```

Then check for it in your code:
```ts
if (navigator.userAgent === 'playwright') return true
// or, if you need to exclude Jest unit tests as well
if (navigator.userAgent === 'playwright' || navigator.userAgent.includes('jsdom')) return true
```

> https://playwright.dev/docs/emulation#user-agent

### Forward browser logs to test terminal

```ts
page.on("console", (message) => console.log("BROWSER: ", message.text()));
```

### Scroll down to an element which isn't visible yet. 

Unfortunately "scroll to bottom" doesn't exist in Playwright yet, so you have to:

* use either `page.mouse.wheel` or simulate a "pagedown" press. 

    ```ts
    await page.mouse.wheel(0, 8700);
    ```

* OR, scroll to an element that *is* visible and nearby, using `page.locator(whatever).scrollIntoViewIfNeeded()`.

### Testing an element lazy-loaded with an intersection observer

Use `scrollIntoViewIfNeeded` on the class/an id on the `LazyLoadingContainer`, then use a timeout on the `Locator.click()`.

### Deal with pages opened in a new tab

Something unintuitive about Playwright is that, if you click a link in your test with `target="_blank"`, the `page` object you're working with *still refers to the original page you opened the link on.*

To get ahold of the new page, you have to:

```ts
const [newPage] = await Promise.all([
    context.waitForEvent('page'), // get `context` by destructuring with `page` in the test params; 'page' is a built-in event, and **you must wait for this like this,**, or `newPage` will just be the response object, rather than an actual Playwright page object.
    page.locator('text=Click me').click() // note that, like all waiting in Playwright, this is somewhat unintuitive. This is the action which is *causing the navigation*; you have to set up the wait *before* it happens, hence the use of Promise.all().
])

await newPage.waitForLoadState('domcontentloaded'); // this is necessary, I think, to make sure it's loaded.
await expect(newPage).toHaveURL(scholarship.partnerURL);
```

### Conditional testing

```ts
if ((await page.$('#close-modal')) !== null) { // or page.$$().length > 0
    // do shit
}
```
Or, I haven't tried yet, but this was recommended if you wish to use `locator` - 
```ts
const myElement = page.locator('#elem-id') {
if (await myElement.isVisible())
  await myElement.click()
}
```

## API Testing with Playwright

Works. 

> https://playwright.dev/docs/test-api-testing#writing-api-test

## Errors/Debugging

**First Steps**
- Run the test with `npx playwright test --debug myTest.test.ts`, and step through, to get a sense of what's going on.
- Whatever's failing, if it doesn't have an `await` in front of it - try putting an await in front of it.
- Check if the page is loading fully (is the loaded indicator still an `x`?). Pages will frequently get held up by a third-party script failing to resolve, which prevents the `load` event that Playwright listens for before starting tests. If this happens consistently, either use a route intercept, or pass `waitUntil: 'domcontentloaded'` to the `page.goto()`, so that Playwright doesn't wait for `load`, just `domcontentloaded` (which fires before third-party scripts resolve).

**Specific messages**
* *`ERR_SSL_PROTOCOL_ERROR`*
    If you're pointing at a url with `https` in it, try `http` (if it's local, a self-signed cert, etc.).
* *Auto-suggest not appearing in search box*
    Use `type()`, not `fill()`. 
* *Tests failing when nothing has changed*
    * Did you turn on `slowMo` in the config? They may be timing out.
* *`locator...: Target closed`*
    - Make sure you're not missing an `await` prior to the failing line.
    - Are you navigating? Try adding an `await page.waitForNavigation()` after the action that causes navigation, in case it doesn't auto-wait (or the navigation is too complex for the auto-waiting to work, such as if it's waiting for a network response before navigating). 
* *Identical strings failing `toContain` or other equality check*
    Is there an `&nbsp;` in the original? That will fail, but the expected/received output will look identical. (Adding `&nbsp;` to the expected string doesn't seem to work; I just use `toContain` and send the rest of the string.)

**When you can't figure it out...**
- If possible, try rearranging when things are happening. Order affects things in mysterious ways occasionally, it seems.
- See [Tools](#tools)