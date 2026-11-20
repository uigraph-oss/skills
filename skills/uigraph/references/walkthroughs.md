# Walkthrough Videos

A walkthrough is a narrated video that explains one flow using the repository's own artifacts. A
scene does not hold a picture; it names a diagram, an endpoint, a table or a screen, and the
picture is drawn from that artifact every time the video is made. Rename a component and the next
render shows the new name.

Write one when the user asks for a video, a walkthrough, a narrated explanation, or a tour of a
flow. Never offer one for a repository whose flow has no diagram, API spec or schema synced: a
scene has nothing to name, and the first job is mapping the flow.

## Where the file goes

`.uigraph/videos/<name>.video.yaml`, beside the artifacts it explains, and listed in
`.uigraph.yaml` under `docs` with `fileType: walkthrough`:

```yaml
docs:
  - name: How login and single sign-on work
    path: .uigraph/videos/login-and-sso.video.yaml
    fileType: walkthrough
    description: What happens between "Sign in" and a session.
```

That entry is what syncs it. A `.video.yaml` left out of `docs:` reaches nothing.

One video explains one flow. Checkout, sign-in, a deploy. A user who needs the whole system needs
a map, not a film.

## The file

```yaml
walkthrough: login-and-sso          # required. lowercase, numbers, dashes
title: How login and single sign-on work   # required
description: What happens between "Sign in" and a session.
uses:                               # the artifacts, by their names in the repo
  diagram: login-and-sso
  api: uigraph-api
  db: uigraph
  map: checkout
scenes:                             # required, at least one
  - diagram: { focus: [User, Session handler, Postgres] }
    say: Three parts are involved when someone signs in with a password.
recap:                              # up to 6 lines, becomes a closing card
  - Passwords and SSO both end in the same 30-day session
```

An opening card is always generated, from `intro`, else `description`, else `title`. A closing card
appears only when `recap:` or `outro:` is given — give one; it is what a viewer remembers.

`uses:` names the default artifact for every scene. A scene that needs a different one sets `api:`
or `db:` on itself.

## Scene kinds

**One scene shows one thing.** A scene with two kinds is an error that says to split it.

| Key | Reads | Write it as |
|---|---|---|
| `diagram` | a synced architecture or sequence diagram | a name, or `{ name?, focus?, highlight?, trace?, messages?, draw? }` |
| `endpoint` | the OpenAPI, GraphQL or proto spec | `POST /api/v1/orders`, `mutation createOrder`, `OrderService/CreateOrder` |
| `table` | the database schema | the table name, with `highlight: [col, col]` |
| `frame` | one screen of a map | a frame name, or `{ name, map?, show? }` (up to 8 focal points by name) |
| `journey` | several screens in order, along the map's links | `{ map?, frames: [A, B, C] }` — 2 to 6 |
| `changes` | one frame before and after | a frame name, or `{ frame, map?, from?, to? }` |
| `template` | nothing — a built-in or workspace template | see [Templates](#templates) |

Every scene also takes `say`, and optionally `id`, `title`, `duration`, `transition`, `style`.

### Moving the camera on a diagram

This is what makes a diagram scene worth watching. Without one of these the whole diagram sits
still on screen while the voice talks past it.

```yaml
- diagram: { focus: [User, Session handler, Postgres] }   # zoom to these nodes
- diagram: { messages: 1-5 }        # a sequence diagram's messages, 3 or 2-5
- diagram: { highlight: [Postgres] }                      # point at them in place
- diagram: { trace: [Browser, API, Postgres] }            # follow a path, 2+ nodes
- diagram: { draw: page }           # as the Diagrams page draws it, not animated
```

Use several scenes on one diagram, each moving somewhere new, rather than one scene with a long
`say`. The example below spends five of its seven scenes on a single diagram.

### Pointing at fields

`highlight` on an `endpoint` or `table` scene names the columns, parameters or fields to pick out:

```yaml
- table: user_sessions
  highlight: [token_hash, expires_at]
  say: Each session is a row here. Only a hash of the token is stored.
```

### Showing a real call

`example:` on an endpoint scene names a synced API test case by its title, and the scene shows that
call's real request and response instead of the spec's shape.

## Writing the narration

`say:` is spoken, not read. This is where first attempts go wrong, because the habits of good
written documentation work against you.

- **Short sentences.** One clause where you would write two.
- **No bullet grammar.** A list read aloud sounds like a list read aloud.
- **No "as you can see".** The viewer can see. Say the thing.
- **Spell out what a voice would stumble on.** "a four-oh-one", not "a 401".
- **Never narrate the artifact.** "This diagram shows the login flow" wastes the scene. Say what
  the flow does.

**Narration sets the pace.** A scene is as long as its line takes to speak: roughly
`0.5s + words ÷ 160 × 60 + 0.9s`. So 25 words is about 11 seconds, and the way to lengthen a scene
is to say more, not to set `duration`.

Aim for **20 to 35 words a scene**. Over 45 is a warning ("a lot for one scene"); split it. A whole
walkthrough of **6 to 8 scenes** lands around 90 seconds, which is the right length.

Setting `duration` by hand is almost always wrong: too short and the compiler warns that the voice
will be cut off, too long and the screen sits silent.

## Styling: do not set one

**Write no style at all.** This is the single biggest thing that keeps videos looking good.

Every workspace has a default style — a brand kit, or one of the seven built-in looks (`graphite`
the default, `clean`, `aurora`, `blueprint`, `minimal`, `cinema`, `bright`). A video with no style
is drawn in it, and every video in the workspace matches. A video that sets its own colours or
fonts stops matching the moment the brand changes, and an invented palette is how a video ends up
looking worse than the default.

Styles are made in the app, under **Catalog → Video templates**: start a new style, upload style
YAML, or build one from a brand guide with AI. They are **not** synced from the repository — the
CLI pushes no style file, so a brand kit written into `.uigraph/` reaches nothing. If the user
wants their brand on their videos, point them at the Catalog; that is one setup for the whole
workspace, not a per-video job.

If the user explicitly asks for a particular look on this one video:

```yaml
video: { look: blueprint }    # a built-in look. The engine reads this.
```

Use `video: { style: <id> }` only when the user names a workspace style — the app resolves it, the
engine on its own ignores it.

A file-level `style:` block (colours, fonts, type sizes) is a last resort for one deliberate effect
the user asked for. It wins over the workspace brand, so it is the layer that breaks consistency.
The full cascade, later winning:

```
engine defaults → built-in look → workspace style → variation → this file → template → scene
```

**Never set colours, fonts or `duration`.** Per-scene variety is a different thing, and it is
allowed — see below.

## Make it feel designed, not a slide deck

A video of seven identical cards is correct and dull. These are the rules the app's own video
writer follows; follow them too, so a walkthrough written in the repo looks like one written in
Studio.

- **Open with a hook.** A `statement`, a `stat`, an `announce`, or straight into the diagram.
  Never open on a list.
- **Use a moment where the content has one.** `stat` for a number that matters, `quote` for
  someone's words, `before-after` for a change, `pair` for two options, `announce` for something
  new, `statement` for the one idea to remember. Most videos should have at least one.
- **Show something in at least a third of the scenes** — a diagram, endpoint, table, code,
  sequence or data scene — whenever the repository has one.
- **Never use the same template twice in a row.** Walking one diagram across several scenes is
  fine, and is usually the best thing a walkthrough can do. Use `bullets` at most twice; prefer
  `checklist`, `flow`, `pair` or `step` for lists.
- **When something passes between parties** — requests, redirects, callbacks, tokens, events — use
  a `sequence` scene rather than `flow`, so it draws them talking.
- **Only use numbers the repository states**, exactly. Never estimate or fill in a value.
- **Don't end with a `summary`** that repeats the closing card. Finish on a `statement`, a `stat`,
  or the diagram.

Four things vary a scene without touching the brand. These are the only style keys to write:

| Key | Values | Use it for |
|---|---|---|
| `layout:` | another of the template's layouts | `stat`, `quote`, `announce` and `statement` each take `left` and `bold`; `checklist` takes `split` |
| `style: { layout: { step: … } }` | `left`, `right`, `center`, `stacked` | moving the visual on step and image scenes; alternate sides |
| `style: { motion: { enter: … } }` | `slide`, `rise`, `scale`, `fade` | changing how things come in |
| `style: { background: { kind: … } }` | `grid`, `dots`, `aurora`, `gradient` | setting one or two moment scenes apart |
| `transition:` | `zoom`, `slide`, `fade` | the start of a new part: zoom into a diagram or a key number |

```yaml
- template: statement
  text: Every session expires after thirty days.
  layout: bold
  transition: zoom
  style: { background: { kind: grid } }
  say: One rule covers both paths, and it is the thing to remember.
```

`layout: bold` turns a scene onto the brand colour. Keep it for one or two moments in a video.
A layout a template does not have is a warning that lists the ones it does.

## Templates

**Prefer a scene that names an artifact.** A walkthrough's value is that its pictures are real; a
template scene is a hand-written card. Use one for the moments an artifact cannot carry — an
opening statement, a number that matters, a comparison, a closing checklist.

When you do, **use the ones that already exist.** The workspace has them all:

- **36 built-in templates**, among them `title`, `bullets`, `callout`, `summary`, `flow`,
  `sequence`, `code`, `terminal`, `screenshot`, `compare`, `decision`, `chart`, `delta`,
  `api-change`, `incident`, `trace`, `states`, `access`, `error-budget`.
- **7 "moments"** for the beats that make a video worth watching: `stat`, `before-after`, `quote`,
  `announce`, `statement`, `checklist`, `pair`.
- **6 packs** a workspace can add — new joiners, releases, incidents, training, customer-facing,
  support.

```yaml
- template: statement
  text: Every session expires after thirty days.
  say: One rule covers both paths.
```

Do not write a custom template to get a video made. Custom templates are YAML block compositions
for a scene shape a team repeats across many videos, and they belong in the workspace Catalog so
every video can use them. Writing one inline in a video file is possible but makes that shape
available to nothing else. If the user wants a template of their own, say that Catalog → New
template is where it goes, and that AI can suggest a set from their existing videos.

**Order of work, when the user wants videos to look like theirs:** set the style once in the
Catalog, add a pack if a template fits their kind of video, then write walkthroughs that set
neither. Not the other way round.

## When an artifact is missing

A scene naming something that does not exist becomes a plain card carrying a warning with the file
and line, and the rest of the video still plays. The errors say what to write:

- `"checkout" has no frame called "Nowhere".`
- `Say which map: add `uses: { map: <name> }` or `map: <name>` here.`
- `"Cart" has no earlier version to compare with.`

Treat every one as a mistake in the file, not an acceptable gap. Only name artifacts you have read
in `.uigraph.yaml` and under `.uigraph/`.

## Check it before you finish

Nothing checks a walkthrough for you. `uigraph sync` only requires a `walkthrough:` key, so a file
with broken scene references syncs clean and fails when someone plays it, and the local viewer
(`npx uigraph`) does not render videos at all. Check it yourself, by reading:

1. **Every name resolves.** For each scene, find the diagram under `.uigraph/diagrams/`, the
   endpoint in the API spec `.uigraph.yaml` points at, the table in the schema file, the frame in
   the map. A name you cannot find is a scene that will play as a warning card.
2. **Node and message names match the diagram exactly.** `focus:` and `trace:` take node labels as
   written in the `.mmd`; `messages:` are message numbers, counted in order from 1.
3. **Word counts.** 20 to 35 words per `say:`, nothing over 45.
4. **Scene count.** 6 to 8. Fewer is a thin video; more is two videos.
5. **Valid YAML**, and `walkthrough:` is lowercase letters, numbers and dashes.
6. **It is listed** in `.uigraph.yaml` under `docs` with `fileType: walkthrough`.

Then show the user the file and say which artifact each scene names, so they can correct a wrong
choice before it is rendered.

## Syncing

```sh
uigraph-cli sync --enterprise      # UIGraph Cloud
uigraph-cli sync --api-url <url>   # self-hosted
```

The video appears in Library, where it is narrated and rendered. Narration audio is cached by what
it says, so re-syncing an unchanged file re-records nothing.

## A complete example

```yaml
walkthrough: login-and-sso
title: How login and single sign-on work
description: What happens between "Sign in" and a session, for passwords and for SSO.
uses:
  diagram: login-and-sso
  api: uigraph-api
  db: uigraph

scenes:
  - diagram: { focus: [User, Session handler, Postgres] }
    say: Three parts are involved when someone signs in with a password. The browser, the session handler in the API, and Postgres.

  - diagram: { messages: 1-5 }
    say: The browser posts the email and password. The handler looks the user up, checks they aren't disabled, compares the bcrypt hash, and creates a thirty-day session.

  - endpoint: POST /api/v1/auth/login
    say: That first request is this endpoint. On success it returns the session token, and a wrong password is a four-oh-one.

  - table: user_sessions
    highlight: [token_hash, expires_at]
    say: Each session is a row here. Only a hash of the token is stored, and it expires after thirty days.

  - diagram: { messages: 6-13 }
    say: Single sign-on starts with a redirect. The API saves the login state and a nonce, sends the person to their provider, and checks that state again when they come back.

  - diagram: { focus: [OIDC or SAML provider] }
    say: The provider is OIDC or SAML. Either way the API validates the answer itself before trusting any claims.

  - diagram: { messages: 18-23 }
    say: Then the same rules apply to everyone. The email domain is checked, the user and their membership are created or updated with the mapped role, and a session is issued.

recap:
  - Passwords and SSO both end in the same 30-day session
  - Only a hash of the session token is stored
  - Login state and a nonce protect the SSO round trip
```

Seven scenes, 96 seconds, no style, no custom template, every picture drawn from an artifact that
is already in the repository.

## Common mistakes

| Mistake | Instead |
|---|---|
| Inventing colours, fonts or a brand | Write no style. The workspace default is already right. |
| Writing a custom template to get started | Use a built-in template, or better, an artifact scene. |
| One diagram scene with a 60-word `say` | Several scenes, each moving the camera somewhere new. |
| Setting `duration` to pace the video | Write more or fewer words; the voice sets the length. |
| Writing the file but not listing it under `docs:` | The entry with `fileType: walkthrough` is what syncs it. |
| Explaining the whole system | One flow per video. |
| Naming an artifact you did not read | Only diagrams, endpoints, tables and frames that exist. |
| Finishing without linting | `video-cli lint`; `uigraph sync` checks nothing here. |
