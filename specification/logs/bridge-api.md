# Logs Bridge API

**Status**: [Experimental](../document-status.md)

<details>
<summary>Table of Contents</summary>

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [LoggerProvider](#loggerprovider)
  * [LoggerProvider operations](#loggerprovider-operations)
    + [Get a Logger](#get-a-logger)
- [Logger](#logger)
  * [Logger operations](#logger-operations)
    + [Emit LogRecord](#emit-logrecord)
- [LogRecord](#logrecord)
- [Optional and required parameters](#optional-and-required-parameters)
- [Concurrency requirements](#concurrency-requirements)
- [Usage](#usage)
  * [How to Create Log4J Style Appender](#how-to-create-log4j-style-appender)
  * [Implicit Context Injection](#implicit-context-injection)
  * [Explicit Context Injection](#explicit-context-injection)

<!-- tocstop -->

</details>

<b>Note: this document defines a log *backend* API. The API is not intended to be called
by application developers directly. It is provided for logging library authors
to build [Appenders](#how-to-create-log4j-style-appender), which use
this API to bridge between existing logging libraries and the OpenTelemetry log
data model.</b>

The Logs Bridge API consist of these main classes:

* [LoggerProvider](#loggerprovider) is the entry point of the API. It provides access to `Logger`s.
* [Logger](#logger) is the class responsible for emitting logs as [LogRecords](#logrecord).

```mermaid
graph TD
    A[LoggerProvider] -->|Get| B(Logger)
    B -->|Emit| C(LogRecord)
```

## LoggerProvider

`Logger`s can be accessed with a `LoggerProvider`.

In implementations of the API, the `LoggerProvider` is expected to be the stateful
object that holds any configuration.

Normally, the `LoggerProvider` is expected to be accessed from a central place.
Thus, the API SHOULD provide a way to set/register and access a global default
`LoggerProvider`.

### LoggerProvider operations

The `LoggerProvider` MUST provide the following functions:

* Get a `Logger`

#### Get a Logger

This API MUST accept the following parameters:

* `name`: This name uniquely identifies the [instrumentation scope](../glossary.md#instrumentation-scope),
  such as the [instrumentation library](../glossary.md#instrumentation-library)
  (e.g. `io.opentelemetry.contrib.mongodb`), package, module or class name.
  If an application or library has built-in OpenTelemetry instrumentation, both
  [Instrumented library](../glossary.md#instrumented-library) and
  [Instrumentation library](../glossary.md#instrumentation-library) may refer to
  the same library. In that scenario, the `name` denotes a module name or component
  name within that library or application.

* `version` (optional): Specifies the version of the instrumentation scope if
  the scope has a version (e.g. a library version). Example value: 1.0.0.

* `schema_url` (optional): Specifies the Schema URL that should be recorded in
  the emitted telemetry.

* `attributes` (optional): Specifies the instrumentation scope attributes to
  associate with emitted telemetry. This API MUST be structured to accept a
  variable number of attributes, including none.

* `include_trace_context` (optional): Specifies whether the Trace Context should
  automatically be passed on to the `LogRecord`s emitted by the `Logger`.
  If `include_trace_context` is not specified, it SHOULD be `true` by default.

`Logger`s are identified by `name`, `version`, and `schema_url` fields.  When more
than one `Logger` of the same `name`, `version`, and `schema_url` is created, it
is unspecified whether or under which conditions the same or different `Logger`
instances are returned. It is a user error to create Loggers with different
`include_trace_context` or `attributes` but the same identity.

The term *identical* applied to `Logger`s describes instances where all
identifying fields are equal. The term *distinct* applied to `Logger`s describes
instances where at least one identifying field has a different value.

The effect of associating a Schema URL with a `Logger` MUST be that the telemetry
emitted using the `Logger` will be associated with the Schema URL, provided that
the emitted data format is capable of representing such association.

## Logger

The `Logger` is responsible for emitting `LogRecord`s.

### Logger operations

The `Logger` MUST provide functions to:

#### Emit LogRecord

Emit a `LogRecord` to the processing pipeline.

This function MAY be named `logRecord`.

**Parameters:**

* `logRecord` - the [LogRecord](#logrecord) to emit.

## LogRecord

The API emits [LogRecords](#emit-logrecord) using the `LogRecord` [data model](data-model.md).

A function receiving this as an argument MUST be able to set the following
parameters:

- [Timestamp](./data-model.md#field-timestamp)
- [Observed Timestamp](./data-model.md#field-observedtimestamp)
- [Context](../context/README.md) that contains the
  [TraceContext](./data-model.md#trace-context-fields)
- [Severity Number](./data-model.md#field-severitynumber)
- [Severity Text](./data-model.md#field-severitytext)
- [Body](./data-model.md#field-body)
- [Attributes](./data-model.md#field-attributes)

All parameters are optional.

## Optional and required parameters

The operations defined include various parameters, some of which are marked
optional. Parameters not marked optional are required.

For each optional parameter, the API MUST be structured to accept it, but MUST
NOT obligate a user to provide it.

For each required parameter, the API MUST be structured to obligate a user to
provide it.

## Concurrency requirements

For languages which support concurrent execution the Logs Bridge APIs provide
specific guarantees and safeties.

**LoggerProvider** - all methods are safe to be called concurrently.

**Logger** - all methods are safe to be called concurrently.

## Usage

### How to Create Log4J Style Appender

An Appender implementation can be used to bridge logs into the [Log SDK](./sdk.md)
OpenTelemetry [LogRecordExporters](sdk.md#logrecordexporter). This approach is
typically used for applications which are fine with changing the log transport
and is [one of the supported](README.md#direct-to-collector) log collection
approaches.

The Appender implementation will typically acquire a [Logger](#logger) from the
global [LoggerProvider](#loggerprovider) at startup time, then
call [Emit LogRecord](#emit-logrecord) for `LogRecord`s received from the
application.

[Implicit Context Injection](#implicit-context-injection)
and [Explicit Context Injection](#explicit-context-injection) describe how an
Appender injects `TraceContext` into `LogRecord`s.

![Appender](img/appender.png)

This same approach can be also used for example for:

- Python logging library by creating a Handler.
- Go zap logging library by implementing the Core interface. Note that since
  there is no implicit Context in Go it is not possible to get and use the
  active Span.

Appenders can be created in OpenTelemetry language libraries by OpenTelemetry
maintainers, or by 3rd parties for any logging library that supports a similar
extension mechanism. This specification recommends each OpenTelemetry language
library to include out-of-the-box Appender implementation for at least one
popular logging library.

### Implicit Context Injection

When Context is implicitly available (e.g. in Java) the Appender can rely on
automatic context propagation by [obtaining a Logger](#get-a-logger)
with `include_trace_context=true`.

Some log libraries have mechanisms specifically tailored for injecting contextual
information into logs, such as MDC in Log4j. When available such mechanisms may
be the preferable place to fetch the `TraceContext` and inject it into
the `LogRecord`, since it usually allows fetching of the context to work
correctly even when log records are emitted asynchronously, which otherwise can
result in the incorrect implicit context being fetched.

TODO: clarify how works or doesn't work when the log statement call site and the
log appender are executed on different threads.

### Explicit Context Injection

In languages where the Context must be provided explicitly (e.g. Go) the end
user must capture the context and explicitly pass it to the logging subsystem in
order for `TraceContext` to be recorded in `LogRecord`s.

Support for OpenTelemetry for logging libraries in these languages typically can
be implemented in the form of logger wrappers that can capture the context once,
when the span is created and then use the wrapped logger to execute log
statements in a normal way. The wrapper will be responsible for injecting the
captured context in the `LogRecord`s.

This specification does not define how exactly it is achieved since the actual
mechanism depends on the language and the particular logging library used. In
any case the wrappers are expected to make use of the Trace Context API to get
the current active span.

See
[an example](https://docs.google.com/document/d/15vR7D1x2tKd7u3zaTF0yH1WaHkUr2T4hhr7OyiZgmBg/edit#heading=h.4xuru5ljcups)
of how it can be done for zap logging library for Go.
