+++
title = "OpenTelemetry Basics Tour"
date = 2025-04-01

[taxonomies]
tags = ["observability"]
+++

# OpenTelemetry Basics Tour

## Intro

OpenTelemetry is an observability framework and toolkit to **create** and **manage** telemetry data like **traces**, **metrics**, and **logs**.

### Concepts

#### Traces

Composed of spans in a tree structure. A trace represents the end-to-end journey of a request, while a span represents a single operation within that journey.

`TracerProvider` is a factory for `Tracer`s. A `Tracer` creates spans with information about what is happening for a given operation.
`Trace Exporter` sends traces to a consumer, which can be standard output, the OpenTelemetry Collector, or any open source or vendor backend.

##### Spans

Fields contained:
- Name
- Parent span ID (empty for root spans)
- Start and End Timestamps
- Span Context
- Attributes — info in KV pairs
- Span Events — things happening at a point in time, whose timestamp is meaningful
- Span Links — indicate causal relationships between spans in distributed situations
- Span Status — indicates whether the tracked operation completed with or without error (`Unset`, `Error`), or was explicitly set to `Ok`

Kind:
- Client — remote call such as an outgoing HTTP request or DB call
- Server — incoming remote call such as an incoming HTTP request or RPC
- Internal — operations that do not cross a process boundary, such as instrumenting a function call
- Producer — represents the creation of a job
- Consumer — represents the processing of a job, after the Producer span has already ended

#### Metrics

A **metric** is a measurement of a service captured at runtime. The moment of capturing a measurement is known as a **metric event**.

Application and request metrics are important indicators of availability and performance.

A `MeterProvider` is a factory for `Meter`s. A `Meter` creates metric instruments, capturing measurements about a service at runtime. `Metric Exporter`s send metric data to a consumer (similarly to `Trace Exporter`).

##### Metric Instruments

Fields:
- Name
- Kind
- Unit (optional)
- Description (optional)

Name, unit, and description are chosen by the developer or defined via **semantic conventions** for common ones like request and process metrics.

Kind enums:
- Counter — a value accumulating over time
- Asynchronous Counter — same, but is collected **once** for the export
- UpDownCounter — counter that can go down
- Asynchronous UpDownCounter
- Gauge
- Histogram

##### Aggregation

Exact or estimated statistics about metric events during a time window.

e.g.
- Total bytes read by a service, per protocol type
- Total bytes read and bytes per request
- Duration of a system call
- Request sizes for trend prediction
- CPU or memory usage of a process

#### Logs

A log is a timestamped text record.

Log Appender/Bridge emits logs into an OpenTelemetry `LogRecordExporter`.

`LoggerProvider` is a factory for `Logger`s. A `Logger` creates log records. `LogRecordExporter`s send log records to a consumer.

Fields:
- Timestamp — Time when the event occurred
- ObservedTimestamp — Time when the event was observed
- TraceId — Request trace ID
- SpanId — Request span ID
- TraceFlags — W3C trace flag
- SeverityText — The severity text (also known as log level)
- SeverityNumber — Numerical value of the severity
- Body — The body of the log record
- Resource — Describes the source of the log
- InstrumentationScope — Describes the scope that emitted the log
- Attributes — Additional information about the event

### Major Components

- A specification for all components
- A standard protocol that defines the shape of telemetry data
- Semantic conventions that define a standard naming scheme for common telemetry data types
- APIs that define how to generate telemetry data
- Language SDKs that implement the specification, APIs, and export of telemetry data
- A library ecosystem that implements instrumentation for common libraries and frameworks
- Automatic instrumentation components that generate telemetry data without requiring code changes
- The OpenTelemetry Collector, a proxy that receives, processes, and exports telemetry data
- Other tools
