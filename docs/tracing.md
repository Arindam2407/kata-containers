# Overview

This document explains how to trace Kata Containers components.

# Introduction

The Kata Containers runtime and agent are able to generate
[OpenTelemetry][opentelemetry] trace spans, which allow the administrator to
observe what those components are doing and how much time they are spending on
each operation.

# OpenTelemetry summary

An OpenTelemetry-enabled application creates a number of trace "spans". A span
contains the following attributes:

- A name
- A pair of timestamps (recording the start time and end time of some operation)
- A reference to the span's parent span

All spans need to be *finished*, or *completed*, to allow the OpenTelemetry
framework to generate the final trace information (by effectively closing the
transaction encompassing the initial (root) span and all its children).

For Kata, the root span represents the total amount of time taken to run a
particular component from startup to its shutdown (the "run time").

# Architecture

## Runtime tracing architecture

The runtime, which runs in the host environment, has been modified to
optionally generate trace spans which are sent to a trace collector on the
host.

## Agent tracing architecture

An OpenTelemetry system (such as [Jaeger][jaeger-tracing]) uses a collector to
gather up trace spans from the application for viewing and processing. For an
application to use the collector, it must run in the same context as
the collector.

This poses a problem for tracing the Kata Containers agent since it does not
run in the same context as the collector: it runs inside a virtual machine (VM).

To allow spans from the agent to be sent to the trace collector, Kata provides
a [trace forwarder][trace-forwarder] component. This runs in the same context
as the collector (generally on the host system) and listens on a
[`VSOCK`][vsock] channel for traces generated by the agent, forwarding them on
to the trace collector.

> **Note:**
>
> This design supports agent tracing without having to make changes to the
> image, but also means that [custom images][osbuilder] can also benefit from
> agent tracing.

The following diagram summarises the architecture used to trace the Kata
Containers agent:

```
+--------------------------------------------+
| Host                                       |
|                                            |
| +---------------+                          |
| | OpenTelemetry |                          |
| | Trace         |                          |
| | Collector     |                          |
| +---------------+                          |
|       ^                  +---------------+ |
|       | spans            | Kata VM       | |
| +-----+-----+            |               | |
| | Kata      |    spans   o     +-------+ | |
| | Trace     |<-----------------| Kata  | | |
| | Forwarder |    VSOCK   o     | Agent | | |
| +-----------+    Channel |     +-------+ | |
|                          +---------------+ |
+--------------------------------------------+
```

# Agent tracing prerequisites

- You must have a trace collector running.

  Although the collector normally runs on the host, it can also be run from
  inside a Docker image configured to expose the appropriate host ports to the
  collector.

  The [Jaeger "all-in-one" Docker image][jaeger-all-in-one] method
  is the quickest and simplest way to run the collector for testing.

- If you wish to trace the agent, you must start the
  [trace forwarder][trace-forwarder].

> **Notes:**
>
> - If agent tracing is enabled but the forwarder is not running,
>   the agent will log an error (signalling that it cannot generate trace
>   spans), but continue to work as normal.
>
> - The trace forwarder requires a trace collector (such as Jaeger) to be
>   running before it is started. If a collector is not running, the trace
>   forwarder will exit with an error.

# Enable tracing

By default, tracing is disabled for all components. To enable _any_ form of
tracing an `enable_tracing` option must be enabled for at least one component.

> **Note:** 
>
> Enabling this option will only allow tracing for subsequently
> started containers.

## Enable runtime tracing

To enable runtime tracing, set the tracing option as shown:

```toml
[runtime]
enable_tracing = true
```

## Enable agent tracing

To enable agent tracing, set the tracing option as shown:

```toml
[agent.kata]
enable_tracing = true
```

> **Note:**
>
> If both agent tracing and runtime tracing are enabled, the resulting trace
> spans will be "collated": expanding individual runtime spans in the Jaeger
> web UI will show the agent trace spans resulting from the runtime
> operation.

# Appendices

## Agent tracing requirements

### Host environment

- The host kernel must support the VSOCK socket type.

  This will be available if the kernel is built with the
  `CONFIG_VHOST_VSOCK` configuration option.

- The VSOCK kernel module must be loaded:

   ```
   $ sudo modprobe vhost_vsock
   ```

### Guest environment

- The guest kernel must support the VSOCK socket type:

  This will be available if the kernel is built with the
  `CONFIG_VIRTIO_VSOCKETS` configuration option.

  > **Note:** The default Kata Containers guest kernel provides this feature.

## Agent tracing limitations

- Agent tracing is only "completed" when the workload and the Kata agent
  process have exited.

  Although trace information *can* be inspected before the workload and agent
  have exited, it is incomplete. This is shown as `<trace-without-root-span>`
  in the Jaeger web UI.

  If the workload is still running, the trace transaction -- which spans the entire
  runtime of the Kata agent -- will not have been completed. To view the complete
  trace details, wait for the workload to end, or stop the container.

## Performance impact

[OpenTelemetry][opentelemetry] is designed for high performance. It combines
the best of two previous generation projects (OpenTracing and OpenCensus) and
uses a very efficient mechanism to capture trace spans. Further, the trace
points inserted into the agent are generated dynamically at compile time. This
is advantageous since new versions of the agent will automatically benefit
from improvements in the tracing infrastructure. Overall, the impact of
enabling runtime and agent tracing should be extremely low.

## Agent shutdown behaviour
 
In normal operation, the Kata runtime manages the VM shutdown and performs
certain optimisations to speed up this process. However, if agent tracing is
enabled, the agent itself is responsible for shutting down the VM. This it to
ensure all agent trace transactions are completed. This means there will be a
small performance impact for container shutdown when agent tracing is enabled
as the runtime must wait for the VM to shutdown fully.

## Set up a tracing development environment

If you want to debug, further develop, or test tracing,
[enabling full debug][enable-full-debug]
is highly recommended. For working with the agent, you may also wish to
[enable a debug console][setup-debug-console]
to allow you to access the VM environment.

[agent-ctl]: https://github.com/kata-containers/kata-containers/blob/main/tools/agent-ctl
[enable-full-debug]: https://github.com/kata-containers/kata-containers/blob/main/docs/Developer-Guide.md#enable-full-debug
[jaeger-all-in-one]: https://www.jaegertracing.io/docs/getting-started/
[jaeger-tracing]: https://www.jaegertracing.io
[opentelemetry]: https://opentelemetry.io
[osbuilder]: https://github.com/kata-containers/kata-containers/blob/main/tools/osbuilder
[setup-debug-console]: https://github.com/kata-containers/kata-containers/blob/main/docs/Developer-Guide.md#set-up-a-debug-console
[trace-forwarder]: https://github.com/kata-containers/kata-containers/blob/main/src/trace-forwarder
[vsock]: https://wiki.qemu.org/Features/VirtioVsock