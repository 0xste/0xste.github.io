---
title: Running Prometheus
date: 2024-10-03 03:00:00 +0000
categories: [Observability]
tags: [software,engineering,prometheus,opensource]
image:
  path: /assets/img/prometheus/running.png
  alt: Prometheus
---

In this blog, we'll explore running prometheus locally

In the latest version of prometheus, we'll explore some of the new features including the new UI

https://prometheus.io/blog/2024/09/11/prometheus-3-beta/#opentelemetry-support

## Getting Started

To start using Prometheus, you typically:

1. Deploy the Prometheus server
2. Configure Prometheus to scrape metrics from your targets
3. (Optional) Set up Alertmanager for notifications
4. (Optional) Connect Grafana or another visualization tool for dashboarding

### 1. Deploy Prometheus

Going to assume you have docker running on your machine, if not the setup guide [here](https://docs.docker.com/get-started/get-docker/) should cover you.

We need to write some configuration for prometheus, but the default will do us for now, by default we'll register the metamonitoring endpoint i.e. metrics from the underlying prometheus server.

We'll make use of the latest version of prometheus that's still in beta, it gives us a shinier UI as well as some [OpenTelemetry](https://prometheus.io/blog/2024/03/14/commitment-to-opentelemetry/) features we'll look at later in the series
```
# run prometheus server
docker run -d \
    -p 9090:9090 \
    -v prometheus-data:/prometheus \
    --network appnet \
    --name prometheus \
    prom/prometheus:v3.0.0-beta.0
```
This will start a prometheus server, expose the UI on `localhost:9090`, ensure that any metrics ingested are persisted to disk, create a docker overlay network called `appnet` and launch a container called `prometheus`


We can check this by running `docker ps`

![alt text](assets/img/prometheus/docker-ps.png)

We can open up `localhost:9090` in a browser, and we can have a poke around the configuration ui

![alt text](assets/img/prometheus/gui.png)

We can even begin to run some queries on the native prometheus metrics, but we can revisit this soon.

### 2. Configuring our app

Given we're launching prometheus in docker, we'll make use of docker networking, so for our app from the previous post, we'll make use of the following dockerfile. For our purposes this will do for this purpose

```
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine as BUILD
WORKDIR /src
COPY . .
RUN ls -ltra
RUN go build -o /bin/blog ./main.go

FROM alpine:3.20
COPY --from=BUILD /bin/blog /bin/blog
CMD ["/bin/blog"]
```

We can then build and deploy this locally following something like

```
# create the network, we'll use this later
docker network create --attachable -d bridge appnet

# build image
docker build -t 0xste/blog-api .

# run and expose app and prometheus ports in the appnet
docker run -d --network appnet -p 9000:9000 -p 8080:8080 --name blog-api 0xste/blog-api
```

### 2. Configuring a scraper

We can now look at applying a configuration to retrieve from the above published metrics endpoint.

We can create a file at `./config/prometheus.yml`, the part to note is that because we're using docker networking, we're using the bridge network we defined above to communicate with `blog-api` container using the container name.

```
# my global config
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - localhost:9090
  - job_name: 0xste_blog
    static_configs:
      - targets:
          - blog-api:9000
```

We can then follow the same process as above, in which we run and start our prometheus endpoint, note the command below references the configuration explicitly and mounts it to the container.

```
# run prometheus server
docker run -d \
    -p 9090:9090 \
    -v ${PWD}/config/prometheus.yml:/etc/prometheus/prometheus.yml \
    -v prometheus-data:/prometheus \
    --network appnet \
    --name prometheus \
    prom/prometheus:v3.0.0-beta.0
```

If we jump back into the UI by navigating to `localhost:9090/targets` we can see if our configuration is correct

![alt text](assets/img/prometheus/targets.png)

If we're all green here, then we're ingesting metrics!


## PromQL

A lot of people diving deeper into these kinds of dashboards, or trying to implement themselves pretty quickly find themselves running into the main query language for Prometheus timeseries.

Anecdotally, engineers i've worked with on this agree that the learning curve for PromQL is... quite steep.

It may sometimes feel like you're chained to a rock, having your liver pecked out by an eagle every day for eternity when trying to get a PromQL query to do what you expect it to. (a terrible mythology joke)

### Query Basics

We'll often see in UIs for metric query engines a few query options you can play around with

You usually see two types of queries: 
- In grafana, it's referred to as "Instant" or "Range"
- In Prometheus, it's referred to as "Table" or "Graph"

we'll usually see a overarching time filter in the top right of most query engines, to dictate the range of the query, which is different from the query range period, which we'll get into a bit more later on in this guide.

### Common Counter Queries

So let's start with some basics, we'll explore more nuance of the language later

So first, let's make some requests to our api, opening a browser at `localhost:8080/123` will suffice.

If we navigate to our query viewer in the GUI, we'll be able to construct some queries

#### Requests by path over time

This function makes use of the `increase` operator, with a period of `[30s]`, this tells the query engine to look only at the delta between the first and last value in a 30s contiguous period for a monotonically increasing counter.

The `max by` aggregation simply rolls up the labels across all metric series so we're able to view only the labels we care about, this is not particularly relevant for our query, but may be as we scale up our queries

Simply, this will give us the overall count of requests by path

```
max by (path)(increase(http_request_count[30s]))
```

#### Overall API Uptime

If we wanted to take this a step further, we'd look to export an additional label for `response_status`, to which we can form a query that will get us our overall request success percentage

```
sum(
  increase(
    http_request_count{response_status!~"5[0,1,2,4-9][0,2-9]"}[$__range]
  )
)
/ 
sum(
  increase(
    http_request_count{}[$__range]
  )
)
```

In this query we're making use of a more complex setup, in that we're matching labels that match a regex for our `response_status` label, this will get us a set of requests that are non-200 range responses.

The second part of the query, we return all responses, we're able to use the `/` operand to divide the two results, and we come out with the overall uptime of the service, this can be useful for overall SLA reporting.



## Conclusion

In a world where system complexity is ever-increasing, Prometheus offers a robust, scalable solution for monitoring and alerting. Its ability to handle dynamic environments, powerful query language, and extensive ecosystem make it an invaluable tool for DevOps and SRE teams.

Whether you're running a small cluster or a large-scale distributed system, Prometheus provides the visibility you need to ensure your services are running smoothly. As you dive deeper into the world of observability, you'll find Prometheus to be a trusty companion in your journey towards more reliable, performant systems.

