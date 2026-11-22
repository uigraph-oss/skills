# Walkthrough Videos

A walkthrough is a narrated video that explains one flow, mostly with the repository's own
artifacts. A scene that names a diagram, an endpoint or a table holds no picture: it is drawn from
that artifact every time the video is made. Rename a component and the next
render shows the new name.

Write one when the user asks for a video, a walkthrough, a narrated explanation, or a tour of a
flow. Offer one unasked only for a flow with a diagram. When asked for one where the flow has no
artifacts, build it from template scenes with what the repository and the user state.

When priorities pull apart, facts and what the user asked come first, then the brand, then the
design defaults below.

## Ask, then show a plan

1. Read the flow in the repository, and the artifacts a video could show.
2. Ask only for what the video can't be made well without and you couldn't find, like numbers,
   screenshots, who it's for when that's unclear, or a brand asset they want. One question at a
   time, two at most, each with 2 to 4 short choices.
3. Show the plan and wait for a yes: the title, the brand it will use, and one line per scene
   saying what it shows and covers.
4. Write the file, check it, and list it in `.uigraph.yaml`, along with any artifact it shows
   that isn't listed there yet.

When the user says to go ahead or not to ask, skip steps 2 and 3, choose sensibly and say what you
assumed. A small change to an existing video needs no plan.

## Where the file goes

Anywhere in the repo; `.uigraph/videos/<name>.yaml`, beside the artifacts it explains, is a good
default. List its path under the top-level `videos:` section of `.uigraph.yaml`, like a diagram or a
doc:

```yaml
videos:
  - .uigraph/videos/login-and-sso.yaml
```

The CLI names the video by its `title:`. Videos are not `docs:` entries, and a file that is not
listed reaches nothing.

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
  services: [UIGraph Gateway]       # only for a flow that crosses services; see below
scenes:                             # required, at least one
  - diagram: { focus: [User, Session handler, Postgres] }
    say: Three parts are involved when someone signs in with a password.
```

An opening card is always generated, from `intro`, else `description`, else `title`, so don't
start with a `title` scene. A closing card is optional: `recap:` (up to 6 short lines) or `outro:`
(what is said over them) adds one. Use it when the points are worth repeating, and leave both out
when the last scene already lands the ending.

`uses:` names the default artifact for every scene. A scene that needs a different one sets `api:`
or `db:` on itself.

## A flow that crosses services

Only artifacts from this repo's own service are available by default, because that is the service
the video is saved under. A flow that crosses a boundary -- a request arriving at the gateway and
ending in this service's database -- names the other services it needs:

```yaml
uses:
  services: [UIGraph Gateway]
  diagram: request-lifecycle
```

Use the service's name as the workspace has it (`service.name` in its own `.uigraph.yaml`); a slug
like `uigraph-gateway` is matched too. This service is always loaded and always comes first, so you
never list it, and a name two services share -- a `users` table in both -- resolves to this
service's. Artifact names stay plain: there is no `service/artifact` form.

One thing you cannot check from here: whether those names exist. They are in another repo, so
`uses.services` is the one place a name you cannot verify is allowed. Name a service you have seen
in the workspace, and keep the list to the services the flow genuinely passes through.

Diagrams follow the same rule: the player has only the diagrams linked to these services. A
diagram of this service can draw another service's components without `uses.services`, but a
diagram, endpoint or table that belongs to another service needs it.

## Scene kinds

**One scene shows one thing.** A scene with two kinds is an error that says to split it.

| Key | Reads | Write it as |
|---|---|---|
| `diagram` | a synced architecture or sequence diagram | a name, or `{ name?, focus?, highlight?, trace?, messages?, draw? }` |
| `endpoint` | the OpenAPI, GraphQL or proto spec | `POST /api/v1/orders`, `mutation createOrder`, `OrderService/CreateOrder` |
| `table` | the database schema | the table name, with `highlight: [col, col]` |
| `template` | nothing — a built-in or workspace template | see [Templates](#templates) |

Don't use the map kinds `frame`, `journey` and `changes`: the workspace player doesn't load maps,
so they play as placeholder cards. Show screens with the `screenshot`, `demo` or `gallery`
templates and image links the user gave.

Every scene also takes `say`, and optionally `id`, `title`, `transition`, `layout`, `style`.

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
`say`.

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
walkthrough of **6 to 8 scenes** lands around 90 seconds, the right length unless the user wants another.

Setting `duration` by hand is almost always wrong: too short and the compiler warns that the voice
will be cut off, too long and the screen sits silent.

## Brand and style

Start from the workspace's default style, which a video gets by setting nothing. When the user
wants another brand or look, use the first of these that fits:

1. A saved workspace style, by its ID (`video: { style: <id> }`, or `<id>:<variation>` for one of
   its variations). The app resolves it; linting ignores it.
2. A built-in look (`video: { look: blueprint }`): `graphite`, `clean`, `aurora`, `blueprint`,
   `minimal`, `cinema`, `bright`.
3. Their brand in this file, from colours, fonts and a logo they gave or the repository has. Don't
   invent a brand or a logo URL. Add `video: { style: graphite }`, or another built-in look, so
   nothing of the workspace's own brand carries over.

The CLI syncs no style or template file, so a brand that many videos share belongs in
**Catalog → Video templates** as a style, and a brand for this video belongs in this file. The
cascade, later winning:

```
engine defaults → built-in look → workspace style → variation → this file → template → scene
```

Brand goes at the top of the file. Don't put colours on single scenes; UIGraph's checker flags
them. A logo needs its image, as a direct https link or a data URL, and where it shows; `show: all`
puts it on every scene, the opening card included. It has to read on each background it lands on,
so leave out `layout: bold` when the logo is the brand colour.

To mix dark and light scenes, give the video one tone and make templates for the other from
built-in ones with `from:`. Switch tone where the story turns, and set every colour role shown
here so text stays readable:

```yaml
brand:
  name: Acme
  logo: https://example.com/logo.svg
style:
  logo: { show: all, position: top-left, height: 56 }
  background: { kind: solid }
  colors:
    { background: '#101418', surface: '#1b2128', border: '#2c343d', text: '#f7f7f7', muted: '#aab2bd', primary: '#f97316', onPrimary: '#101418' }
templates:
  light-flow:                 # use as `template: light-flow`, with flow's fields
    from: flow
    style: &light
      background: { kind: solid }
      colors:
        { background: '#ffffff', surface: '#f4f5f7', border: '#dfe3e8', text: '#14181d', muted: '#59636e', primary: '#c2410c', onPrimary: '#ffffff' }
  light-line:                 # a big line of text: `title` is its field
    from: title
    style: *light
```

`from:` takes built-in templates only, so diagram, endpoint and table scenes and the seven moments
stay in the video's tone. For a moment in the other tone, use `from: title` for a line or
`from: delta` for a number that moved. Darken a pale brand colour where it's text on white. The
colours above only show the syntax.

## Make it feel designed, not a slide deck

A video of seven identical cards is correct and dull. These are the rules the app's own video
writer follows; follow them too, so a walkthrough written in the repo looks like one written in
Studio.

- **Open with a hook.** A `statement`, a `stat`, an `announce`, a `quote`, a `pair`, or straight
  into the diagram. Don't open on a list.
- **Use a moment where the content has one.** `stat` for a number that matters, `quote` for
  someone's words, `before-after` for a change, `pair` for two options, `announce` for something
  new, `statement` for the one idea to remember. Most videos should have at least one.
- **Show something in at least a third of the scenes** — a diagram, endpoint, table, code,
  sequence or data scene — whenever the repository has one.
- **Avoid the same template twice in a row.** Walking one diagram across several scenes is
  fine, as are a how-to's consecutive steps: never drop or merge real content for variety. Use
  `bullets` at most twice; prefer
  `checklist`, `flow`, `pair` or `step` for lists.
- **When something passes between parties** — requests, redirects, callbacks, tokens, events — use
  a `sequence` scene rather than `flow`, so it draws them talking.
- **Show numbers as data**: `dashboard` for several, `chart` over time, `breakdown` for parts of
  a whole, `funnel` for stages, `goals` against targets, `delta` for one that moved.
- **Only use facts and numbers the repository or the user states**, exactly. Never estimate, fill
  in a value or make up a quote.
- **End deliberately**, on a `statement`, a `stat` or the diagram. With a `recap`, no `summary`
  scene as well.

By purpose: an explainer says why it matters, walks how it works, shows the pieces that matter and
ends on what to remember. A how-to gives each step its own scene, in order, with a `callout`
where there's a real warning. A pitch opens on the problem or the result and shows the product
doing the work.

A `step` scene acts its step out, and fills what you leave empty with its own claims, like "All
checks passed" or "Proof attached to the record". So use one only with the true detail its kind
draws: `collect` a `field` with its value, `decide` its `choices`, `check` its `evidence`, `action`
its `result` (it also says the viewer runs it), `verify` its `result` (it also says verified). For
any other step use a `statement`, or a `from: title` line, with `eyebrow: Step 2 of 4`.

These vary a single scene without touching the brand:

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

Where an artifact can carry the scene, show the artifact: its picture is real. Templates carry the
rest, and the moments that make a video worth watching.

The seven moments, with their fields (`?` is optional):

| Template | Fields | For |
|---|---|---|
| `statement` | `text`, `eyebrow?` | the idea to open on or to remember |
| `stat` | `label`, `value`, `change?` | a number that matters |
| `before-after` | `title`, `before: [...]`, `after: [...]` | a change |
| `quote` | `text`, `by?` | someone's real words |
| `announce` | `title`, `text?` | something new |
| `checklist` | `title`, `items: [...]` | things to have or check |
| `pair` | `title`, `left`, `left_text`, `right`, `right_text` | two options or sides |

```yaml
- template: sequence
  title: Signing in
  actors: [Browser, API, Postgres]
  messages:
    - { from: Browser, to: API, text: Sends the password, when: browser sends }
    - { from: API, to: Postgres, text: Looks the user up, when: looks the user }
  say: The browser sends the password, and the API looks the user up before it trusts anything.
```

A `when:` is a few words from that scene's `say`, so the item lands as they are spoken.

The other built-ins include `flow`, `step`, `bullets`, `callout`, `code`, `terminal`,
`screenshot`, `decision`, `chart`, `delta`, `incident`, `trace`, `states`, `access` and
`error-budget`. With the UIGraph MCP connected, `video_authoring_guide` has each one's fields and
an example. `request-path`, `service`, `impact` and `api-change` can't be used in a walkthrough:
their fields clash with scene keys.

A workspace can add six packs of templates: `onboarding`, `releases`, `incidents`, `security`,
`sales`, `support`.

For a shape none of these has, a template built from blocks must be made in **Catalog → Video
templates → New template**, or with the MCP's `create_video_template`. Inside a video file only
`from:` templates work.

## When an artifact is missing

A scene naming something that does not exist becomes a plain card carrying a warning with the file
and line, and the rest of the video still plays. Such names are warnings, so linting still exits 0
with them. The messages say what to fix:

- `There's no diagram called "checkout". Check .uigraph/diagrams.`
- `"users" has no column called "ghost".`

Treat every one as a mistake in the file, not an acceptable gap. Only name artifacts you have read
in `.uigraph.yaml` and under `.uigraph/`.

## Check it before you finish

`uigraph-cli sync` doesn't check what is in a video, and the local viewer (`npx uigraph`) doesn't
play videos, so a broken file syncs clean and fails when someone plays it. Run the linter from
`@uigraph-oss/video-cli` (on GitHub Packages) on it:

```sh
uigraph-video lint .uigraph/videos/<name>.yaml
```

It checks the file, and its diagram, endpoint and table names against this repository. It can't
see other repositories, so names reached through `uses.services` are reported as missing. With the
UIGraph MCP connected, `validate_video` with the service's ID checks those against what is synced.

When neither can run, tell the user it wasn't checked by a tool, and check by reading:

1. **Every name resolves.** For each scene, find the diagram under `.uigraph/diagrams/`, the
   endpoint in the API spec `.uigraph.yaml` points at, and the table in the schema file.
2. **Node and message names match the diagram exactly.** `focus:` and `trace:` take node labels as
   written in the `.mmd`; `messages:` are message numbers, counted in order from 1.
3. **Word counts.** 20 to 35 words per `say:`, nothing over 45.
4. **Valid YAML.** Quote any text containing `: `, and `walkthrough:` is lowercase letters,
   numbers and dashes.

Either way, the file must be listed under `videos:` in `.uigraph.yaml`. Then show the user the file
and say what each scene shows, so they can correct a wrong choice before it is rendered.

## Syncing

```sh
uigraph-cli sync --enterprise      # UIGraph Cloud
uigraph-cli sync --api-url <url>   # self-hosted
```

The video appears in Library, where it is narrated and rendered. Narration audio is cached by what
it says, so re-syncing an unchanged file re-records nothing.

## A complete example

Where the repository has these artifacts and states the thirty days:

```yaml
walkthrough: login-and-sso
title: How login and single sign-on work
description: What happens between "Sign in" and a session, for passwords and for SSO.
uses:
  diagram: login-and-sso
  api: uigraph-api
  db: uigraph

scenes:
  - template: statement
    layout: bold
    text: Two ways in, one kind of session.
    say: "People sign in with a password or through their company's provider, and both roads end at exactly the same kind of session."

  - diagram: { focus: [User, Session handler, Postgres] }
    transition: zoom
    say: "A password sign-in has three parts. The browser, the session handler in the API, and Postgres, where the account and its session live."

  - endpoint: POST /api/v1/auth/login
    say: "The browser posts the email and password here. The right pair gets a session token back, and a wrong password gets a four-oh-one."

  - table: user_sessions
    highlight: [token_hash, expires_at]
    say: "Each session is one row here. Only a hash of the token is stored, so reading this table doesn't let anyone sign in as someone else."

  - diagram: { focus: [OIDC or SAML provider] }
    transition: slide
    say: "Single sign-on adds one more part, the company's provider. The API checks the provider's answer itself before it trusts anything that answer claims."

  - template: stat
    layout: left
    label: Every session lasts
    value: 30 days
    say: "From there the rules are the same for everyone. Whichever way they came in, the session lasts thirty days and then expires."

  - template: statement
    text: The way in changes. The session doesn't.
    transition: fade
    say: "So when you're debugging access, start from the session. How someone signed in only matters up to the moment that row was written."
```

## Common mistakes

| Mistake | Instead |
|---|---|
| Inventing colours, a logo or a brand | The workspace style, or the brand the user or repository gives. |
| Dropping the brand the user asked for | Set it at the top of this file. |
| Seven cards of words | Show an artifact, sequence or data scene in at least a third of them. |
| One diagram scene with a 60-word `say` | Several scenes, each moving the camera somewhere new. |
| Setting `duration` to pace the video | Write more or fewer words; the voice sets the length. |
| Writing the file but not listing it under `videos:` | Only listed files sync. Do not add it under `docs:`. |
| Explaining the whole system | One flow per video. |
| Naming an artifact you did not read | Only diagrams, endpoints and tables that exist. |
| Naming another service's artifact without `uses.services` | List the service, or the scene is a warning card. |
