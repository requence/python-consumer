# requence

This package connects python to the operator bus. It manages the retrieval and responses of messages.

## Usage

```python
from requence.consumer import ContextHelper, Consumer

def handleMessage(context: ContextHelper):
    return "this is my response"

Consumer({}, handleMessage)
```

Every consumer instance needs a `url` parameter to connect to the operator and a `version`. In the basic example, those parameters get automatically retrieved from environment variables `REQUENCE_URL` and `VERSION`.
To be more explicit about those configurations, you can pass an object as the first parameter:

```python
Consumer(
  {
    'url': 'your operator connection url string',
    'version': 'your version in format major.minor.patch',
  },
  handleMessage
)
```

## Additional configuration

By default, the consumer retrieves one message from the operator, processes it and passes the response back to the operator. If your service is capable of processing multiple messages in parallel, you can define a higher `prefetch`.

```python
Consumer(
  {
    'prefetch': 10, # this will process max. 10 messages at once when available
  },
  handleMessage
)
```

## Unsubscribing service

Should you ever need to unsubscribe your service from the operator programmatically, `Consumer` has a close method.

```python
consumer = Consumer(...)

# later
consumer.close()
```

To resubscribe, you have to instantiate a new `Consumer`

## Processing messages

The message handler callback provides one argument: the message context.
The context provides helper methods to access previous operator results and also methods to abort the processing early.

### context api data retrieval

```python
ctx.get_input(): Optional[Any]
```

The input that was defined when the task started

```python
ctx.get_meta(): Optional[Any]
```

The additional meta information added to the task

```python
ctx.get_configuration(): Optional[Any]
```

The optional configuration of the current service

```python
ctx.get_service_meta(serviceIdentifier: str): TypedDict{
    'executionDate': Optional[datetime], # null when the service was not yet executed
  'id': str, // service id used for internal routing
  'alias': Optional[str], # service alias (see service Identifier)
  'name': str,
  'configuration': Optional[Any],
  'version': str
}
```

The meta parameters of a service, usually not needed

```python
ctx.get_service_data(serviceIdentifier: str): Any
```

The response of a previously executed service

```python
ctx.get_last_service_data(serviceIdentifier: str): Any
```

Same as `ctx.get_service_data` but only returns the last data when a service is used multiple times

```python
ctx.get_service_error(serviceIdentifier: str): str | None
```

The error message of a previously executed service or null when said service was not yet executed or did process without error.

```python
ctx.get_last_service_error(serviceIdentifier: str): str | None
```

Same as `ctx.get_service_error` but only returns the last error when a service is used multiple times

```python
ctx.get_results(): Array[TypedDict{
  'executionDate': Optional[datetime], # null when the service was not yet executed
  'id': str, # service id used for internal routing
  'alias': Optional[str], # service alias (see service Identifier)
  'name': str,
  'configuration': Optional[Any],
  'version': str
  'error': Optional[str]
  'data': Optional[Any]
}]
```

get results of all configured services in this task. When a service did run prior to the current service, `executionDate` and `error` or `data` will be available.


```python
ctx.get_tenant_name(): str
```

The name of the tenant that initiated the task


### context api processing control

```python
ctx.retry(delay: Optional[int]): None
```

Instructs the operator to retry the service after an optional delay in milliseconds (minimum is 100ms). No code gets executed after this line.
**Currently, it is your responsibility to prevent infinite loops.**

```python
ctx.abort(reason: Optional[str]): None
```

Instructs the operator to abort the processing of this service immediately.
If the service is not configured with a fail over, the complete task will fail.

### service identifier

Most context methods require a service identifier as parameter. This identifier can either be a service name or a service alias. The latter is useful for situations where a service is used multiple times in a task and needs the data from one specific execution.

## Full example

Pseudo implementation of a database service

```python
from requence.consumer import ContextHelper, Consumer
from someDb import db

def handleMessage(context: ContextHelper):
    ocrData = ctx.get_service_data("ocr")

    if (not ocrData):
      ctx.abort("Ocr service is mandatory for this AI service")

    if (not db.isConnected):
        # lets wait 2s for the db to recover
        ctx.retry(2000)

    response = db.getDataBasedOnOcr(ocrData)

    return response
```
