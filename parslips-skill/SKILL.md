---
name: parslips-skill
description: >-
  Drives the Parslips Eclipse plugin's dev server (localhost:9485) and a running
  WebObjects or ng-objects app's dev endpoints, so disk edits take effect and the app
  can be observed. Use after editing any .java, .html or .wod file (Eclipse misses disk
  edits until refreshed); when the app still shows old behavior; to validate a template
  without rendering it; to create or import a project; to start, stop or restart an app;
  to see what's running and on which port; to read the app's log or startup console;
  when a request to the app hangs; to check whether a change hot-swapped; to look up an
  element's bindings ("what can I bind on WOPopUpButton?") or a component's; to resolve
  a keypath, find where a key is declared or which templates use a component; to rename
  a component, key or WOD element across Java and templates; or to run Java inside the
  live app. Applies to projects with .wo bundles, <wo:...> or <webobject> tags,
  build.properties with project.base, WOComponent/NGComponent classes, or
  wonder-slim/ERExtensions.
---

# Parslips

You're editing a WebObjects / ng-objects project whose developer runs Eclipse with the
**Parslips** plugin (the editor for the Parsley template language). Parslips exposes HTTP
hooks that let you — an agent editing files on disk — make changes take effect, validate
templates, launch and observe the app, and read its output, without the human relaying
anything by hand.

## The rules

Eclipse only tracks edits made through its own editor. A file you change on disk sits
unnoticed: no recompile, no resource copy, and the running app keeps its old state. From
the outside your edit appears to do nothing.

**After editing ANY project file, run `/refreshProject`. Templates, `.wod` files and
static resources too, not just Java.** Templates and resources aren't compiled, but the
app reads them from the output folder (`target/classes`), which Eclipse only refreshes
when it notices a change — and a disk edit is never noticed until you refresh. Assume
nothing has changed until you have.

**Work where Eclipse looks.** The dev server knows only the projects in the Eclipse
workspace. Edit a copy anywhere else (a git worktree, a second clone) and every call still
answers `ok` while nothing you did takes effect: refresh, validation and the running app all
use the workspace's checkout, and the developer can't follow your work in Eclipse. Before
your first edit, `GET /where?path=$PWD`; it must say `inWorkspace:true`. For a worktree it
names the directory to work in instead (`workIn`). If a task really needs an isolated branch,
ask the human how they want it done; don't quietly work elsewhere.

**Hand the workspace back settled.** The moment you stop, the human will open a
component, run the app, or launch it from Eclipse — and they assume the workspace
matches the disk. Never hand back a mid-state: unrefreshed edits, half-copied resources,
an unbuilt change, or errors you meant to fix later. Before you finish a task, pause for
their input, or report back, copy this checklist into your response and tick it off:

```
Handing back:
- [ ] every edit I made is in a workspace project (/where said inWorkspace:true), not a worktree or copy
- [ ] /refreshProject?project=NAME for EVERY project I touched (a dependency counts as much as the app) — each answered `ok` (anything else: not done)
- [ ] /problems?project=NAME on each of them — no errors I introduced
- [ ] any template I edited validates clean (/validate?component=NAME)
- [ ] any app that was running when I started is running again
- [ ] anything I had to leave broken is named in my report
```

## When to do what

| You just… | Do this |
|---|---|
| About to edit, in a directory you haven't checked | `GET /where?path=$PWD` — `inWorkspace:true`, or edit where `workIn` says (a worktree is invisible to Eclipse) |
| Edited **anything** in the project | `GET /refreshProject?project=NAME` — always, first. Check the response (below). |
| Finished, pausing, or reporting back | `GET /refreshProject?project=NAME` for **every** project you touched, each answering `ok`, then `GET /problems?project=NAME` on them — leave the workspace settled for the human (see "Hand the workspace back settled") |
| Edited a template (`.html`/`.wod`) | …then `GET /validate?component=NAME` — refresh makes it take effect, validate catches mistakes; they're separate |
| Want to know what's running | `GET /status?app=NAME` — one entry **per launch config** of the project; read the one with `running:true` |
| Need the app's port, framework, or readable dependencies | `GET /apps?name=NAME` — port, `runtime` (`ng`/`wo`, picks the endpoint URL form), and the dependencies whose source is open in the workspace |
| Need a **new** project | `GET /createProject?name=NAME&template=ng-objects-app` (or `wonder-slim-app`, or `maven` for a plain library) — generates it, imports it into Eclipse, makes it launchable; add `&launch=true&waitForPort=1200` to end with a running app. **Never hand-write a project skeleton** |
| Have a project on disk that Eclipse doesn't know | `GET /importProject?path=/abs/dir` — m2e import, no wizard; an application gets a launch config |
| Need to start the app | `GET /launch?app=NAME&waitForPort=PORT` — blocks until it answers or provably failed |
| Launch refused: `port N is in use by "X"` | Another app holds the dev port. **Default: take it** — `…&stopOthers=true` stops X and launches yours. Need both running? `…&port=1201` runs yours alongside (any free port; then use that port in `log`/`eval` URLs) |
| Workspace cold (projects closed) | `GET /launch?config=NAME&open=true&waitForPort=PORT&timeout=300` — opens the project + its workspace dependencies, clean-builds, launches, waits. Never try to open "everything" |
| Need a restart (classpath change, broken swap, wedged reload) | `GET /restart?app=NAME&refresh=PROJ1,PROJ2&waitForPort=PORT` — stop, rebuild, launch, wait, in one call |
| Launch refused for compile errors in a *dependency* | Run the refusal's `hint` (`/refreshProject?project=DEP&clean=true`), then retry — stale build state is the usual cause |
| Launch/startup failed, app died, or app answers 500 to everything | `GET /console?app=NAME&tail=200` — the Eclipse console, kept after the process dies; read its header first (exit, end time, port-clash note) |
| Every route 404s (even `/eval`, `/log`) but `/` renders | The app is on the wrong HTTP adaptor — grep `/console` for `WOAdaptor=`. `wo-adaptor-jetty` (0.12.0+) selects itself when it is a **dependency of the project**, so check the pom has it; no property needs setting. If it's there and still not chosen, something pins another adaptor: a `WOAdaptor=` line in the project's `Properties` or `~/WebObjects.properties`, or a `-WOAdaptor` launch argument. Not the app's route table |
| A request to the app times out or hangs | `GET /threads` — a thread the debugger stopped (breakpoint, exception, compile error), with where and why; then `GET /dialogs`. Fix, then `/threads?resume=all` |
| A call hangs, or a wait ends `blocked by a modal dialog` | `GET /dialogs` — Eclipse's modal dialogs (title, message, buttons); `?press=BUTTON` answers one. **Check this before "fixing" anything** |
| Suspect compile errors | `GET /problems?project=NAME` — the Problems view as JSON (`count` is the true total; entries carry a `source`) |
| App slow/frozen only under Eclipse | `GET /breakpoints` — a forgotten breakpoint; `?skipAll=true` disarms them all |
| Need what the app logged | `GET …/<App>.woa/log` (WO) or `…/ng/dev/log` (ng), `?contains=…&tail=…` — port and runtime from `/apps` |
| Need an element's real bindings | `GET /elementApi?element=NAME&project=NAME` — the editor's resolved API as JSON; don't reverse-engineer the Java |
| About to work on a component | `GET /context?component=NAME` — files, class, what it takes, its current problems, the elements its template uses (with their bindings) and who uses it, in one call |
| Need what one of the project's components takes | `GET /componentApi?component=NAME` — its declared API, or its settable keys with types |
| Need what a keypath is, or why it's invalid | `GET /keypath?component=NAME&keypath=a.b.c` — type and declaration per hop; the broken key and suggestions |
| Need where a key is declared or used | `GET /find?component=NAME&key=KEY` — declaration, template uses, Java references; for a model class's key, `class=pkg.Team` instead of `component=` finds every template keypath reaching it. Who uses a component: `/callers?component=NAME` |
| Renaming a component, a key or a WOD element | `GET /rename?kind=component\|key\|element&…&preview=true`, then without `preview` — Java, HTML, WOD and call sites in one step; a model class's key with `kind=key&class=pkg.Team`. **Never rename by hand across files** |
| `/validate` reports a missing or misspelled key | `GET /quickfix?component=NAME` lists the editor's fixes; `&problem=ID&fix=ID` applies one (replace, or create the key/action) |
| Need which elements exist | `GET /elementRegistry?project=NAME&filter=TEXT` — the project's roster, with tags and what aliases replace |
| Want to run code in the live JVM | `GET …/<App>.woa/eval` (WO) or `…/ng/dev/eval` (ng), `?snippet=…` — a REPL inside the running app |
| Need the binding errors the app rendered | `GET …/<App>.woa/problems` (WO) or `…/ng/dev/problems` (ng) — the inline error boxes as JSON |
| Markers look stale (survive rebuilds) | `GET /revalidate?project=NAME` re-validates every template; `GET /purgeMarkers` removes orphaned legacy markers |
| Picking up after another session | `GET /activity` — every request the dev server handled, with responses; the human can watch live at `/watch` |
| Unsure what this build offers | `GET /` — the self-describing index; trust it over this document when they disagree |

## Probe first

```bash
curl -s http://localhost:9485/
```

A JSON index → you're good. Connection refused → **stop and tell the human**: Eclipse
isn't running or the plugin isn't loaded; don't retry blindly. (A plain `ok` means an
old plugin build with only the classic endpoints — `/refreshProject`, `/validate`,
`/launch`, `/stop`, `/apps`.) The dev server is loopback-only with no auth.

## Reading any answer

Every endpoint says no the same two ways, so one check covers them all:

- **`error`** — your call is wrong (a missing or invalid parameter, named). Fix the call;
  repeating it can't help.
- **`reason`** — the call was fine but **nothing happened** (unknown or closed project,
  port held, nothing to open). Read it: it says what to change. A **`hint`**, when
  present, is the exact call that fixes it — run it, then retry.
- Neither → it happened. A plain `ok` means done; JSON without either key is the result.

HTTP 500 is a dev-server bug (`"internal":true`), not your mistake: report it, don't
retry in a loop.

## Never wait blind

**Put a timeout on every request to the app: `curl -m 15 …`.** The app runs under
Eclipse's debugger, and a thread that stops (a breakpoint, an uncaught exception, code that
didn't compile) never answers. Without a timeout you wait until your own tool gives up,
minutes later, and learn nothing. With one, a timeout is a signal: `GET /threads` shows what
stopped, where and why; `GET /dialogs` shows what Eclipse is asking. Fix the cause, then
`/threads?resume=all`.

Apps you launch through the dev server don't stop on compile errors or uncaught exceptions;
those requests fail fast (a 500, the error in the log), and `/threads` lists them under
`autoResumed`. Breakpoints still stop. Dev-server calls carry their own limits (`timeout=`
on `/launch`); give `/revalidate` and `/launch?open=true` a generous client timeout, and
everything else 30s.

## Traps — the lessons that cost time

**Check the refresh response.** Plain `ok` means the build settled clean. A JSON body
with `buildErrors` means your edit did **not** compile: the app still runs the previous
classes, and exercising it now tests nothing. Fix the listed problems first.

**The build settling is not the swap landing.** `ok` means the `.class` files exist; the
JVM redefines the classes a beat later. Wait ~2–3s after a refresh before exercising a
Java change, and re-exercise once before concluding a swap failed. (Templates don't race
— they're read per render.)

**Recognise a broken hot swap — then restart, don't debug.** This stack (JBR + DCEVM +
HotswapAgent) reloads structural changes live, so restarts are rare. But a swap can
corrupt a class: lambdas get renumbered, an observer or callback registered against the
old numbering breaks, and the app starts failing on *every* request. The signature is a
`NoSuchMethodError`, `NoSuchFieldError` or `AbstractMethodError` in the console —
especially one naming a `lambda$N` method — after a refresh, on code that is fine. The
app's own log endpoint is dead at that point too, so `/console` is your only source.
`/restart` cures it. Don't spend a minute "fixing" the code.

**Never `clean=true` while the app is running.** A clean+full rebuild produces no per-class
delta for the swapper, so the running app stops picking up changes until restarted. Clean
belongs to build recovery and to freshly opened projects (where `/openProject` and
`/launch?open=true` do it for you, safely, because nothing is running yet).

**Never verify or "fix" the loop with Maven.** Eclipse resolves inter-project dependencies
inside the workspace; the local Maven repository plays no part. A `mvn compile` failing
against a stale installed jar is a false alarm — `/problems` is the truth, and `mvn
install` only wastes time and hides that nothing was broken.

**A hang or a timeout is usually a dialog.** Eclipse's modal prompts are invisible to you.
Before diagnosing anything after a request hangs or a wait times out, `GET /dialogs` — and
answer it with `?press=BUTTON`. Dev-server launches never raise Eclipse's own launch
prompts (compile errors, save, switch-to-debug are all decided and reported as data), but
hot-code-replace failures and other tooling still can.

**`reachable` is a port probe, not health.** `/status` and `/apps` report `running` and
`reachable` from a TCP connect. An app whose every request returns 500 (see the broken
swap above) is "reachable". To confirm health, fetch a page or the app's log endpoint.

**`/status?app=NAME` returns every launch config of the project** — one-off main classes
and a Production config included, all sharing the same registered block. Read the entry
whose `running` is true; don't take the first one.

**Know your source reach; never invent framework internals.** When a question crosses
into a framework or dependency (why does `ERExtensions` do X, is the bug in `helium5` or
here), check `/apps?name=APP` first: its `dependencies` are the ones whose source is open
in the workspace, with their on-disk `path`. If it's listed, read the real source there
and fix bugs across that boundary. If it isn't, you don't have its source — say so
plainly rather than guessing at its behavior.

**A clean console and `exit: 1` is usually a port clash, not a crash.** When a second
instance of the same project starts — typically the developer launching from the Eclipse
UI, which bypasses `/launch`'s already-running check — the frameworks stop the earlier
instance cleanly and silently. `/console`'s header says so when it can (`# note: a launch
of "…" started Ns before this one ended`). Don't hunt for a stack trace that isn't there;
check `/status` for what's running now, and relaunch if needed.

**Refused for compile errors nobody can find? Retry once.** Right after opening projects,
m2e is still resolving classpaths and the workspace briefly shows errors that vanish
seconds later. `/launch` waits for those jobs and re-checks before refusing, but if a
refusal still names errors that `/problems` doesn't show, retry once before reaching for
`ignoreErrors=true`. The refusal always lists the problems it counted — read them; only
Java errors ever block a launch, never template markers.

**One dev port, and the app you're working on gets it.** Apps here share port 1200 by
convention, so starting yours while another runs is a clash — `/launch` refuses
explicitly, naming the holder. The rule in this shop: **stop the other app and start
yours** (`stopOthers=true`); the developer expects that, and running several apps is
fine but not the norm. Only when you genuinely need both up — testing an integration,
say — run yours on another port with `port=N` (the argument is injected into an unsaved
copy of the launch config; nothing is edited), and remember that port when you build the
app's `log`/`eval`/`problems` URLs. Never resolve a clash by hand-killing processes when
`stopOthers` will do it cleanly.

**Start new projects through the plugin, not by hand.** `/createProject` produces the
house layout (pom, `build.properties`, Application/Session/DirectAction/Main, the right
resource folders for the runtime), imports it through m2e so the classpath is real, and
creates the launch configuration. A skeleton you write yourself is a project the developer
must then import, fix up and wire a launch config for. `location=` is the *parent*
directory (default: the Eclipse workspace directory) — ask where the developer keeps
projects if it isn't obvious. It refuses to overwrite; an existing directory goes through
`/importProject` instead.

**A new app may be blind on the app side.** A generated project pins the latest *released*
framework, and that release can predate the app-side dev endpoints: then
`…/ng/dev/log` (or `…/<App>.woa/log`), `eval` and `problems` answer 404, and the app never registers, so `/apps`
doesn't list it. That isn't a broken app. Read its output through `/console?app=NAME`, its
state through `/status?app=NAME`, and use the port you launched on (1200 unless you passed
`port=`). Everything on the Eclipse side — refresh, validate, `/elementApi`, hot swap —
works regardless.

**Supporting logic goes in its own project, wired through the workspace.** Create the
library with `template=maven`, then add it to the app's pom as `groupId` = the library's
package, `artifactId` = its name, `version` `1.0.0-SNAPSHOT`. Refresh the library first,
then the app; m2e resolves the dependency inside the workspace (no `mvn install`). Adding
the dependency is a classpath change, so that one time you `/restart`. After that, edits in
either project hot-swap — including new methods in the library.

**Display-only components don't synchronize.** A component that just shows what it's
given should extend `ERXNonSynchronizingComponent` (or `ERXStatelessComponent`) and read its
bindings with `valueForBinding("x")`. A synchronizing component pushes every binding back to
the parent after rendering, so `<wo:Table rows="$league.standings">` fails at render when
`standings` has no setter (the validator warns). The editor reads the `valueForBinding`
names as the component's API, and `/rename` renames them with the key.

**Framework source on disk may not be the version the app runs.** A checkout of wonder-slim
or ng-objects is usually ahead of the release the app's pom pins. Before using an API you
read there, check the dependency's version in the pom; when they differ, the jar is the
truth (`javap -cp ~/.m2/…/ERExtensions-8.0.13.jar er.extensions.routes.RouteHandler`).

**`ExceptionInInitializerError`, then `Could not initialize class`: restart.** A static
initializer that threw leaves its class unusable for the life of the JVM; no hot swap
revives it. Fix the cause, then `/restart`. (eval reports only the outer error: read the
static initializers.)

**Don't launch Production.** `/launch` prefers a `local`/`dev` config and refuses to guess
when ambiguous — `{"launched":false,"candidates":[…]}`. Pick an exact name from the list.

**A JVM the dev server can't reach.** A swap occasionally leaves the JVM holding its port
but answering nothing, and not reacting to a clean `/stop`. `/stop?force=true` needs a
registered pid; when there is none (`/status` says `running:false` while the port is
held), `lsof -nP -iTCP:PORT -sTCP:LISTEN`, `kill -9` the pid, then `/launch`. (A fresh
ng-objects launch in development mode first tries to evict whatever holds its port, so
`/launch` alone sometimes clears a half-dead instance — but a truly wedged JVM ignores
that too, and `kill -9` is the fallback.)

## Validate templates — your highest-value habit

Templates are long strings with weak definition discovery; mistakes hide until render.
After a template edit do **both**: `/refreshProject` (so the change takes effect) and
`/validate?component=NAME` (to catch mistakes before rendering). `problems` empty →
clean; each problem has `severity`, `line`, `charStart`/`charEnd`, `message`, `file`.
`/refreshProject` does not validate, and `/validate` does not make an edit take effect.

## Ask the editor, don't reconstruct

The editor already resolves what you'd otherwise piece together from Java and grep: what a
component takes, what a keypath is at each hop, where a key is declared and used, who uses a
component. Before working on a component, `/context` gives you all of it in one call; the
narrower endpoints (`/componentApi`, `/keypath`, `/find`, `/callers`) answer one question
each. To change names, `/rename` does it through the editor's refactorings: Java (exact, via
JDT), templates and call sites move together, so nothing is left pointing at the old name.
Shapes and details are in the reference.

## Ask what an element can do

Don't read an element's Java to work out its bindings. `/elementApi?element=NAME` returns
them as data: each binding's `pull`/`push` types and `direction`, `required`, `default`,
`deprecated`, plus cross-binding `constraints` with their plain-English `message`, and the
`content`/`unknownAttributes` policies. Names resolve as a template resolves them
(`str` → `WOString` → `ERXWOString`, classic shortcuts included); `resolved` says what the
name became, `kind:none` means no definition exists. `raw=true` returns the `.apiext` XML.

## Two logs, two jobs

- **`/console?app=NAME`** — the raw Eclipse console, captured by the plugin and kept after
  the process dies. Startup failures, `waitForPort` reporting termination, and an app
  broken by a bad swap live only here.
- **The app's log endpoint** (`…/<App>.woa/log` or `…/ng/dev/log`, port and form from
  `/apps`) — filterable with `contains=`, dies with the app. For debugging markers: add
  `log.info("MYDEBUG …")`, refresh, exercise, read back `?contains=MYDEBUG`, remove.

## Inside the app: `eval` and `problems`

`…/eval?snippet=…` (same URL form as `log`) evaluates a Java snippet inside the running
JVM against its live classes and data — a REPL in the process, with a persistent session.
POST long snippets as `text/plain` (form encoding shreds `=` and `&`). `…/problems`
returns the binding-error boxes the app rendered, as JSON: `clear=true`, exercise, read
back — only the errors that exercise produced. Details and shapes in the reference.

## A full iteration

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'                      # any edit → refresh first
curl -s 'http://localhost:9485/validate?component=SomeComponent&project=MyApp'    # template edit → also validate
sleep 2                                                                           # Java edit → let the swap land
curl -s -m 15 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'   # then read what it logged (always -m)
# …and before handing back: every touched project refreshed and clean
curl -s 'http://localhost:9485/refreshProject?project=my-model'                   # the dependency you edited too, not just the app
curl -s 'http://localhost:9485/problems?project=MyApp'                            # count 0 → settled
```

## More detail

[`references/endpoints.md`](references/endpoints.md) is the reference: every endpoint with parameters and response
shapes, the runtime endpoints (`log`, `eval`, `problems`) in full, ng-vs-WO differences,
template conventions, and the developer-side setup (plugin, JBR + HotswapAgent, `/watch`).
