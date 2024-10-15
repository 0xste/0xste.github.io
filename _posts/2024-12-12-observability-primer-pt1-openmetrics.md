---
title: OpenMetrics Primer
date: 2024-12-12 01:00:00 +0000
categories: [Observability]
tags: [software,engineering,openmetrics,opensource]
image:
  path: /assets/img/openmetrics/openmetrics.jpg
  alt: OpenMetrics
---

One thing i've been heavily exposed to across a lot of organizations is general observability strategy 

I've worked with various SaaS providers including DataDog, AppDynamics, and the various native cloud offerings and hybrid packaged models like Elastic, as well as native cloud provider logging solutions like AWS CloudWatch and GCP StackDriver.

I'm a big advocate of Open Source Software and have generally been a proponent of the OpenMetrics standard, Sketch Algorithms and general approach to lower cost monitoring using of time-series data structures.

Ultimately this has led to a few different open source contributions across projects largely in the web3 space, namely the Ethereum Prysm Client to participate in the execution layer of the network, and the MEV-Boost project a tool used by node operators to coordinate block preparation to extract rewards via relays. 


We'll talk a little about those contributions an how you might end up getting involved in OSS contributions like this in future, as well as a suggestion for some low hanging fruit in those repositories, but first in this post we'll start delving into the underlying wire format for OpenMetrics, and the implications of using this as a primary datasource for 

## Caveats
The aim of this is to not go into the underlying implementation, but more so into the standard and how it is used practically from an engineering perspective.


## What is the OpenMetrics Standard?

A metric represents a single point from a time series, put simply; a single point from a line on a line graph. The [OpenMetrics standard](https://github.com/OpenObservability/OpenMetrics/blob/main/specification/OpenMetrics.md) looks to standardize this by going a step further and suggesting the wire format, the collection standards and the exposure of these metrics. Following this standard will plot you a line on a graph. 

The format is flexible enough to support various types of data types, which at a super high level are as follows:


### Counter

Represents an increasing counter, the value can only ever go up, use this to represent things like # of times an event has occurred, this is probably the most used metric type and the really standard example for this is an API request counter.

```
api_request_count{path="/health"} 10
```

Breaking this down further, the above metric tells us that we have made `10` requests to the `/health` endpoint of our service.

### Gauge

Can go up and down, used commonly to represent percentages or a live value, for example % disk usage, usually suffixed with _meter

```
disk_used_percent{mount="/",disk="rootfs"} 0.127
```

This metric tells us the current percent disk usage as a decimal for the specified mount and disk at `12.7%`

### Histogram

A complex metric type, a histogram creates multiple metrics, it’s used to separate time-series into buckets, you’d use this to represent durations i.e. request latency

```
api_request_latency{path="/health",le="0} 0
api_request_latency{path="/health",le="10} 10
api_request_latency{path="/health",le="100} 12
api_request_latency{path="/health",le="1000} 5
api_request_latency{path="/health",le="2000} 1
api_request_latency_sum{path="/health",} 8524
api_request_latency_count{path="/health"} 28
```

Let's take the metrics that have an `le` label, what we're doing is increasing the counter for each labelled bucket for each request latency that falls into that bucket, so from this, we can see that `le=1000` having a value of `5` means that we've seen that latency 5 times overall.

Usually, this is implemented through an abstraction in the implementing SDK, in GO it's using the `.Observe()` method, this will automatically bucket things into the pre-defined buckets

Histograms have had the most notable changes in terms of architecture, but i won't go into detail about that in this post.

## Logs to Metrics

One tool i've found great to ease the transition into prometheus metric production is using "grok-exporter", the principle is to parse structured logs and count occurrences of patterns, for example from the following logs

```
{time="1658934286", method="get", endpoint="/api/v1/nodes"}
{time="1658934287", method="delete", endpoint="/api/v1/nodes"}
{time="1658934288", method="get", endpoint="/api/v1/nodes"}
{time="1658934290", method="get", endpoint="/api/v1/nodes"}
{time="1658934290", method="get", endpoint="/api/v1/nodes"}
... +96 more
```

We could look to produce the following metrics

```
http_requests_total{method="delete",endpoint="/api/v1/nodes"} 1
http_requests_total{method="get",endpoint="/api/v1/nodes"} 100
```

Shameless plug of a repository I  put together ([0xste/grok-exporter](https://github.com/0xste/grok-exporter)) to containerize another project which does this ([fstab/grok-exporter](https://github.com/fstab/grok_exporter))

You can make use of the configuration below to parse a simple json formatted log, or alternatively a more comprehensive example including separating out the parser configuration is found in this contribution to the OSS project for MEV-Boost https://github.com/flashbots/mev-boost/issues/370#issuecomment-1271715154

```
global:
  config_version: 3
input:
  type: file
  paths:
    - ./my-log-file.log
  readall: true
  fail_on_missing_logfile: true
imports:
  - type: grok_patterns
    dir: ./patterns
metrics:
  - type: counter
    name: http_requests_total
    help: The the number of http requests
    match: 'endpoint=%{URIPATH:path}'
    labels:
      path: '{.endpoint}'
server:
  port: 9144
```

## Check yourself before you wreck yourself

### Cardinality

Remember that every unique combination of key-value label pairs represents a new active time series, which can dramatically increase the amount of data stored. 

Cardinality is how many unique values of something there are. So for example a label containing HTTP methods would have a cardinality of 2 if you had only GET and POST in your application.


![cardinality](assets/img/prometheus/cardinality.png)


It's generally advised not to use labels with high cardinality (many different label values), such as user IDs, email addresses, or other unbounded sets of values. 

This can grow to be incredibly unwieldly, and is always something that engineers should keep in mind when using the various SDKs, as for the most part, there are no guard rails!


## In the next post

In the next post we'll look to explore the prometheus SDK in GO, and start to explore some practical examples.
