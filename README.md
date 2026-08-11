# requence

The official Python SDK for the Requence platform. This package covers both halves of the integration:

- **`requence.service`** — connect a Python service to Requence and process messages
- **`requence.task`** — start and monitor Requence tasks programmatically

## Requirements

Python 3.12 or later.

## Installation

```bash
pip install requence
```

---

## Service

A **service** is a program that connects to Requence, receives messages, processes them, and returns results. Services are the building blocks of every task template.

### Authentication

Every service needs an **access token**. Copy it from the **Services** list view in the Requence UI by clicking **Copy credentials**.

The token is resolved in this order:

1. `access_token` key in the config dict passed to `Service()`
2. `REQUENCE_SERVICE_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.service_access_token` or `requence.access_token` in `pyproject.toml`

```bash
REQUENCE_SERVICE_ACCESS_TOKEN=your-token python main.py
```

### Basic Usage

```python
from requence.service import Service

def handler(ctx):
    return {"message": f"Hello, {ctx.input['name']}!"}

Service("1.0.0", handler)
```

The first argument is the **version** of the service definition you are implementing. The constructor blocks the current thread — Requence delivers messages, the handler runs, and results are sent back automatically.

### Options object

Instead of a bare version string, pass a config dict:

```python
from requence.service import Service

def handler(ctx):
    return process(ctx.input)

Service(
    {
        "version": "1.0.0",
        "prefetch": 5,         # process up to 5 messages in parallel (default: 1)
        "access_token": "...", # overrides env / pyproject.toml
        "ssl_context": ctx,    # optional ssl.SSLContext for TLS connections
    },
    handler,
)
```

For `amqps://` connections, TLS settings can also be overridden with
environment variables (a **fallback** — a passed `ssl_context` takes
precedence):

| Variable              | Effect                                                               |
| --------------------- | -------------------------------------------------------------------- |
| `CA`                  | Inline PEM certificate(s) to trust as the CA.                        |
| `CA_FILE`             | Path to a PEM file to trust as the CA (`CA` wins when both are set). |
| `REJECT_UNAUTHORIZED` | Set to `false`/`0`/`no` to disable certificate verification.         |

### Dev Overlay

When developing locally alongside a deployed service, pass a **dev token** (your personal access token) so Requence routes only your own tasks to the local instance instead of the production pool:

```python
import os
from requence.service import Service

def handler(ctx):
    return ctx.input

Service(
    {"version": "1.0.0"},
    handler,
    dev_token=os.getenv("REQUENCE_DEV_TOKEN"),
)
```

The dev token is resolved in the same order as the access token — `dev_token` argument → `REQUENCE_SERVICE_DEV_TOKEN` / `REQUENCE_DEV_TOKEN` env var → `requence.service_dev_token` / `requence.dev_token` in `pyproject.toml`.

### Two ways to start — and why it matters

`Service(...)` configures the service and then **consumes on the calling thread**, so it never returns while the service is up:

```python
from requence.service import Service

Service("1.0.0", lambda ctx: ctx.input)   # blocks here forever
```

That is the right shape when the service is the whole program. But it means there is **no instance to call methods on** — so `act()` and `close()`, which are both methods, are out of reach. Use `Service.start(...)` when you need either. It is the same configuration with the consume loop on a thread of its own, and it **returns the instance** once connected:

```python
import threading
from requence.service import Service

service = Service.start("1.0.0", lambda ctx: ctx.input)
# ...connected; `service` is usable here

threading.Event().wait()   # keep the main thread alive; see below
```

| | `Service(...)` | `Service.start(...)` |
|---|---|---|
| Returns | never (while up) | once connected |
| `act()` / `close()` | unreachable | usable |
| Bad credentials | raises | raises |
| Consume thread | the caller's | a daemon thread |

`start(..., timeout=30.0)` bounds only the **first** connect; every drop after it reconnects in the background as always. A first connect that never lands raises `TimeoutError` and stops retrying, so you never get back an instance you cannot use.

Its consume thread is a **daemon**, so it never holds the process open. If the service is the whole program, keep the main thread alive yourself — a `threading.Event().wait()`, a web server, a REPL — or just use the blocking `Service(...)`.

### Closing the service

```python
service = Service.start("1.0.0", handler)
# ...later, from anywhere:
service.close()
```

`close()` stops consuming and stops reconnecting. It is safe to call from any thread: the channel work is marshalled onto the connection's own thread.

---

## Context API

Every handler receives a `ctx` object (a `Context` instance).

### Data access

| Attribute | Description |
|---|---|
| `ctx.input` | The input data routed to this service node |
| `ctx.configuration` | The static configuration set on the node in the UI |
| `ctx.task_id` | The unique ID of the current task execution |

### Logging

`ctx.debug` sends log messages to the Requence UI in real time:

```python
def handler(ctx):
    ctx.debug.log("Processing started")
    ctx.debug.info("Step complete", {"step": 1})
    ctx.debug.warn("Something looks off")
    ctx.debug.error("An error occurred")
```

### Flow control

#### `ctx.retry(delay=None)`

Instructs Requence to retry this service after an optional delay in milliseconds (minimum 100 ms). No code executes after this call.

```python
def handler(ctx):
    db = get_db_connection()

    if not db.is_connected:
        ctx.retry(2000)  # retry in 2 seconds

    return db.query("SELECT ...")
```

> **Note:** It is your responsibility to prevent infinite retry loops.

#### `ctx.abort(reason='')`

Instructs Requence to abort this service immediately. If the service node's **on fail** output is not connected, the entire task fails.

```python
def handler(ctx):
    if not ctx.input.get("required_field"):
        ctx.abort("Missing required field")

    return process_data(ctx.input)
```

#### `ctx.skip()`

Puts the message back on the queue without processing it. The next available service instance will receive it.

#### `ctx.to_output(output_name, value)`

Routes the result to a specific **named output** on the service node. Use this when your service definition has multiple outputs:

```python
def handler(ctx):
    if ctx.input.get("type") == "pdf":
        return ctx.to_output("pdf", {"url": "..."})

    return ctx.to_output("other", {"raw": ctx.input})
```

#### `ctx.defer(reason=None)`

Marks the current message as **deferred**. The service acknowledges the message but signals that the result will be delivered later via `service.act()`. Returns a **message key**:

```python
def handler(ctx):
    message_key = ctx.defer("waiting for external process")
    save_to_db(ctx.task_id, message_key)
```

#### `ctx.terminated`

A `threading.Event` that is set when the task is stopped — either cancelled via the UI or API, or terminated by another node. Use it in generator handlers to know when to stop:

```python
def handler(ctx):
    while not ctx.terminated.is_set():
        yield poll_for_updates()
        ctx.terminated.wait(timeout=5)
```

---

## Continuous (Generator) Mode

When a service node is configured in **continuous mode**, the handler can be a Python generator. Each `yield` sends an incremental result; the generator completing signals the end of processing:

```python
from requence.service import Service

def handler(ctx):
    for chunk in fetch_chunks(ctx.input):
        if ctx.terminated.is_set():
            break
        yield {"chunk": chunk}

Service("1.0.0", handler)
```

The generator is stopped automatically when `ctx.terminated` is set. Any yielded values up to that point are still sent.

---

## Deferred Delivery via `service.act()`

After deferring a message with `ctx.defer()`, deliver the result later — even from a different process — using the `act()` method on the `Service` instance.

**Start the service with `Service.start(...)`**, not `Service(...)`: `act()` is a method, and the blocking constructor never returns the object to call it on.

```python
from requence.service import Service

def handler(ctx):
    message_key = ctx.defer()
    save_to_db(ctx.task_id, message_key)

service = Service.start("1.0.0", handler)   # returns
```

```python
# In a webhook handler or background job:
message_key = load_from_db(task_id)

def actor(api):
    api["send"]({"result": "done"})
    # or: api["send_to_output"]("success", {"result": "done"})
    # or: api["abort"]("something went wrong")

service.act(message_key, actor)
```

`act()` may be called from any thread — a web request handler, a scheduler, a durable surface handler. It publishes through the connection's own thread and does not return until everything it published has actually gone out, so a script that calls `act()` and then exits does not lose the answer.

The `actor` callable receives an `api` dict with three keys:

| Key | Description |
|---|---|
| `api["send"](data)` | Send data to the default output |
| `api["send_to_output"](name, data)` | Send data to a named output |
| `api["abort"](error)` | Abort the deferred message with an error string or exception |
| `api["render"](ui, props, surface_id=…)` | Draw or redraw a surface on the deferred node — see [Surfaces](#surfaces--a-ui-of-your-own-on-the-node) |

The actor can also return a value or a generator directly, which behaves like calling `api["send"]()` for each value.

---

## Surfaces — a UI of your own on the node

A service can ship its own UI component and have Requence render it **on the
publishing node** in the task canvas. There is no UI protocol: the contract is
**props**, and a callable inside them arrives in the component as an invokable
function.

The design authority is
[`knowledge/service-ui-surfaces.md`](../knowledge/service-ui-surfaces.md); this
section is the Python API. The TypeScript SDK
(`packages/service/README.md`) is the reference for behaviour — the wire is
identical, and a Python service and a TS service draw the same surfaces.

**The one difference is the typing of the props, not the behaviour.**
`create_ui[Props]` and `RequenceUI[Props]` take a type parameter, but
`ctx.render(ui, props)` accepts `Any`: a wrong prop name or a wrong value type is a
runtime concern here, where TypeScript catches it against `Partial<Props>`. There is
also no type bridge between the service and the component — no `PropsOf`, no
`createSurface` — because a component is always a built JavaScript module (see
below), so its props are declared in TypeScript on the component's side and in
Python on the service's side, and keeping the two in step is yours. Everything else
in this section behaves exactly as the TS README describes it.

```python
from pathlib import Path
from requence.service import Service, create_ui

ui = create_ui(
    # An ABSOLUTE path to the BUILT component. `Path(__file__).parent / …`
    # resolves against this module, not against whatever directory the service was
    # started from.
    Path(__file__).parent / "dist" / "panel.js",
    # Optional default props: what does not change from one message to the next.
    {"dense": True},
)

def handler(ctx):
    ctx.render(ui, {"title": "Processing"})
    return ctx.input

Service("1.0.0", handler)
```

**Requence never bundles.** The component is always a built JavaScript ES module
with one default `mount(element, ctx)` export, whatever language the service is
written in — build it with your own JS toolchain and point `create_ui` at the
result. The bundle is read and hashed once, at import time, and capped at 1 MiB,
so an oversized component fails at startup rather than on its first render in
production.

### `create_ui(path, defaults=None, *, id=None)`

| Argument | Meaning |
|---|---|
| `path` | Absolute path (`str` or `PathLike`) to the built module. A relative path or a URL is refused — the first resolves against the working directory, and Requence reads the component's bytes rather than fetching them. |
| `defaults` | Props restated on every render of this component. Copied one level deep. |
| `id` | The component's **stable id**, defaulting to the built file's name. |

The id is what a durable handler is addressed by, and it is the one thing about a
component a rebuild leaves alone — the content hash deliberately does not. Two
handles claiming one id raise at import time, so two `index.js`-shaped components,
or one file registered twice with different defaults, need explicit ids.

### `ctx.render(ui, props, *, surface_id="default")`

Fire-and-forget, like a data send: the publish is enrolled in the message's ledger,
so a failure fails the node at the settle gate rather than at the call site.

**A render is a patch.** What the surface shows is the *fold* of three layers — the
UI's defaults, everything previously rendered onto that surface, and this render —
so redrawing one field takes one prop and leaves the rest standing:

```python
ctx.render(ui, {"title": "Processing", "at": now()})
ctx.render(ui, {"at": now()})       # still says "Processing"
ctx.render(ui, {"title": "Done"})   # and still carries the last `at`
```

The merge is **shallow**, and nothing is ever removed — an omitted prop means
"nothing to say about this", so a prop is taken back by saying something else about
it (`{"error": None}`). Defaults are the **weakest** layer: they are what a surface
shows until some render says otherwise.

`surface_id` names which surface on the node this render targets; one node can
carry several.

**Prop names are the component's, not Python's.** The props are a wire contract with
a built JavaScript module, so they stay in whatever case that module reads —
`onSubmit`, not `on_submit`. Nothing here renames anything, in either direction, and
the same is true of the argument an interaction comes back with: the component decides
what it sends, and the handler receives exactly that.

### Handler props — where you write the callable decides how long it can be pressed

A callable in a surface's props arrives in the component as an invokable function.
There are **two kinds**, and you pick by *where you write it*:

```python
def on_submit(answer):
    # DURABLE. Module scope, so any instance of this version can run it — this one,
    # or the one that replaces it in a redeploy. An invocation waits in the
    # version's handler queue for ~60s, so the handler below is free to return, and
    # a rolling restart is invisible to the viewer.
    record(answer)

ask_ui = create_ui(HERE / "dist" / "ask.js", {"onSubmit": on_submit})

def handler(ctx):
    ctx.render(ask_ui, {
        "question": "ship it?",
        # CONNECTION-BOUND. Closes over this running handler, so only this process
        # can serve it and it dies when the message settles — which is also why it
        # needs a generator handler.
        "onCancel": lambda _arg: ctx.debug.log("cancelled"),
    })
    yield from ()
```

A **non-generator** handler may not declare a callable in a render's props: its
registry goes the moment it returns, so the reference would be dead on arrival. It
raises with that message and names the fix. A callable in the UI's **defaults** is
durable and registers nothing, so a non-generator handler can publish one freely.

A durable handler gets **no injected context**: it reaches its own stores and
anything else through its own closure, exactly as a module-scope function would in
any other process of the version. Consuming the handler queue is automatic — it is
`service-<name>@<version>-handlers`, one per version, created by Requence when the
version is provisioned and declared again by the SDK on connect.

**A durable handler outlives its message, not its task.** The rule the browser
applies per prop is:

```
reachable(durable)    = the task is running  AND some instance of the version is up
reachable(connection) = …and this surface's connection is alive with the message unsettled
```

So if the handler that rendered simply **returns**, the node completes, the task goes
terminal, and the button is correctly reported unreachable and disabled within
seconds — the surface stays, but there is nothing behind it. A handler that wants its
controls to keep working calls `ctx.defer(...)`, which settles the message (so no
connection-bound prop can survive it) and leaves the task running:

```python
def handler(ctx):
    ctx.defer("waiting for a click")
    ctx.render(ask_ui, {"question": "ship it?"})
    # returns here; the process may now exit and the button still works
```

The argument comes from the **browser** and is forgeable: validate it in the
handler, and keep anything the handler must not be told in its closure rather than
in the props. An interaction is *accepted*, not awaited, so a click carries no
result — feedback is a re-render.

**A thrown handler is visible.** Whatever went wrong — an unregistered component, a
prop holding nothing invokable, a handler that raised — is recorded on the surface
and rendered by Requence beneath the frame, and the invocation is **acked** either
way (a requeue would rerun a side-effecting handler forever).

### Component reload in a dev session

A service connected with a **dev token** watches the built file behind every
`create_ui`. Rebuild it and the frame swaps to the new bundle in place — same
props, same handlers, same node, no re-render and no new task. It reaches every
surface your dev session ever drew and nothing else, so a production surface of
the same component is untouched. Saving the file without changing its content
swaps nothing, and a rebuild over the 1 MiB cap is refused with a warning while
the previous bundle stays live.

The watch is a poll (one `stat` per registered component every 200 ms) rather than
a filesystem-event subscription, so the SDK grows no runtime dependency for a
dev-only feature. It runs on one daemon thread, never holds the process open, and
an unwatchable path degrades the loop to a warning — you still get a rebuild on
restart.

### The deferred form, end to end

A durable handler answering the node it was rendered on is the shape all of this is
for. It needs `Service.start(...)`, because the handler is at module scope and has to
reach the instance:

```python
def on_submit(arg):
    # Durable: module scope, so any instance of this version can run it. The defer
    # key came out in the props and comes back in the argument.
    def actor(api):
        api["render"](ui, {"submitted": True})   # before the send: act() settles it
        api["send"]({"answer": arg["answer"]})

    service.act(arg["deferKey"], actor)

ui = create_ui(HERE / "dist" / "ask.js", {"onSubmit": on_submit})

def handler(ctx):
    # Defer FIRST: there is no key until the message is parked, and deferring is
    # what keeps the task running so the button stays reachable.
    ctx.render(ui, {"question": "ship it?", "deferKey": ctx.defer("waiting")})

service = Service.start({"version": "1.0.0"}, handler)
threading.Event().wait()
```

Kill the process afterwards and the form still works: answering it runs the handler
in whichever instance of the version is free at the time, and an invocation with none
running waits in the version's handler queue. Requence holds nothing.

The argument is the **browser's** and forgeable — the `deferKey` above included. That
is safe because a key is single-use, belongs to that task's own node, and `act()`
refuses anything else. A handler acting on a viewer-supplied amount, id or address
must validate it, and anything the handler must not be told belongs in its closure
rather than in the props.

`test/demo-service/surface_demo.py` is this, runnable.

---

## CLI — `requence-service generate-types`

The package installs a `requence-service` CLI that generates Python type stubs from the schemas you defined in the Requence UI:

```bash
requence-service generate-types
```

Type stubs are written to `typings/requence/service/`. Pylance and pyright read a top-level `typings/` directory automatically, so no extra configuration is needed. (If you pass a custom `--outdir`, point your type checker's stub path at it — e.g. `python.analysis.stubPath` for Pylance.)

### Using the generated types

Unlike TypeScript — where the `createService` callback is typed automatically — a Python type checker **cannot** infer the type of a named handler's parameter from the `Service(...)` call. Import the type for your service version and annotate the handler's `ctx` parameter:

```python
from requence.service import Service
from requence.service.types import some_types  # one class per service version

def handler(ctx: some_types.context) -> some_types.output:
    name = ctx.input["name"]        # typed from the version's input schema
    return {"number": 10}           # checked against the version's output schema

Service("some-types", handler)      # the version string is validated too
```

The class name is your service version with every character that isn't a letter or digit replaced by `_`, and a leading `v` added if it starts with a digit — e.g. `some-types` → `some_types`, `1.2.3` → `v1_2_3`.

- **`ctx: <version>.context`** types `ctx.input`, `ctx.configuration`, and `ctx.to_output()`.
- **`-> <version>.output`** is required when you `return` an output dict literal directly (without it the checker infers a plain `dict` that won't match the output schema). You can omit it if you return through `ctx.to_output(...)`, whose return type already matches.
- For a trivial one-liner you can skip both the import and the annotation with a lambda, which the type checker types contextually:

  ```python
  Service("some-types", lambda ctx: {"number": 10})  # ctx is fully typed
  ```

### Options

| Option | Default | Description |
|---|---|---|
| `--access-token` | — | Service access token (falls back to env / `pyproject.toml`) |
| `--dev-token` | — | Personal access token for branch-specific types |
| `--outdir` | `typings` | Directory to write the type stubs to |
| `--watch` | `false` | Watch for schema changes and regenerate automatically |
| `--clear` / `--no-clear` | `true` | Clear the terminal on watch updates |

### Watch mode

```bash
requence-service generate-types --watch
```

The CLI connects via SSE and regenerates type stubs whenever a schema changes in the UI.

---

## Task

Start and monitor Requence tasks programmatically.

### Authentication

The token is resolved in this order:

1. `access_token` argument passed to `Task()`
2. `REQUENCE_TASK_ACCESS_TOKEN` or `REQUENCE_ACCESS_TOKEN` environment variable
3. `requence.task_access_token` or `requence.access_token` in `pyproject.toml`

```bash
REQUENCE_ACCESS_TOKEN=your-token python main.py
```

### Basic Usage

```python
from requence.task import Task

task = Task(task_template="my-template", input={"name": "World"})
result = task.sync.result  # Blocks until the task completes

print(result["result"])    # The final output of the task
```

### `Task` constructor options

| Parameter | Type | Default | Description |
|---|---|---|---|
| `task_template` | `str` | — | **Required.** Name of the task template |
| `input` | `dict` | `{}` | Input data (required when the template defines an input schema) |
| `name` | `str` | `None` | Human-readable task name shown in the UI |
| `priority` | `int` | `2` | Priority `0` (lowest) to `4` (highest) |
| `access_token` | `str` | `None` | Access token (falls back to env / `pyproject.toml`) |
| `require_ack` | `bool` | `False` | Enable delivery guarantee (see below) |
| `suppress_branch_warning` | `bool` | `False` | Suppress the branch warning on non-live branches |

### Synchronous access via `task.sync`

`task.sync` exposes blocking accessors — useful in standard synchronous code:

```python
task = Task(task_template="my-template", input={})

task_id = task.sync.id       # Blocks until the task is created
task_url = task.sync.url     # URL to view the task in the Requence UI
result = task.sync.result    # Blocks until the task completes
```

#### Awaiting finalization (cheap monitoring)

`task.sync.finalized()` blocks only until the task passes input validation and is created on the backend — without starting full SSE monitoring:

```python
from requence.exceptions import TaskException

try:
    task_id = task.sync.finalized()
    print(f"Task {task_id} is running in the background")
except TaskException as e:
    print("Validation failed:", e)
```

### Async access

`Task` also exposes async-compatible properties:

```python
task_id = await task.task_id
task_url = await task.task_url
result = await task.result
task_id = await task.finalized()
```

### Aborting and protecting

```python
task.abort("No longer needed")
task.protect()  # Exclude from automatic cleanup
```

### Standalone helpers

```python
from requence.task import abort_task, protect_task

abort_task(task_id="some-task-id", reason="Cancelled by user")
protect_task(task_id="some-task-id")
```

### Result object

When `task.sync.result` (or `await task.result`) resolves, it returns a `TaskResult` dict:

```python
result["task_id"]                      # Unique task identifier
result["task_url"]                     # URL to view the task in the Requence UI
result["input"]                        # The input you provided
result["result"]                       # The final task output
result["node_data"]["my_alias"]        # Output from a specific node by alias
result["node_error"]["my_alias"]       # Error from a specific node by alias
```

---

## Streaming Updates

### Synchronous iteration

```python
from requence.task import Task

task = Task(task_template="my-template", input={"data": [1, 2, 3]})

for update in task.sync.updates:
    match update["type"]:
        case "taskStart":
            print(f"Task {update['taskId']} started")
        case "nodeStart":
            node = update["node"]
            print(f"Node {node.get('alias', node['id'])} started")
        case "nodeUpdate":
            print("Node output:", update["data"])
        case "nodeError":
            print("Node error:", update["error"])
        case "nodeDefer":
            print("Node deferred:", update.get("reason"))
        case "taskEnd":
            print("Task completed:", update["context"]["result"])
        case "taskError":
            print("Task failed:", update.get("reason"))
        case "taskAborted":
            print("Task aborted:", update.get("reason"))
```

### Async iteration

```python
async for update in task.updates:
    print(update["type"], update["context"]["task_id"])
```

### Update types

| Type | Description |
|---|---|
| `taskStart` | Task execution has begun. Contains `input` and `taskId`. |
| `nodeStart` | A node started processing. Contains `node` info (id, type, alias). |
| `nodeUpdate` | A node produced output. Contains `data` and `output` (named output, if any). |
| `nodeError` | A node encountered an error. Contains `error` message. |
| `nodeDefer` | A node has been deferred (waiting for an external callback). |
| `nodeEnd` | A node finished processing. |
| `taskEnd` | The task completed successfully. Contains final `result`. |
| `taskError` | The task failed. Contains `reason`. |
| `taskAborted` | The task was aborted. Contains `reason`. |

### Context on every update

Every update includes a `context` dict with the current accumulated state:

```python
update["context"]["task_id"]                   # The task ID
update["context"]["input"]                     # The task input
update["context"]["result"]                    # Accumulated result (partial until taskEnd)
update["context"]["node_data"]["my_alias"]     # Output from a specific node
update["context"]["node_error"]["my_alias"]    # Error from a specific node
```

---

## Fetching a Task by ID

Retrieve the current state of any task:

```python
from requence.task import get_task

status, context = get_task("some-task-id")
# or: get_task("some-task-id", access_token="...")

# status: 'successful' | 'failed' | 'idle' | 'pending' | 'running' | 'stopped'
print(status)
print(context["input"])
print(context["result"])
print(context["node_data"].get("my_alias"))
print(context["node_error"].get("my_alias"))
```

---

## Watching All Tasks (`TaskWatcher`)

Subscribe to real-time updates across **all tasks** — useful for dashboards, monitoring systems, or audit logs.

```python
from requence.task import TaskWatcher
from datetime import datetime

watcher = TaskWatcher(since=datetime.now())

# Synchronous iteration
for update, incomplete in watcher.sync().updates:
    print(f"[{update['type']}] Task {update['context']['task_id']}")
    print("Incomplete (joined mid-task):", incomplete)
```

```python
# Async iteration
async for update, incomplete in watcher.updates:
    print(update["type"])
```

Call `watcher.stop()` to disconnect:

```python
watcher.stop()
```

### `TaskWatcher` options

| Parameter | Type | Description |
|---|---|---|
| `since` | `date` | **Required.** Only receive updates after this timestamp. |
| `access_token` | `str` | Access token (falls back to env / `pyproject.toml`) |
| `on_connect` | `callable` | Called when the connection is established |

### Incomplete flag

Each update tuple from `TaskWatcher` includes an `incomplete` boolean. When `True`, it means the watcher connected after the task had already started — the `taskStart` event was missed. Use this to decide whether to skip or partially process the update.

### Delivery Guarantee

When a task is started with `require_ack=True`, `TaskWatcher` automatically ACKs the terminal event (`taskEnd`, `taskError`, `taskAborted`) as soon as it is received — no extra code required.

---

## Delivery Guarantee

By default, a task transitions immediately to its final status when it finishes. If the process that started the task crashes at the exact moment the terminal event is emitted, the event can be lost.

Setting `require_ack=True` enables an explicit acknowledgement step:

1. When the task finishes the backend holds it in **`AWAITING_DELIVERY`** instead of immediately finalising.
2. The SDK receives the terminal event and automatically sends an ACK back.
3. Only after the ACK is received does the task move to its real final status.

```python
task = Task(
    task_template="my-template",
    input={},
    require_ack=True,
)

result = task.sync.result  # Only resolves after the ACK has been confirmed
```

---

## CLI — `requence-task generate-types`

The package installs a `requence-task` CLI that generates Python type stubs from the task template schemas defined in the Requence UI:

```bash
requence-task generate-types
```

Type stubs are written to `typings/requence/task/`, giving you type hints for task inputs and results. As with the service stubs, Pylance and pyright read a top-level `typings/` directory automatically (point your stub path at a custom `--outdir` otherwise).

### Options

| Option | Default | Description |
|---|---|---|
| `--access-token` | — | Access token (falls back to env / `pyproject.toml`) |
| `--outdir` | `typings` | Directory to write the type stubs to |

---

## Full Example — Service

```python
from requence.service import Service
from my_db import db

def handler(ctx):
    if not db.is_connected:
        ctx.retry(2000)  # wait 2s for the DB to recover

    if not ctx.input.get("ocr_data"):
        ctx.abort("OCR data is mandatory")

    result = db.get_data_based_on_ocr(ctx.input["ocr_data"])
    return result

Service({"version": "1.2.3", "prefetch": 2}, handler)
```

## Full Example — Task

```python
from requence.task import Task
from requence.exceptions import TaskException

task = Task(
    task_template="invoice-processing",
    input={"invoice_url": "https://..."},
    name="Invoice #1234",
    priority=3,
)

task_id = task.sync.id
print("Started task:", task_id)

try:
    result = task.sync.result
    print("Final result:", result["result"])
except TaskException as e:
    print("Task failed:", e)
```
