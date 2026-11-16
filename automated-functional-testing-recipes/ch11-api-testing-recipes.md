# Chapter 11: API Testing Recipes

---

Your web application is an iceberg. The browser-facing UI is the tip -- what users see and click. Below the surface sits the API layer: REST endpoints, JSON payloads, authentication headers, status codes, and data contracts. When the UI breaks, users notice. When the API breaks, everything breaks.

webship-js includes a complete API testing toolkit built on axios. You can test REST APIs without writing a single line of JavaScript -- every HTTP method, header, authentication scheme, and response assertion is available as a Gherkin step. This means your API tests sit in the same test suite as your UI tests, run with the same command, and speak the same BDD language.

This chapter covers every API testing pattern you need.

---

## Understanding REST API Testing

<!-- diagram: api-flow -->

```
┌──────────────────────────────────────────────────────────────────┐
│ API Request/Response Lifecycle                                   │
│                                                                  │
│  ┌──────────────────┐    HTTP Request    ┌──────────────────┐   │
│  │ Test (axios)     │   ──────────────►  │ Server           │   │
│  │                  │                    │                  │   │
│  │ Method: POST     │                    │ Route: /api/users│   │
│  │ URL: /api/users  │                    │ Controller       │   │
│  │ Headers:         │                    │ Validation       │   │
│  │  Content-Type:   │                    │ Database query   │   │
│  │  application/json│                    │ Business logic   │   │
│  │ Body:            │                    │                  │   │
│  │  {"name":"John"} │                    │                  │   │
│  │                  │   ◄──────────────  │                  │   │
│  │ Assert:          │    HTTP Response   │ Status: 201      │   │
│  │  Status = 201    │                    │ Body: {"id": 1,  │   │
│  │  Body has "id"   │                    │  "name":"John"}  │   │
│  └──────────────────┘                    └──────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

webship-js API tests use axios (not the browser) to make HTTP requests directly. This means:
- No browser is needed (faster execution)
- You test the API contract directly (not through the UI)
- You can test endpoints that have no UI at all
- Request/response data is precise and inspectable

```
UI Test vs API Test:
┌──────────────────────────────────┬──────────────────────────────┐
│ UI Test                          │ API Test                     │
│                                  │                              │
│ Browser → UI → API → Database   │ axios → API → Database       │
│                                  │                              │
│ Slow (renders page)              │ Fast (no rendering)          │
│ Tests UI + API together          │ Tests API in isolation       │
│ Catches visual bugs              │ Catches data/logic bugs      │
│ Brittle (UI can change)          │ Stable (API contract)        │
│                                  │                              │
│ "I press 'Save' and see         │ "POST to /api/users returns  │
│  'User created'"                 │  201 with user object"       │
└──────────────────────────────────┴──────────────────────────────┘
```

---

## Recipe 116: Setting Up the API Base URL

**Problem:** The third time you copy an API suite to a new project, you find yourself doing the same find-and-replace: swap the hardcoded `https://api.client-a.com` for client B's host, in forty scenarios. Miss one and the suite quietly tests the wrong company's API. Base-URL configuration is a one-line step, but where you put that line decides whether your suite is portable or welded to a single environment.

**Solution:**

```gherkin
Feature: API Configuration
  Scenario: Set API base URL
    Given the API base URL is "https://jsonplaceholder.typicode.com"
    When I send a GET request to "/posts/1"
    Then the API response code should be 200

  Scenario: Use relative URL with launch URL
    # If LAUNCH_URL is http://localhost:8080
    Given the base URL is "/api/v1"
    # Requests go to http://localhost:8080/api/v1/...
    When I send a GET request to "/users"
    Then the API response code should be 200

  Scenario: Full URL base
    Given I set the API base URL to "https://api.example.com/v2"
    When I send a GET request to "/products"
    Then the API response code should be 200
```

**How It Works:**

The base URL step accepts three syntax variants:
- `Given the API base URL is "URL"`
- `Given I set the API base URL to "URL"`
- `Given the base URL is "URL"`

If the URL starts with `http://` or `https://`, it is used as-is. Otherwise, it is appended to the `LAUNCH_URL` environment variable.

```
URL Resolution:
┌─────────────────────────────────────────────────────────────┐
│ Base URL                    │ Endpoint      │ Final URL      │
├─────────────────────────────────────────────────────────────┤
│ https://api.example.com     │ /users        │ https://api.   │
│                             │               │ example.com/   │
│                             │               │ users          │
│                             │               │                │
│ /api/v1                     │ /users        │ http://local   │
│ (LAUNCH_URL=localhost:8080) │               │ host:8080/api  │
│                             │               │ /v1/users      │
└─────────────────────────────────────────────────────────────┘
```

The relative form is the one that keeps suites portable. `Given the base URL is "/api/v1"` resolves against `LAUNCH_URL` -- the same environment variable your browser tests already use (default `http://localhost:8080`). Set `LAUNCH_URL` once per environment in CI, and both your UI scenarios and your API scenarios follow it; nothing in the feature files changes between dev, staging, and the client's UAT server.

**At scale:** Reserve absolute URLs in feature files for genuinely external services (a public sandbox like jsonplaceholder, a third-party API you integrate with). Everything that belongs to the project under test should go through the relative form. When your DevOps team wires forty pipelines, "every suite obeys LAUNCH_URL" is one convention; "every suite has its own hardcoded hosts in unpredictable places" is forty small archaeology projects.

---

## Recipe 117: GET Requests

**Problem:** A mobile developer pings you: the app's article screen is blank, but the website shows articles fine. The website goes through its own rendering layer with fallbacks; the app consumes `/posts` raw -- and the endpoint started returning an empty array for a specific query parameter three deploys ago. UI tests never saw it because the UI papers over it. GET endpoints need their own assertions, at the contract level, where the mobile app and every other consumer actually live.

**Solution:**

```gherkin
Feature: GET API Requests
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Fetch a single resource
    When I send a GET request to "/posts/1"
    Then the API response code should be 200
    And the API response should contain "sunt aut facere"
    And the response should be valid JSON

  Scenario: Fetch a list of resources
    When I send a GET request to "/posts"
    Then the API response code should be 200
    And the response should be valid JSON

  Scenario: Fetch with query parameters
    When I send a GET request to "/posts?userId=1"
    Then the API response code should be 200
    And the API response should contain "userId"

  Scenario: Fetch a non-existent resource
    When I send a GET request to "/posts/99999"
    Then the API response code should be 404
```

**How It Works:**

`When I send a GET request to "/posts/1"` makes an HTTP GET request using axios:

```javascript
// What webship-js does internally
apiResponse = await axios({
  method: 'GET',
  url: baseURL + '/posts/1',
  headers: apiHeaders,
  validateStatus: () => true  // Don't throw on any status
});
```

The `validateStatus: () => true` is crucial -- it means axios will NOT throw an error for 4xx or 5xx status codes. This lets you assert the status code in your Then step instead of catching exceptions. The practical consequence: a 404 is not a test failure unless you say it is. That is what makes the fourth scenario possible -- `Then the API response code should be 404` treats the error response as the expected, assertable outcome, which is exactly how you pin down "missing resources return 404, not 200-with-empty-body" as a contract guarantee.

The response object is held for the rest of the scenario, so you can stack as many assertions as you like against one request -- status, body text, JSON properties -- without re-fetching. One request, many checks, is both faster and kinder to rate-limited APIs.

---

## Recipe 118: POST Requests with JSON Body

**Problem:** Your team writes API tests in four different styles -- one engineer inlines JSON strings, another builds bodies in custom JavaScript steps, a third pastes curl commands into README files as "the tests". Reviewing a pull request means first decoding which dialect it speaks. webship-js gives you four built-in ways to send a JSON body; the job is to know what each is for and pick a house style, so the two-hundredth POST test reads like the first.

**Solution:**

```gherkin
Feature: POST API Requests
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Create a resource with inline body
    Given I set the request body to '{"title": "New Post", "body": "Content", "userId": 1}'
    When I send a POST request to "/posts"
    Then the API response code should be 201
    And the API response should contain "New Post"

  Scenario: Create a resource with doc string body
    When I send a POST request to "/posts" with body:
      """
      {
        "title": "My New Post",
        "body": "This is the post content",
        "userId": 1
      }
      """
    Then the API response code should be 201
    And the JSON response should have "title" equal to "My New Post"

  Scenario: Create a resource with data table
    When I send a POST request to "/posts" with values:
      | title  | My New Post              |
      | body   | This is the post content |
      | userId | 1                        |
    Then the API response code should be 201
    And the API response should contain "My New Post"

  Scenario: Create a resource with table-based body setup
    Given I set the request body with:
      | title  | My New Post              |
      | body   | This is the post content |
      | userId | 1                        |
    When I send a POST request to "/posts"
    Then the API response code should be 201
```

**How It Works:**

webship-js offers four ways to set the POST body:

```
Method 1: Inline JSON string
  Given I set the request body to '{"key": "value"}'

Method 2: Doc String (multiline JSON)
  When I send a POST request to "/url" with body:
    """
    {"key": "value"}
    """

Method 3: Data Table (key-value pairs)
  When I send a POST request to "/url" with values:
    | key | value |

Method 4: Table body setup (separate from request)
  Given I set the request body with:
    | key | value |
  When I send a POST request to "/url"
```

```
When to use each:
┌────────────────────────┬───────────────────────────────────┐
│ Method                 │ Best For                          │
├────────────────────────┼───────────────────────────────────┤
│ Inline JSON string     │ Simple, short JSON payloads       │
│ Doc String             │ Complex, nested JSON              │
│ Data Table             │ Flat key-value objects             │
│ Table body setup       │ Reusable body across requests     │
└────────────────────────┴───────────────────────────────────┘
```

A nuance worth knowing: the data-table forms build a flat object -- every value arrives as a string, and there is no nesting. If the endpoint cares about types (`userId` as a number, `active` as a boolean) or expects nested structures, use the doc string, where you write real JSON and what you see is exactly what is sent. The doc string is also the form a product owner can read in a pull request and recognize as "the payload from the API docs", which is worth something during sign-off.

**At scale:** Pick the doc string as the default house style and write it down. The four variants exist because different payloads have different shapes, but a portfolio where every suite freely mixes all four is a portfolio where nobody can grep for "all tests that create a user". One style, documented in your team's testing guideline, makes cross-project review and copy-between-suites cheap.

---

## Recipe 119: PUT Requests (Full Update)

**Problem:** An integration partner updates a user's phone number through your API -- and the user's email vanishes. Their developer sent a PUT with only the changed field, your API took "replace the resource" literally, and the missing fields became nulls. Whether that is the partner's bug or yours depends entirely on what your API contract promises for PUT -- and if no test pins that promise down, the answer changes with every refactor.

**Solution:**

```gherkin
Feature: PUT API Requests
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Update an entire resource
    When I send a PUT request to "/posts/1" with body:
      """
      {
        "id": 1,
        "title": "Updated Title",
        "body": "Updated body content",
        "userId": 1
      }
      """
    Then the API response code should be 200
    And the JSON response should have "title" equal to "Updated Title"
    And the JSON response should have "body" equal to "Updated body content"

  Scenario: Update with data table
    When I send a PUT request to "/posts/1" with values:
      | id     | 1                     |
      | title  | Completely New Title  |
      | body   | Completely new body   |
      | userId | 1                     |
    Then the API response code should be 200
    And the API response should contain "Completely New Title"
```

**How It Works:**

PUT replaces the entire resource. Every field must be included, or the missing fields will be removed (depending on the API design). This is the difference between PUT and PATCH:

```
Original resource:
{ "id": 1, "title": "Old", "body": "Old body", "userId": 1 }

PUT with { "title": "New" }:
{ "title": "New" }                   ← Other fields removed!

PATCH with { "title": "New" }:
{ "id": 1, "title": "New", "body": "Old body", "userId": 1 }
                                      ← Other fields preserved
```

That parenthetical -- "depending on the API design" -- is exactly what your test exists to pin down. If your API treats PUT as a strict full replacement, send a complete body and assert every field comes back; if it rejects partial PUTs, send one with a missing field and assert the 400 or 422. Either design is defensible. An *untested* design is the problem, because it changes silently with the next framework upgrade -- which is how the partner's phone-number update deleted that email address.

One practical note on the `with values:` table variant: it builds a flat object and every value arrives as a string. For a full-resource PUT that carries numeric ids or booleans, prefer the doc string -- `"id": 1` and `"id": "1"` are different payloads, and plenty of APIs accept only one of them.

---

## Recipe 120: PATCH Requests (Partial Update)

**Problem:** Your mobile team ships autosave: every time a user edits one field, the app PATCHes just that field. Then a backend refactor quietly starts treating every update as a full replacement -- and autosaving a new title begins nulling out the body. Each side behaves correctly by its own reading of the API; the data loss lives in the gap between them. PATCH semantics are a contract, and this is the test that gives the contract teeth.

**Solution:**

```gherkin
Feature: PATCH API Requests
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Partially update a resource
    When I send a PATCH request to "/posts/1" with body:
      """
      {
        "title": "Only Title Changed"
      }
      """
    Then the API response code should be 200
    And the JSON response should have "title" equal to "Only Title Changed"
    # Other fields should remain unchanged
    And the JSON response should have property "body"
    And the JSON response should have property "userId"

  Scenario: Update multiple fields
    When I send a PATCH request to "/posts/1" with values:
      | title | Updated Title |
      | body  | Updated Body  |
    Then the API response code should be 200
```

**How It Works:**

The assertions that matter most are the last two in the first scenario: after patching only the title, the scenario checks that `body` and `userId` still exist in the response. That pair of property checks *is* the PATCH contract -- fields you did not send must survive. Drop them and the scenario only proves the title changed, which a destructive full-replace implementation would also pass.

The doc-string form sends exactly the JSON you wrote, so a one-field payload stays a one-field payload. The `with values:` table builds a flat object with string values -- fine for titles and bodies, wrong for anything type-sensitive. And as with every API step, the response stays held for the rest of the scenario, so you can stack as many property assertions after one PATCH as the contract deserves.

---

## Recipe 121: DELETE Requests

**Problem:** A user double-clicks the delete button, the frontend fires two DELETE requests, and the second one returns a 500 that lands in your error tracker every day for a month. Or your nightly cleanup job deletes expired records, hits one that is already gone, and the whole job crashes at 3 a.m. DELETE looks like the simplest verb in HTTP until you ask what happens when the resource is *already deleted* -- and that answer is worth pinning down before your consumers discover it for you.

**Solution:**

```gherkin
Feature: DELETE API Requests
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Delete a resource
    When I send a DELETE request to "/posts/1"
    Then the API response code should be 200

  Scenario: Delete a non-existent resource
    When I send a DELETE request to "/posts/99999"
    Then the API response code should be 404

  Scenario: Verify deletion
    # Create a resource
    When I send a POST request to "/posts" with body:
      """
      {"title": "Temporary Post", "body": "Will be deleted", "userId": 1}
      """
    Then the API response code should be 201

    # Delete it
    When I send a DELETE request to "/posts/101"
    Then the API response code should be 200
```

**How It Works:**

The second scenario is the one most suites skip: deleting something that does not exist. A clean 404 lets consumers treat "already gone" as success and retry safely; a 500 turns every retry and every double-click into an incident. Whichever behavior your API has today, this scenario freezes it so a framework upgrade cannot change the answer without a test going red.

The third scenario shows the create-then-delete shape, but be honest about what proves deletion: a 200 from the DELETE only proves the server accepted the request. The full proof is a follow-up GET to the same resource asserting 404 -- and on a real API you would thread the created resource's id through with a placeholder (Recipe 129) instead of hardcoding it. Note also that some APIs return 204 No Content for successful deletes rather than 200; both are legitimate designs, so assert whichever yours actually promises.

**At scale:** Treat DELETE scenarios the way Chapter 9 treats bulk actions: tag them `@destructive` and point them only at environments seeded for the purpose. An API suite that deletes records on a shared staging server can sabotage the browser suite running against the same database five minutes later -- a cross-team failure that takes far longer to diagnose than it would have taken to prevent.

---

## Recipe 122: Setting Request Headers

**Problem:** The request works perfectly in Postman and fails with a 403 in your suite. After an hour of comparing payloads you find it: Postman was quietly sending an API key header you configured months ago and forgot existed. Headers are invisible context -- authentication, content negotiation, tenant selection, API versioning all ride in them -- and the first time you automate against a gateway that wants three of them at once, you need a clean way to declare headers where a reviewer can actually see them.

**Solution:**

```gherkin
Feature: Request Headers
  Background:
    Given the API base URL is "https://api.example.com"

  Scenario: Set a single header
    Given I set header "Content-Type" with value "application/json"
    And I set header "Accept" with value "application/json"
    When I send a GET request to "/users"
    Then the API response code should be 200

  Scenario: Set header with alternative syntax
    Given I set the header "Authorization" to "Bearer eyJhbGciOiJIUzI1NiJ9..."
    When I send a GET request to "/protected/resource"
    Then the API response code should be 200

  Scenario: Set multiple headers with data table
    Given I set the following headers:
      | Content-Type  | application/json         |
      | Accept        | application/json         |
      | Authorization | Bearer token123          |
      | X-Custom-Key  | my-custom-value          |
    When I send a GET request to "/users"
    Then the API response code should be 200

  Scenario: API Key authentication via header
    Given I set header "X-API-Key" with value "sk-test-12345"
    When I send a GET request to "/api/data"
    Then the API response code should be 200
```

**How It Works:**

Headers are stored in a global `apiHeaders` object and sent with every subsequent request in the scenario. Set a header once and it rides along on everything that follows; set the same header again and the new value overwrites the old, which is how you exercise things like token rotation mid-scenario. The `Before` hook resets the whole object between scenarios, so a header set in one scenario cannot contaminate the next -- no matter what order the scenarios run in.

```
Header syntax variants:
┌───────────────────────────────────────────────────────────────┐
│ Syntax                                          │ Use Case    │
├───────────────────────────────────────────────────────────────┤
│ I set header "X" with value "Y"                 │ Single      │
│ I set the header "X" to "Y"                     │ Single      │
│ the header "X" is "Y"                           │ Single      │
│ I set the following headers: (table)            │ Multiple    │
│ I am authenticating as "X" with "Y" password    │ Basic Auth  │
└───────────────────────────────────────────────────────────────┘
```

**At scale:** The header values in these examples are illustrations -- in a real project, API keys and tokens come from CI secrets, never from feature files committed to the repository. The pattern that holds up across many projects: a `Background` that sets the boring headers every request needs, plus a small custom step that reads the secret from an environment variable and sets the Authorization header. Twenty projects can then share identical feature files while each pipeline injects its own credentials.

---

## Recipe 123: Basic Authentication

**Problem:** The client's staging server sits behind HTTP Basic Auth -- the ops team put it there years ago to keep crawlers and curious eyes out, and every request without credentials bounces with a 401. Your suite, pointed at staging for the first time on a Tuesday afternoon, goes entirely red in thirty seconds. Basic Auth is the oldest gate in the HTTP book and it is still everywhere: internal APIs, staging environments, quick admin endpoints. One step opens it.

**Solution:**

```gherkin
Feature: Basic Authentication
  Background:
    Given the API base URL is "https://api.example.com"

  Scenario: Authenticated request succeeds
    Given I am authenticating as "admin" with "password123" password
    When I send a GET request to "/api/users"
    Then the API response code should be 200
    And the API response should contain "users"

  Scenario: Unauthenticated request fails
    When I send a GET request to "/api/users"
    Then the API response code should be 401
    And the API response should contain "Unauthorized"

  Scenario: Wrong credentials fail
    Given I am authenticating as "admin" with "wrong-password" password
    When I send a GET request to "/api/users"
    Then the API response code should be 401

  Scenario: Authenticated CRUD operations
    Given I am authenticating as "admin" with "password123" password

    # Create
    When I send a POST request to "/api/users" with body:
      """
      {"name": "John", "email": "john@example.com"}
      """
    Then the API response code should be 201

    # Read
    When I send a GET request to "/api/users/1"
    Then the API response code should be 200
    And the API response should contain "John"

    # Update
    When I send a PATCH request to "/api/users/1" with body:
      """
      {"name": "John Updated"}
      """
    Then the API response code should be 200

    # Delete
    When I send a DELETE request to "/api/users/1"
    Then the API response code should be 200
```

**How It Works:**

`Given I am authenticating as "admin" with "password123" password` creates a Basic Authentication header:

```
Step: I am authenticating as "admin" with "password123" password

Internally:
  base64("admin:password123") → "YWRtaW46cGFzc3dvcmQxMjM="
  Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=

This header is sent with every subsequent request in the scenario.
```

Underneath, this is just a header: the step base64-encodes `username:password` and stores the result in the same `apiHeaders` object as Recipe 122, so it follows every request in the scenario and resets before the next one. Two things trip people up. First, base64 is encoding, not encryption -- anyone who can read the request can recover the credentials, which is why Basic Auth belongs behind HTTPS only. Second, this header authenticates the axios client and nothing else: your browser steps know nothing about it, because the browser and the API client carry separate identities -- the same split Chapter 10 called out.

---

## Recipe 124: Bearer Token Authentication

**Problem:** Someone pastes a working JWT into the feature file, the suite goes green, everyone moves on. Seven days later the token hits its expiry and every API scenario fails at once -- and because the failure says "401" rather than "your hardcoded token died of old age", the team spends a morning suspecting the API itself. Tokens are born expiring. A suite that depends on a pasted one has a countdown timer built into it; the fix is to earn a fresh token at the start of every scenario.

**Solution:**

```gherkin
Feature: Bearer Token Authentication
  Background:
    Given the API base URL is "https://api.example.com"

  Scenario: Authenticate with Bearer token
    Given I set header "Authorization" with value "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    When I send a GET request to "/api/profile"
    Then the API response code should be 200
    And the API response should contain "name"
    And the API response should contain "email"

  Scenario: Login to get token, then use it
    # Step 1: Login to get a token
    When I send a POST request to "/api/auth/login" with body:
      """
      {"email": "admin@example.com", "password": "AdminPass123"}
      """
    Then the API response code should be 200
    And the JSON response should have property "token"

    # Step 2: Use the token for authenticated requests
    # Note: You would extract the token and set it as a header
    # This requires a custom step definition (see below)

  Scenario: Expired token returns 401
    Given I set header "Authorization" with value "Bearer expired.token.here"
    When I send a GET request to "/api/profile"
    Then the API response code should be 401
    And the API response should contain "Token expired"

  Scenario: Missing token returns 401
    When I send a GET request to "/api/profile"
    Then the API response code should be 401
    And the API response should contain "Unauthorized"
```

**How It Works:**

A Bearer token is just an Authorization header with a different prefix, so `Given I set header "Authorization" with value "Bearer ..."` is all the built-in machinery you need -- the header lands in the scenario's `apiHeaders` and rides every request that follows. The interesting question is where the token comes from. The login-then-use scenario shows the shape: POST credentials, receive a token, use it. Out of the box the two halves do not connect, because no built-in step copies a value from a response into a header -- that is the one gap the custom step below closes.

**Custom Step for Token Extraction:**

To chain a login request with subsequent authenticated requests, add this custom step:

```javascript
// tests/step-definitions/custom-api-auth.js
const { Then } = require('@cucumber/cucumber');

Then('I save the response {string} as the auth token', function (propertyPath) {
  const token = getNestedProperty(apiResponse.data, propertyPath);
  if (!token) throw new Error(`No value found at "${propertyPath}" in response`);
  apiHeaders['Authorization'] = 'Bearer ' + token;
});
```

Then use it:

```gherkin
Scenario: Login and use token
  When I send a POST request to "/api/auth/login" with body:
    """
    {"email": "admin@example.com", "password": "AdminPass123"}
    """
  Then the API response code should be 200
  And I save the response "token" as the auth token
  When I send a GET request to "/api/profile"
  Then the API response code should be 200
```

**At scale:** Put the token-extraction step in your shared step-definition package, with credentials read from environment variables, and every project gets real token auth without pasting anything. One caution from running this across many suites: a fresh login per scenario, multiplied by hundreds of scenarios and dozens of nightly pipelines, is real load on your identity service. If it starts rate-limiting your CI, log in once per run and cache the token -- the API twin of the storage-state trick from Chapter 10.

---

## Recipe 125: Form Data Requests

**Problem:** You point your JSON-speaking test at the legacy login endpoint and get a 400 with an error message that helps nobody. After some archaeology you discover the endpoint predates the JSON era: it wants `application/x-www-form-urlencoded`, the format HTML forms have posted since the nineties. OAuth token endpoints, payment gateways, and anything that began life as a server-rendered form still speak it -- and sending JSON at them is answering a letter with a phone call.

**Solution:**

```gherkin
Feature: Form Data Requests
  Background:
    Given the API base URL is "https://api.example.com"

  Scenario: Login with form data
    When I send a POST request to "/login" with form data:
      """
      username=admin
      password=secret123
      remember=true
      """
    Then the API response code should be 200
    And the API response should contain "session"

  Scenario: Submit contact form via API
    When I send a POST request to "/api/contact" with form data:
      """
      name=Tester Demo
      email=rajab@example.com
      message=Hello from the test suite
      """
    Then the API response code should be 200
    And the API response should contain "Message received"
```

**How It Works:**

The form data step parses `key=value` pairs from the doc string and sends them with `Content-Type: application/x-www-form-urlencoded`:

```
Input:                          Sent as:
username=admin                  Content-Type: application/x-www-form-urlencoded
password=secret123              Body: username=admin&password=secret123&remember=true
remember=true
```

The step sets the Content-Type for you -- no manual header needed -- and URL-encodes the values, so a message containing spaces or ampersands arrives intact. One pair per line keeps the doc string readable in review. When an endpoint rejects your request, let the status code tell you which layer is wrong: a 415 means the content type itself was unexpected, while a 400 or 422 means the format was fine and the values were not -- two different bugs that look identical from a distance.

---

## Recipe 126: Response Status Code Assertions

**Problem:** A framework migration changes the backend's default error handler, and suddenly every failure returns 200 with `{"error": "..."}` tucked in the body. Nothing crashes -- it degrades. Clients that branch on status codes stop seeing errors, retry logic stops retrying, and the monitoring dashboard stays green while users stare at broken screens. Status codes are the one part of your contract every consumer relies on without reading the docs, which makes them the cheapest thing to pin down and the most expensive thing to get wrong.

**Solution:**

```gherkin
Feature: Status Code Verification
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario Outline: Common HTTP status codes
    When I send a <method> request to "<endpoint>"
    Then the API response code should be <status>

    Examples:
      | method | endpoint     | status |
      | GET    | /posts/1     | 200    |
      | GET    | /posts/99999 | 404    |
      | POST   | /posts       | 201    |
      | DELETE | /posts/1     | 200    |
```

**How It Works:**

The Scenario Outline runs once per Examples row -- one request, one status assertion, one independent pass/fail line in the report. Because the API steps configure axios with `validateStatus: () => true` (Recipe 117), the 404 row is just another passing test rather than an exception to catch. The real asset is the Examples table itself: method, endpoint, expected status -- a status-code contract in a format anyone can extend without touching a step definition. When a new endpoint ships, covering it costs one row.

**HTTP Status Code Reference:**

```
┌──────────┬─────────────────────────┬───────────────────────────────┐
│ Code     │ Meaning                 │ When You See It               │
├──────────┼─────────────────────────┼───────────────────────────────┤
│ 200      │ OK                      │ Successful GET, PUT, PATCH    │
│ 201      │ Created                 │ Successful POST (new resource)│
│ 204      │ No Content              │ Successful DELETE             │
│ 400      │ Bad Request             │ Invalid input/payload         │
│ 401      │ Unauthorized            │ Missing/invalid auth          │
│ 403      │ Forbidden               │ Auth OK but no permission     │
│ 404      │ Not Found               │ Resource does not exist       │
│ 405      │ Method Not Allowed      │ Wrong HTTP method for route   │
│ 409      │ Conflict                │ Duplicate resource            │
│ 422      │ Unprocessable Entity    │ Validation failed             │
│ 429      │ Too Many Requests       │ Rate limit exceeded           │
│ 500      │ Internal Server Error   │ Server bug                    │
│ 503      │ Service Unavailable     │ Server down/maintenance       │
└──────────┴─────────────────────────┴───────────────────────────────┘
```

**At scale:** Keep one of these outlines per service and treat its Examples table as the canonical answer to "what should this endpoint return?". It works in both directions: developers add rows when they add routes, and a delivery manager can read the table during release review without anyone translating it. When every project in the portfolio carries the same outline structure, auditing any service's status contract takes thirty seconds -- and the question "is release X safe?" starts having evidence behind it.

---

## Recipe 127: Response Body Assertions

**Problem:** The backend team adopts a new serializer, and every field quietly changes from `snake_case` to `camelCase`. The API still "works" -- valid JSON, right data, status 200 -- but every consumer parsing `user_name` now reads `undefined`. A text-level check like "the response contains John" sails straight past this, because the break is in the structure, not the content. Body assertions have to work at the level your consumers actually read: properties, values, and shapes.

**Solution:**

```gherkin
Feature: Response Body Verification
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Response contains text
    When I send a GET request to "/posts/1"
    Then the API response should contain "sunt aut facere"
    And the API response should not contain "error"

  Scenario: Response is valid JSON
    When I send a GET request to "/posts/1"
    Then the response should be valid JSON

  Scenario: JSON property exists
    When I send a GET request to "/posts/1"
    Then the JSON response should have property "id"
    And the JSON response should have property "title"
    And the JSON response should have property "body"
    And the JSON response should have property "userId"

  Scenario: JSON property has specific value
    When I send a GET request to "/posts/1"
    Then the JSON response should have "id" equal to 1
    And the JSON response should have "userId" equal to 1

  Scenario: JSON property should not exist
    When I send a GET request to "/posts/1"
    Then the JSON response should not have property "password"
    And the JSON response should not have property "secret"

  Scenario: Nested JSON property
    When I send a GET request to "/users/1"
    Then the JSON response should have property "address.city"
    And the JSON response should have property "company.name"

  Scenario: Response matches expected JSON structure
    When I send a POST request to "/posts" with body:
      """
      {"title": "Test Post", "body": "Test body", "userId": 1}
      """
    Then the API response code should be 201
    And the API response should contain json:
      """
      {
        "title": "Test Post",
        "body": "Test body",
        "userId": 1
      }
      """
```

**How It Works:**

webship-js provides a rich set of response assertions:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Assertion Step                                │ What It Checks      │
├─────────────────────────────────────────────────────────────────────┤
│ API response should contain "text"            │ Text in body        │
│ API response should not contain "text"        │ Text NOT in body    │
│ response should be valid JSON                 │ Body is valid JSON  │
│ JSON response should have property "x"        │ Property exists     │
│ JSON response should not have property "x"    │ Property missing    │
│ JSON response should have "x" equal to Y      │ Property value      │
│ API response should contain json: """..."""   │ JSON subset match   │
│ response header "X" should be "Y"             │ Header value        │
│ API response code should be N                 │ Status code         │
└─────────────────────────────────────────────────────────────────────┘
```

The `contain json` assertion does a **subset match**: it checks that every property in the expected JSON exists in the actual response with the same value. The response can have additional properties that are not checked.

```
Subset Match Example:
Expected: { "title": "Test", "userId": 1 }
Actual:   { "id": 101, "title": "Test", "body": "content", "userId": 1 }
Result:   PASS (expected properties match, extra properties ignored)
```

Choose the assertion to match the failure you fear. The `contain "text"` steps do a raw substring search over the serialized body -- quick, readable, and blind to structure; they would never notice the snake_case rename as long as the value still appears somewhere. The `JSON response` steps parse the body and walk the property path (dot notation for nesting), so they fail precisely when a field is renamed, moved, or dropped. And the most underrated line in the whole feature is `should not have property "password"`: positive assertions prove the API does its job, negative ones prove it does not leak -- password hashes, internal flags, fields meant for another role.

**At scale:** Maintain a short "never in a response" list -- password, token, secret, internal ids -- and assert it as a standard block in every project's API suite. It is three lines of Gherkin per service, it gives the security questionnaire a concrete answer, and the one time it fires (a serializer change that starts including password hashes) it pays for every suite that ever carried it.

---

## Recipe 128: Response Header Assertions

**Problem:** The platform team swaps the reverse proxy during an infrastructure migration, and the new one drops the `Cache-Control` header your API used to send. Nothing fails -- responses just quietly stop being cached, the database absorbs traffic the CDN used to take, and three weeks later someone traces a mysterious load problem back to a header nobody looks at. Headers are where your API meets the infrastructure, and the infrastructure changes without telling your test suite. Unless the suite checks.

**Solution:**

```gherkin
Feature: Response Header Verification
  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Verify Content-Type header
    When I send a GET request to "/posts/1"
    Then the response header "Content-Type" should contain "application/json"

  Scenario: Verify Cache-Control header
    When I send a GET request to "/posts/1"
    Then the response header "Cache-Control" should contain "max-age"

  Scenario: Verify custom headers
    Given the API base URL is "https://api.example.com"
    Given I am authenticating as "admin" with "password123" password
    When I send a GET request to "/api/data"
    Then the response header "X-Rate-Limit-Remaining" should contain "99"
```

**How It Works:**

The step reads the named header from the held response -- header names are case-insensitive, per the HTTP spec -- and does a substring match on the value. That "contain" semantics is deliberate: asserting `Content-Type` contains `application/json` passes whether or not the server appends `; charset=utf-8`, which is exactly the tolerance you want for a value that varies legitimately between servers.

Header assertions earn their keep on the boundaries you do not own: `Content-Type` for clients with strict parsers, `Cache-Control` for anything sitting behind a CDN, rate-limit headers for partners who budget against them. They are also the assertions that catch infrastructure regressions -- the kind that pass every body check, because the application code never changed.

---

## Recipe 129: Placeholder Substitution

**Problem:** Your scenario creates a user, then needs to fetch, update, and delete that same user -- but the id only exists at runtime, inside the create response. So someone hardcodes `/api/users/42`, because user 42 happens to exist on their machine, and the scenario fails in CI where the database starts empty. Hardcoded ids are how API suites end up welded to one particular database snapshot. Placeholders are how you unweld them.

**Solution:**

```gherkin
Feature: Placeholder Substitution
  Background:
    Given the API base URL is "https://api.example.com"
    Given I am authenticating as "admin" with "password123" password

  Scenario: Create and then fetch a resource using placeholder
    Given I set placeholder "{{userId}}" to "42"
    When I send a GET request to "/api/users/{{userId}}"
    Then the API response code should be 200
    And the API response should contain "{{userId}}"

  Scenario: Dynamic placeholders in request body
    Given I set placeholder "{{name}}" to "John Doe"
    Given I set placeholder "{{email}}" to "john@example.com"
    When I send a POST request to "/api/users" with body:
      """
      {
        "name": "{{name}}",
        "email": "{{email}}"
      }
      """
    Then the API response code should be 201
    And the API response should contain "{{name}}"
```

**How It Works:**

`Given I set placeholder "{{userId}}" to "42"` stores the key-value pair. Before any URL, body, or assertion is processed, all placeholders are replaced:

```
Before:  /api/users/{{userId}}
After:   /api/users/42

Before:  {"name": "{{name}}", "email": "{{email}}"}
After:   {"name": "John Doe", "email": "john@example.com"}
```

Placeholders are reset between scenarios (in the `Before` hook), so they do not leak across tests.

---

## Recipe 130: Complete CRUD API Test

**Problem:** Each API endpoint passes its own little test, and the API still ships broken. Create returns a 201, sure -- but does the thing it created actually come back from a GET? Does an update stick, or does the next read show the old value? Testing each verb in isolation misses the bugs that live in the seams between them, which is exactly where data-integrity problems hide. You need one scenario that walks a resource through its whole life and proves each step left the world in the state the next step assumes.

**Solution:**

```gherkin
Feature: API CRUD Lifecycle
  As an API consumer
  I want to create, read, update, and delete resources
  So that I can manage data through the API

  Background:
    Given the API base URL is "https://jsonplaceholder.typicode.com"

  Scenario: Complete CRUD cycle for posts
    # CREATE
    When I send a POST request to "/posts" with body:
      """
      {
        "title": "My New Post",
        "body": "This is the content of my new post",
        "userId": 1
      }
      """
    Then the API response code should be 201
    And the response should be valid JSON
    And the JSON response should have property "id"
    And the JSON response should have "title" equal to "My New Post"

    # READ
    When I send a GET request to "/posts/1"
    Then the API response code should be 200
    And the response should be valid JSON
    And the JSON response should have property "title"
    And the JSON response should have property "body"
    And the JSON response should have property "userId"

    # UPDATE (full)
    When I send a PUT request to "/posts/1" with body:
      """
      {
        "id": 1,
        "title": "Updated Post Title",
        "body": "Updated post content",
        "userId": 1
      }
      """
    Then the API response code should be 200
    And the JSON response should have "title" equal to "Updated Post Title"

    # UPDATE (partial)
    When I send a PATCH request to "/posts/1" with body:
      """
      {
        "title": "Partially Updated Title"
      }
      """
    Then the API response code should be 200
    And the JSON response should have "title" equal to "Partially Updated Title"

    # DELETE
    When I send a DELETE request to "/posts/1"
    Then the API response code should be 200
```

**How It Works:**

This is one scenario, not five, and that is the point. The steps share one `apiResponse` and one base URL across the whole pickle, so the GET genuinely reads back what the POST wrote, and the assertion after the PUT proves the change persisted rather than just that the PUT returned 200. Each `When` overwrites the stored response and each `Then` asserts against that latest one, so the scenario reads top to bottom like the resource's biography. webship-js drives all of it through axios with no browser involved -- API scenarios skip Playwright entirely, which is why they run in milliseconds and never flake on a slow render.

The comment markers (`# CREATE`, `# READ`) are free documentation: Cucumber ignores them, but a product owner reading the report sees the lifecycle laid out in plain language. One caveat worth teaching the team -- public sandbox APIs like jsonplaceholder fake their writes, so the "updated" value you read back is the one you sent, not one the server stored. Against a real API the same scenario actually proves persistence; against a mock it proves your test wiring.

**At scale:** A CRUD lifecycle scenario is the most portable test you own. Every resource in every service -- users, orders, articles, invoices -- has the same create/read/update/delete shape, so this becomes a template your team stamps out per resource across twenty services with only the paths and field names changing. Because it needs no browser, it runs as a near-instant pre-merge gate on every API project at almost no CI cost, catching the data-integrity regressions that are ruinously expensive to find in production.

---

## Recipe 131: Testing Error Responses

**Problem:** The happy path is the easy half. The expensive bugs are the ones where a bad request gets a 200 and a cheerful "success" -- a malformed email saved as-is, a duplicate user silently created, garbage JSON swallowed without complaint. Those failures do not announce themselves; they corrupt data quietly and surface as a support ticket three weeks later. You need tests that deliberately send bad input and insist the API pushes back with the right status code and a useful message.

**Solution:**

```gherkin
Feature: API Error Handling
  Background:
    Given the API base URL is "https://api.example.com"

  Scenario: Missing required field
    Given I am authenticating as "admin" with "password123" password
    When I send a POST request to "/api/users" with body:
      """
      {
        "email": "john@example.com"
      }
      """
    Then the API response code should be 422
    And the API response should contain "name is required"

  Scenario: Invalid email format
    Given I am authenticating as "admin" with "password123" password
    When I send a POST request to "/api/users" with body:
      """
      {
        "name": "John",
        "email": "not-an-email"
      }
      """
    Then the API response code should be 422
    And the API response should contain "invalid email"

  Scenario: Duplicate resource
    Given I am authenticating as "admin" with "password123" password
    When I send a POST request to "/api/users" with body:
      """
      {
        "name": "Admin",
        "email": "admin@example.com"
      }
      """
    Then the API response code should be 409
    And the API response should contain "already exists"

  Scenario: Invalid JSON payload
    Given I am authenticating as "admin" with "password123" password
    Given I set header "Content-Type" with value "application/json"
    When I send a POST request to "/api/users" with body:
      """
      {not valid json}
      """
    Then the API response code should be 400
    And the API response should contain "Invalid JSON"
```

**How It Works:**

Each scenario asserts two things together, and you need both. The status code (`422`, `409`, `400`) is the machine-readable contract that clients branch on; the message substring (`"name is required"`, `"already exists"`, `"Invalid JSON"`) is the human-readable part your own front-end shows a user. An API that returns the right code with a useless message, or a friendly message with the wrong code, is half-broken in a way a code-only check would miss. The `should contain` step does a substring match against the whole response body, so it survives small wording changes while still proving the error names the actual problem.

Notice the codes are specific on purpose: `422` for "well-formed but semantically invalid" (the email field is missing), `409` for a conflict (the resource already exists), `400` for "I could not even parse this." Teaching the team to assert the precise code -- not just "something in the 4xx range" -- is what stops a future refactor from quietly turning a 422 into a 400 and breaking every client that branched on it.

**At scale:** Error-path coverage is where API quality actually lives, and it is the first thing dropped under deadline pressure. Make a standing rule that every endpoint ships with at least one negative scenario per documented failure mode, and the rule travels across services because the shapes repeat -- missing field, bad format, duplicate, unparseable body are the same four tests everywhere. For a PMO, the count of negative scenarios per endpoint is a concrete, comparable quality metric across the whole portfolio, far more telling than a raw "tests passing" number that the happy path alone can inflate.

---

## Recipe 132: Testing API Pagination

**Problem:** Pagination works fine with the ten rows in your dev database, then falls over the day a real customer has eight thousand. The off-by-one that drops the last record, the page-beyond-the-end that 500s instead of returning empty, the total-count header that lies -- none of these show up until the dataset is big, and by then it is production data and a customer is staring at a missing order. You need to pin down the boundary behavior while the cost of being wrong is still just a red test.

**Solution:**

```gherkin
Feature: API Pagination
  Background:
    Given the API base URL is "https://api.example.com"
    Given I am authenticating as "admin" with "password123" password

  Scenario: First page of results
    When I send a GET request to "/api/users?page=1&per_page=10"
    Then the API response code should be 200
    And the response should be valid JSON
    And the response header "X-Total-Count" should contain "50"

  Scenario: Second page of results
    When I send a GET request to "/api/users?page=2&per_page=10"
    Then the API response code should be 200
    And the response should be valid JSON

  Scenario: Last page with fewer results
    When I send a GET request to "/api/users?page=5&per_page=10"
    Then the API response code should be 200

  Scenario: Page beyond available data
    When I send a GET request to "/api/users?page=100&per_page=10"
    Then the API response code should be 200
    # Should return empty array or 404 depending on API design

  Scenario: Custom page size
    When I send a GET request to "/api/users?page=1&per_page=25"
    Then the API response code should be 200
```

**How It Works:**

The interesting scenarios here are the boundaries, not the first page. "Page beyond available data" pins down a design decision every paginated API has to make -- empty array or 404 -- and freezes it so a future change cannot silently flip the contract and break clients that depend on one or the other. The `X-Total-Count` assertion uses webship-js's response-header step to check the count the UI relies on to draw "Page 1 of 5"; a wrong total there means a correct dataset behind a broken pager. Query strings ride along in the request URL exactly as written, so you are exercising the same parameter parsing a real client hits.

These run as fast as any other API scenario because there is no browser -- you can cover first page, last page, past-the-end, and custom size in the time a single UI test takes to open Chromium. The honest gap: against a mock API the total count and page math are whatever the mock returns, so meaningful pagination tests need either a real environment or a stub seeded with a known number of rows (Recipe in Chapter 39 covers mocking a fixed dataset for exactly this).

**At scale:** Pagination, sorting, and filtering are the parameter-driven features most likely to be copy-pasted between services and least likely to be re-tested after the copy. A shared set of boundary scenarios -- first, last, past-the-end, custom size -- becomes a checklist you apply to every list endpoint across the portfolio, so the off-by-one that one team fixes does not silently reappear in the next service that cloned the code. Pin the empty-vs-404 decision once as an organization-wide API convention and assert it everywhere, so every client team can rely on the same answer.

---

## Recipe 133: Debugging API Responses

**Problem:** A scenario asserts the response contains `"success"` and it does not, and the failure message tells you what was expected but not what actually came back. You are now guessing: did the request hit the wrong path, did auth fail, did the API return an error envelope you did not anticipate? Guessing at an invisible response is the slowest way to debug an API test. You want to see the real bytes the server sent.

**Solution:**

```gherkin
Feature: API Debugging
  Scenario: Print response for debugging
    Given the API base URL is "https://jsonplaceholder.typicode.com"
    When I send a GET request to "/posts/1"
    Then print API response
    # This will output to the console:
    # GET https://jsonplaceholder.typicode.com/posts/1 => 200:
    # {
    #   "userId": 1,
    #   "id": 1,
    #   "title": "sunt aut facere...",
    #   "body": "quia et suscipit..."
    # }
```

**How It Works:**

`Then print API response` outputs the request method, URL, status code, and full response body to the console. This is invaluable for debugging:

```
Console output:
GET https://jsonplaceholder.typicode.com/posts/1 => 200:
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati...",
  "body": "quia et suscipit\nsuscipit recusandae..."
}
```

**Tip:** Use `print API response` during development, then remove it before committing. Leaving debug output in your feature files clutters the test output.

**At scale:** `print API response` is a development aid, not a suite feature -- a feature file that ships with it left in is the API equivalent of a forgotten `console.log`, and across twenty repos those add up to noisy CI logs nobody reads. Make "no `print` steps in committed features" a line in the review checklist (it is trivial to grep for in CI: fail the build if a tracked `.feature` contains `print API response`). When a failure does need a captured response in the report rather than the console, lean on the screenshot-and-artifact path from the reporting recipes instead, so the evidence lands in the shared report where a teammate on another project can read it.

---

## API State Reset Between Scenarios

One critical detail: webship-js resets ALL API state before each scenario:

```javascript
Before(async function () {
  apiResponse = null;    // Clear previous response
  apiRequestData = null; // Clear request body
  apiHeaders = {};       // Clear all headers
  authorization = '';    // Clear auth
  placeHolders = {};     // Clear placeholders
});
```

This means:
- Headers set in one scenario do NOT carry over to the next
- Authentication must be set up in each scenario (or in a Background)
- Placeholders are scenario-scoped
- No state leaks between tests

Use `Background` to avoid repeating common setup:

```gherkin
Feature: User API
  Background:
    Given the API base URL is "https://api.example.com"
    Given I am authenticating as "admin" with "password123" password

  Scenario: List users
    When I send a GET request to "/api/users"
    Then the API response code should be 200

  Scenario: Get single user
    When I send a GET request to "/api/users/1"
    Then the API response code should be 200
```

---

## API Testing Quick Reference

```
┌───────────────────────────────────────────────────────────────────┐
│ What You Want                    │ Step to Use                    │
├───────────────────────────────────────────────────────────────────┤
│ Set base URL                     │ Given the API base URL is "X"  │
│ Basic Auth                       │ Given I am authenticating as   │
│                                  │   "user" with "pass" password  │
│ Set single header                │ Given I set header "X" with    │
│                                  │   value "Y"                    │
│ Set multiple headers             │ Given I set the following      │
│                                  │   headers: (table)             │
│ Set request body (inline)        │ Given I set the request body   │
│                                  │   to '{"key":"value"}'         │
│ Set request body (table)         │ Given I set the request body   │
│                                  │   with: (table)                │
│ GET request                      │ When I send a GET request      │
│                                  │   to "/path"                   │
│ POST with JSON doc string        │ When I send a POST request     │
│                                  │   to "/path" with body: """    │
│ POST with data table             │ When I send a POST request     │
│                                  │   to "/path" with values:      │
│ POST with form data              │ When I send a POST request     │
│                                  │   to "/path" with form data:   │
│ Check status code                │ Then the API response code     │
│                                  │   should be 200                │
│ Check body contains text         │ Then the API response should   │
│                                  │   contain "text"               │
│ Check body NOT contains          │ Then the API response should   │
│                                  │   not contain "text"           │
│ Check valid JSON                 │ Then the response should be    │
│                                  │   valid JSON                   │
│ Check JSON property exists       │ Then the JSON response should  │
│                                  │   have property "x"            │
│ Check JSON property value        │ Then the JSON response should  │
│                                  │   have "x" equal to Y          │
│ Check JSON subset                │ Then the API response should   │
│                                  │   contain json: """..."""      │
│ Check response header            │ Then the response header "X"   │
│                                  │   should contain "Y"           │
│ Set placeholder                  │ Given I set placeholder "X"    │
│                                  │   to "Y"                       │
│ Debug response                   │ Then print API response        │
└───────────────────────────────────────────────────────────────────┘
```

---

## Chapter Summary

- webship-js API testing uses axios, not the browser -- fast and precise
- Every HTTP method is supported: GET, POST, PUT, PATCH, DELETE
- Four ways to send request bodies: inline JSON, doc string, data table, form data
- Basic Auth, Bearer Token, and custom headers are all supported
- Response assertions cover: status codes, body text, JSON properties, headers
- Nested JSON properties use dot notation: `"user.address.city"`
- `contain json` does subset matching -- extra properties are ignored
- Placeholders let you reuse values across steps in a scenario
- API state is completely reset between scenarios
- Use `Background` for common setup (base URL, authentication)
- `print API response` for debugging -- remove before committing
- Test the complete CRUD lifecycle: Create → Read → Update → Delete
- Test error cases: invalid data, missing auth, duplicate resources

Next: Multi-Browser Testing -- running your tests across Chromium, Firefox, and WebKit.
