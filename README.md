# requence

The official Python SDK for the Requence platform. It covers both halves of the integration:

- **`requence.service`** — connect a Python service to Requence and process messages
- **`requence.task`** — start and monitor Requence tasks programmatically

> **This README is a getting-started guide.** The reference for both halves lives at
> **<https://docs.requence.cloud>** — every page has a **Python** tab, including the places
> where Python's answer differs from TypeScript's.

## Requirements

Python 3.12 or later.

## Installation

```bash
pip install requence
```

## Authentication

Copy a service's access token from the **Services** list view in the Requence UI
(**Copy credentials**). A task needs a personal or scoped access token instead.

Tokens are resolved in this order:

1. `access_token` passed to `Service()` / `Task()`
2. `REQUENCE_SERVICE_ACCESS_TOKEN` / `REQUENCE_TASK_ACCESS_TOKEN`, then `REQUENCE_ACCESS_TOKEN`
3. `requence.service_access_token` / `requence.task_access_token`, then `requence.access_token`
   in `pyproject.toml`

```bash
REQUENCE_SERVICE_ACCESS_TOKEN=your-token python main.py
```

## Your first service

Create the service in the Requence UI first — its input schema, configuration schema and
outputs. Then implement the version:

```python
from requence.service import Service

def handler(ctx):
    return {"message": f"Hello, {ctx.input['name']}!"}

Service("1.0.0", handler)
```

> **`Service(...)` never returns.** It consumes on the calling thread, so there is no instance
> to call `act()` or `close()` on. Use `Service.start(...)` when you need either — it returns
> the instance once connected, on a daemon consume thread. See
> [Starting & Stopping][lifecycle].

Generate type stubs for `ctx.input`, `ctx.configuration` and `ctx.to_output()` from the schemas
you defined in the UI:

```bash
requence-service generate-types
```

## Your first task

```python
from requence.task import Task

task = Task(task_template="my-template", input={"name": "World"})
result = task.sync.result   # blocks until the task completes

print(result["result"])
```

`task.sync` holds the blocking accessors; `await task.result` and `async for update in
task.updates` are the async equivalents.

## Where to read next

### Writing a service

| Topic                                                                   | Page                                              |
| ----------------------------------------------------------------------- | ------------------------------------------------- |
| The full walkthrough                                                    | [Getting Started][started]                        |
| `ctx.input`, `ctx.debug`, `retry`, `abort`, `skip`, `to_output`, `defer` | [Context API][context]                            |
| `prefetch`, `ssl_context`, TLS via environment                          | [Service Options][options]                        |
| `Service(...)` vs `Service.start(...)`, `close()`                       | [Starting & Stopping][lifecycle]                  |
| Generator handlers and `ctx.terminated`                                 | [Continuous Mode][continuous]                     |
| `ctx.store` — what a node remembers past this message                   | [The Store][store] · [`ctx.store`][context-store] |
| `create_durable_callback`, `.for_(…)`                                   | [Callbacks][callbacks]                            |
| `create_ui`, `ctx.render`, handler props                                | [Surfaces][surfaces]                              |
| `ctx.defer()` → `service.act()`                                         | [Deferred Delivery][deferred]                     |
| Running locally against real tasks                                      | [Dev Overlay][overlay]                            |
| `requence-service generate-types`                                       | [Typing / CLI][typing]                            |

### Starting tasks

| Topic                                                     | Page                            |
| --------------------------------------------------------- | ------------------------------- |
| Options, `task.sync`, abort, protect, `require_ack`       | [Starting Tasks][starting]      |
| Sync and async iteration, the update types                | [Streaming Updates][streaming]  |
| `TaskWatcher` across every task                           | [Watching Tasks][watching]      |
| `get_task`                                                | [Fetching & Recreating][fetch]  |
| `requence-task generate-types`                            | [CLI][cli]                      |

`test/demo-service/surface_demo.py` and `test/demo-python/` in the Requence repository are
runnable examples.

[started]: https://docs.requence.cloud/implementation/getting-started/
[context]: https://docs.requence.cloud/implementation/context-api/
[context-store]: https://docs.requence.cloud/implementation/context-api/#ctxstoreprivatekey-binding--ctxstoresharedkey-binding
[options]: https://docs.requence.cloud/implementation/advanced/service-options/
[lifecycle]: https://docs.requence.cloud/implementation/advanced/
[continuous]: https://docs.requence.cloud/planning/continuous-mode/
[store]: https://docs.requence.cloud/planning/store/
[callbacks]: https://docs.requence.cloud/implementation/callbacks/
[surfaces]: https://docs.requence.cloud/implementation/surfaces/
[deferred]: https://docs.requence.cloud/implementation/advanced/deferred-delivery/
[overlay]: https://docs.requence.cloud/implementation/advanced/dev-overlay/
[typing]: https://docs.requence.cloud/implementation/typing-cli/
[starting]: https://docs.requence.cloud/execution/starting-tasks/
[streaming]: https://docs.requence.cloud/execution/streaming/
[watching]: https://docs.requence.cloud/execution/watching-tasks/
[fetch]: https://docs.requence.cloud/execution/managing-tasks/
[cli]: https://docs.requence.cloud/execution/cli/
