# Test Packs and Test Cases

Test packs are metadata records defined in `.uigraph.yaml` under `testPacks`. A pack's test cases can live inline under `testCases`, in a separate file referenced by `testCasesPath`, or both (the two lists are merged). Use `testCasesPath` to keep the main `.uigraph.yaml` small when a pack has many cases.

UIGraph test packs are not generated test files. They are not Vitest, Jest, Pytest, PHPUnit, or other project test framework tests.

- Use test packs to describe API checks or manual/user-flow checks for UIGraph sync.
- Prefer API test cases linked to OpenAPI `operationId`s when API evidence exists.
- Use manual test cases for flows that cannot be represented as API tests.
- Do not inspect project test files and translate them into UIGraph test packs unless the user explicitly asks.
- Do not generate project test framework files unless the user explicitly asks for project tests outside UIGraph artifacts.

## Test Pack Structure

```yaml
testPacks:
  - name: Adapter Smoke
    type: smoke
    environment: staging
    releaseLabel: v1.2.0
    testCases:
      - ...
```

### Test Pack Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `type` | string | yes | `smoke`, `regression`, `manual` |
| `environment` | string | no | Environment label |
| `releaseLabel` | string | no | Release tag |
| `testCases` | list | no | List of test cases |
| `testCasesPath` | string | no | Path to a YAML file holding a `testCases:` list; merged with inline `testCases` |

### External Test Cases File

The referenced file holds a top-level `testCases:` list only (no `testPacks` wrapper). Its cases are merged with any inline `testCases` on the pack. See `assets/templates/external-test-cases.yaml` for a full template.

```yaml
# .uigraph/tests/adapter-smoke.yaml
testCases:
  - title: Login returns 200
    type: api
    order: 2
    apiGroupName: public-api
    operationId: loginUser
    expectedStatusCode: 200
```

## API Test Case

```yaml
- title: Register user returns 201
  type: api
  order: 1
  priority: p0
  tags:
    - auth
    - registration
  apiGroupName: storefront-api
  operationId: registerUser
  expectedStatusCode: 201
  mapName: Auth Map
  frameName: Login Page
  focalPointName: Register Button
```

### API Test Case Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | Must be `api` |
| `title` | string | yes | Test case title |
| `order` | float | yes | Execution order |
| `description` | string | no | Long description |
| `priority` | string | no | `p0`, `p1`, `p2`, `p3` |
| `tags` | list | no | String labels |
| `linkedTicket` | string | no | External ticket ID |
| `estimatedDurationMins` | int | no | Estimated minutes |
| `testOwner` | string | no | Owner email or name |
| `apiGroupName` | string | no | If set, must match an `apis[].name` |
| `operationId` | string | no | If set, must match an `operationId` in the OpenAPI spec — sync fails if it does not resolve |
| `expectedStatusCode` | int | no | Expected HTTP status |
| `requestTemplate` | string | no | Request body template |
| `responseTimeMs` | int | no | Max response time in ms |
| `responseBody` | string | no | Expected response body |
| `assertions` | list | no | List of `{field, type, value}` |
| `mapName` | string | no | Map to link to |
| `frameName` | string | no | Frame to link to |
| `focalPointName` | string | no | Focal point to link to |
| `screenshots` | list | no | Reference image paths; see [Reference Screenshots](#reference-screenshots) |

## Manual Test Case

```yaml
- title: Verify login flow in UI
  type: manual
  order: 3
  description: Step-by-step login verification
  priority: p1
  tags:
    - ui
    - manual

  # Link to map focal point
  mapName: Auth Map
  frameName: Login Page
  focalPointName: Login Form

  stepsList:
    - action: Navigate to login page
      expectedResult: Login form is displayed
    - action: Enter valid credentials
      expectedResult: User is redirected to dashboard
    - action: Check session cookie
      expectedResult: Session token is set

  expectedOutcome: User successfully logs in and session is established
  preconditions: User account exists and is active
  testData: Use test account testuser@example.com / TestPass123
  postconditions: User session is active
  requiresEvidence: true
  isCritical: true
```

### Manual Test Case Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | yes | Must be `manual` |
| `title` | string | yes | Test case title |
| `order` | float | yes | Execution order |
| `stepsList` | list | no | List of `{action, expectedResult}` |
| `expectedOutcome` | string | no | Overall expected result |
| `preconditions` | string | no | Setup required before test |
| `testData` | string | no | Data to use during test |
| `postconditions` | string | no | Expected state after test |
| `requiresEvidence` | bool | no | Whether screenshot/log is required |
| `isCritical` | bool | no | Whether test is critical |
| `screenshots` | list | no | Reference image paths; see [Reference Screenshots](#reference-screenshots) |

## Reference Screenshots

Any test case, `api` or `manual`, may carry `screenshots`: reference images showing the
expected visual result, attached to the case in UIGraph.

```yaml
- title: Checkout success path
  type: manual
  order: 1
  screenshots:
    - .uigraph/tests/screenshots/checkout-confirmation.png
```

- Each path must point to an existing file, resolved relative to the working directory
  where the CLI runs. A directory is rejected — there is no expansion.
- The extension must be one of `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.svg`. The
  check is on the extension, not the file contents.
- Generated screenshots must live under `.uigraph/tests/screenshots/`.
- Uploads are content-addressed: an unchanged file is skipped on re-sync.
- Do not invent screenshot paths. Reference only images the user already has or has
  explicitly asked to be captured. A path that does not exist fails the whole sync.
- `screenshots` are reference images on the case definition. They are unrelated to
  `requiresEvidence`, which asks a test runner to attach evidence at execution time.

## Linking Rules

- Linking an API test case to the contract is **optional**. Omit `apiGroupName`/`operationId`
  for a test case not tied to a synced endpoint (e.g. one hitting a custom URL).
- If `operationId` is set, it must match an `operationId` inside the referenced OpenAPI spec,
  and `apiGroupName` (if set) must match the `name` of an entry in `apis`. When it resolves, the
  test case is stored with a real link to that API spec (`apiSpecId`) plus the endpoint's HTTP
  method — identical to a case linked from the UI.
- A set `operationId` that does not resolve to a synced endpoint **fails the sync** (it no longer
  silently falls back to `GET`). Sync the API group first, or fix the id.
- `mapName`, `frameName`, `focalPointName` must match entries in `maps`.
