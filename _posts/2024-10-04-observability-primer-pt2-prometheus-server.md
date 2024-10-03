---
title: Prometheus Primer
date: 2024-10-03 02:00:00 +0000
categories: [Blogging]
tags: [software,architecture]
image:
  path: /assets/img/prometheus/prometheus-fire.jpg
  alt: Prometheus
---

In this post, we'll look specifically at Prometheus, and the query language itself

## What is Prometheus?

A lot of people's exposure to Prometheus is through viewing various Grafana dashboards, a few popular public ones come to mind, famously: [Wikipedia](https://grafana.wikimedia.org/?orgId=1), [CNCF DevStats](https://devstats.cncf.io/), and the [Large Hadron Collider](http://monit-grafana-open.cern.ch/goto/QInjqFzNR?orgId=16) at CERN.

A lot of people diving deeper into these kinds of dashboards, or trying to implement themselves pretty quickly find themselves running into the main query language for Prometheus timeseries.

Anecdotally, engineers i've worked with on this agree that the learning curve for PromQL is... quite steep.

It may sometimes feel like you're chained to a rock, having your liver pecked out by an eagle every day for eternity when trying to get a PromQL query to do what you expect it to.

## A practical Example

Go is my main language of choice these days, [here](https://github.com/flashbots/mev-boost/pull/686) we can find a simple go http server implementation in a core Ethereum OSS project.

We'll look to break this down further:

In our main process, we'll look to construct us a prometheus.Registry, this ultimately holds a reference to all the metrics we want to register

```
	prometheusRegistry := prometheus.NewRegistry()
	if err := prometheusRegistry.Register(collectors.NewGoCollector()); err != nil {
		log.WithError(err).Error("Failed to register metrics for GoCollector")
	}
	if err := prometheusRegistry.Register(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})); err != nil {
		log.WithError(err).Error("Failed to register ProcessCollector")
	}
```

We register some default collectors here for the purposes of 
