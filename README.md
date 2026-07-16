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

### Closing the service

`Service.__init__` blocks the current thread (it calls `channel.start_consuming()`). To stop consuming from another thread, call `close()`:

```python
import threading
from requence.service import Service

service = None

def handler(ctx):
    return ctx.input

def run():
    global service
    service = Service("1.0.0", handler)

thread = threading.Thread(target=run)
thread.start()

# Later, from another thread:
service.close()
```

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

After deferring a message with `ctx.defer()`, deliver the result later — even from a different process — using the `act()` method on the `Service` instance:

```python
from requence.service import Service

service = None

def handler(ctx):
    message_key = ctx.defer()
    save_to_db(ctx.task_id, message_key)

service = Service("1.0.0", handler)  # blocks
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

The `actor` callable receives an `api` dict with three keys:

| Key | Description |
|---|---|
| `api["send"](data)` | Send data to the default output |
| `api["send_to_output"](name, data)` | Send data to a named output |
| `api["abort"](error)` | Abort the deferred message with an error string or exception |

The actor can also return a value or a generator directly, which behaves like calling `api["send"]()` for each value.

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
