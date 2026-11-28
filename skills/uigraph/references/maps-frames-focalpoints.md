# Maps, Frames, and Focal Points

Maps are visual canvases composed of Frames (pages), Focal Points (interactive nodes), and Components (linked entities).

## Map Structure in .uigraph.yaml

```yaml
maps:
  - name: Auth Map
    description: User authentication flows
    frames:
      - name: Login Page
        description: The login screen
        imagePath: .uigraph/maps/login-page.png
        focalPoints:
          - name: Register Button
            x: 120
            y: 300
            visibility: public
            components:
              - componentId: component_api-contract
                serviceName: Auth Service
                apiGroupName: auth-api
                operationId: registerUser
          - name: Login Form
            x: 200
            y: 400
            visibility: public
            components:
              - componentId: component_test-case-suite
                serviceName: Auth Service
                testPackName: Auth Smoke
              - componentId: component_backend-flow-diagram
                serviceName: Auth Service
                architectureDiagramName: Login Flow
```

## Map Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `description` | string | no | Description |
| `frames` | list | no | List of frames |

## Frame Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `description` | string | no | Description |
| `imagePath` | string | no | Path to background image |
| `focalPoints` | list | no | List of focal points |

## Focal Point Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `x` | float | yes | X position, on the 3000-unit scale below |
| `y` | float | yes | Y position, on the same scale |
| `visibility` | string | no | `public` or `private`. Default: `public` |
| `components` | list | no | Linked components |

### Coordinates are not pixels

The frame canvas places a focal point at `value / 3000 × the image's rendered width`, for `x` and
for `y` alike: both axes are measured against the image's **width**. A point at pixel `(px, py)`
on an image `W` pixels wide is written as:

```
x = px × 3000 / W
y = py × 3000 / W
```

On an 800-pixel-wide screenshot, the pixel `(330, 215)` is `x: 1238, y: 806`. Writing pixels
straight in puts every point at a fraction of its place, bunched in the top-left corner.

Pick each point by reading the screenshot itself, and after the sync open one frame in the app
to check that the dots sit on what they name.

## Component Linking

Each component entry links a focal point to a UIGraph entity.

### Component IDs

| Component ID | Purpose | Required Resolution Fields (if no componentLinkId) |
|--------------|---------|-----------------------------------------------------|
| `component_api-contract` | Link to an API operation | `serviceName`, `apiGroupName`, `operationId` |
| `component_test-case-suite` | Link to a test pack | `serviceName`, `testPackName` |
| `component_support-kb-troubleshooting` | Link to a doc | `serviceName`, `docName` |
| `component_backend-flow-diagram` | Link to an architecture diagram | `serviceName`, `architectureDiagramName` |

### Resolution Priority

1. If `componentLinkId` is provided, use it directly.
2. Otherwise, the gateway resolves using `serviceName` + entity-specific names.

### Example: API Contract Link

```yaml
components:
  - componentId: component_api-contract
    serviceName: Auth Service
    apiGroupName: auth-api
    operationId: registerUser
```

This links the focal point to the `registerUser` operation inside the `auth-api` API group belonging to `Auth Service`.

`operationId` must match the operation ID UIGraph stored for the endpoint, which depends on the
spec:

| Spec | Stored operation ID |
|---|---|
| OpenAPI with `operationId` | that `operationId` |
| OpenAPI or Swagger 2.0 without one | `METHOD /path` as the spec writes the path, such as `POST /auth/login` (no base path) |
| GraphQL | the root field name, such as `services` or `createDoc` |

A link that does not resolve is dropped without an error. With the UIGraph MCP connected, confirm
each one first: `list_api_groups` for the group name, then `list_endpoints` and `get_endpoint`,
whose **OperationID** is the value to copy.

To link a screen to the calls behind it, read them from the frontend code for that screen, such
as its GraphQL documents and REST clients, rather than guessing from the screen.

### Example: Test Suite Link

```yaml
components:
  - componentId: component_test-case-suite
    serviceName: Auth Service
    testPackName: Auth Smoke
```

### Example: Architecture Diagram Link

```yaml
components:
  - componentId: component_backend-flow-diagram
    serviceName: Auth Service
    architectureDiagramName: Login Flow
```

### Example: Doc Link

```yaml
components:
  - componentId: component_support-kb-troubleshooting
    serviceName: Auth Service
    docName: Auth Troubleshooting Guide
```

## Image Upload Behavior

When a frame has `imagePath`:
1. The CLI computes SHA256 of the image.
2. Calls gateway `v1/sync/frame/prepare`.
3. Gateway responds with action: `skip`, `upload`, or `done`.
4. If `upload`, CLI uploads to S3 presigned URL, then calls `complete`.
5. If `skip`, the image is unchanged and no upload occurs.

The stored image becomes a workspace asset with the ID `frame_<frameId>`. A walkthrough video can
show it as `image: asset:frame_<frameId>`; see `references/walkthroughs.md`.

## Renaming, splitting, and removing maps

Sync creates or updates maps and frames by name and never deletes one. Renaming a map, or
splitting one map into several, leaves the old map in UIGraph beside the new ones. Remove it in
the app, from the map's menu on the Maps page, after the new maps have synced. Deleting a map
deletes its frames and focal points and cannot be undone, so confirm with the user first.
