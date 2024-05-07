---
layout: page
title: AI Application Observability
permalink: /ai-application-observability/
description: |
  Explore essential observability tools for AI applications,
  including OpenTelemetry for telemetry data, Pyroscope for continuous profiling,
  and LangSmith for LLM observability.
---

## Introduction

With the rise of AI applications, it's important to have observability in place.
Especially when control flow is not as clear as traditional applications.
This post will cover basic logs, metrics, traces using OpenTelemetry.
Continuous profiling using Pyroscope.
LLM observability using LangSmith.

## OpenTelemetry

[OpenTelemetry](https://opentelemetry.io/) is a collection of APIs, SDKs, and tools.
Use it to instrument, generate, collect, and export telemetry data
(metrics, logs, and traces) to help you analyze your software’s performance and behavior.

The usual flow is to add OpenTelemetry SDK to application, then it can export
telemetry data to various backends, like Prometheus, Tempo, or an OTEL collector.

Some SDK supports automatic instrumentation, for example in Python, you can add

```shell
pip install opentelemetry-distro[otlp] opentelemetry-instrumentation
```

Then run

```shell
opentelemetry-bootstrap -a requirements
```

It will list all the packages that need to be installed to instrument your application.
Note that some instrumentation may crash the app due to bugs like [this](https://github.com/open-telemetry/opentelemetry-python-contrib/issues/2340),
cherry-pick the ones you need.

Then all options can be configured via environment variables,
[official doc](https://opentelemetry.io/docs/languages/python/automatic/configuration/#environment-variables).

Take the previous [chat search example](https://github.com/hemslo/chat-search),
entrypoint command changed to

```shell
opentelemetry-instrument uvicorn app.server:app --host 0.0.0.0 --port 8000
```

start local development with

```shell
docker compose up --build
```

Then ask a question in [chat playground](http://localhost:8000/chat/playground/)
and explore tempo traces in [grafana](http://localhost:3000/explore).

![image](/assets/images/grafana-tempo.png)

We can inspect redis query and request to LLM model.

![image](/assets/images/trace-redis.png)

![image](/assets/images/trace-llm-request.png)

If export to grafana cloud, there is a default application dashboard to view them all.

![image](/assets/images/grafana-cloud-application.png)

## Pyroscope

Sometimes the performance issue is not in the network or external services,
but in the code itself. During development we can use local profiling tools to diagnose them,
while in production, it's not easy to attach the profiler to the running process,
thus we need continuous profiling.

[Pyroscope](https://pyroscope.io/) is a continuous profiling tool.
It can be used to locate performance issues down to the line of code.
For python, some config in code is needed.

```shell
pip install pyroscope-io
```

```python
import pyroscope

pyroscope.configure(
    application_name="app_name",
    server_address="http://pyroscope:4040",
)
```

Then the flame graph can be viewed in [pyroscope UI](http://localhost:4040/).

![image](/assets/images/pyroscope-ui.png)

## LangSmith

An unique part of AI application observability is LLM observability.
For example in LangChain chain, the response of model can be used to do many things,
can be directly shown to user, or be the input next step,
or even be used to decide the next step in the control flow.
Thus monitoring the full lifecycle of a chain is very important.

[LangSmith](https://www.langchain.com/langsmith) is a platform for LLM observability.
It has built-in support for LangChain, can be used to monitor all interactions with models.

To enable LangSmith, just set environment variables

```shell
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=<your-api-key>
```

Then check the LangSmith UI for traces.
Here is a public demo for my blog [chatbot](https://chat-search.hemslo.io/chat/playground/).

![image](/assets/images/chat-search-demo.png)

Then click `View LangSmith trace` to see the trace.

![image](/assets/images/langsmith-ui.png)

## Conclusion

AI application observability is not easy,
but with several tools like OpenTelemetry, Pyroscope, LangSmith combined,
we can have multi-dimension views of the application.
