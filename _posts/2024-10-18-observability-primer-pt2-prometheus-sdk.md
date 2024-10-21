---
title: Prometheus in the wild
date: 2024-10-18 01:00:00 +0000
categories: [Observability]
tags: [software,engineering,prometheus,openmetrics,opensource]
image:
  path: /assets/img/prometheus/prometheus-fire.jpg
  alt: Prometheus
---

In this post, we'll look specifically at Prometheus, and it's query language, PromQL

## What is Prometheus?

A lot of people's exposure to Prometheus is through viewing various Grafana dashboards, a few popular public ones come to mind, famously: [Wikipedia](https://grafana.wikimedia.org/?orgId=1), [CNCF DevStats](https://devstats.cncf.io/), and the [Large Hadron Collider](https://monit-grafana-open.cern.ch/goto/QInjqFzNR?orgId=16) at CERN.


## A practical Example

Go is my main language of choice these days, [this PR](https://github.com/flashbots/mev-boost/pull/686) outlines an approach for instrumenting a core component for many blockchain node operators use when running validators. The MR instruments with a purposely minimal go http server implementation, and lays the groundwork for additional metric exploration, which we'll dig into more detail through this series.

### Registry Construction

In our main process, we'll look to construct us a prometheus.Registry, this ultimately holds a reference to all the metrics we want to register

If your go code makes use of Dependency Injection, you'll be able to pass this `prometheusRegistry` construct around and have each module define custom metrics on this for export.
```
prometheusRegistry := prometheus.NewRegistry()
if err := prometheusRegistry.Register(collectors.NewGoCollector()); err != nil {
	log.WithError(err).Error("Failed to register metrics for GoCollector")
}
if err := prometheusRegistry.Register(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})); err != nil {
	log.WithError(err).Error("Failed to register ProcessCollector")
}
```


`NewGoCollector` exports native go metrics about the current Go process using `debug.GCStats`, and `runtime/metrics`, where `NewProcessCollector` registers the current state of process metrics including CPU, memory and file descriptor usage as well as the process start time

At this stage, you've just constructed the defaults for some simple instrumentation in the registry, we'll look into this some more once we've exported the server runtime

### Starting the prometheus server

We then look to run a standalone server on a new port for exporting the prometheus runtime, it's generally best practice to run the process on it's own management port so you're not exposing ports externally to customers unneccessarily

In this example, we make use of a pattern i've seen a few times in OSS go, in which your function is implemented as a reciever, and your registry is injected to the "server" struct. 
```
// StartMetricsServer starts the HTTP server for exporting open-metrics
func (m *BoostService) StartMetricsServer() error {
	if m.prometheusRegistry == nil {
		return errNilPrometheusRegistry
	}
	serveMux := http.NewServeMux()
	serveMux.Handle("/metrics", promhttp.HandlerFor(m.prometheusRegistry, promhttp.HandlerOpts{
		ErrorLog:          m.log,
		EnableOpenMetrics: true,
	}))
	return http.ListenAndServe(
		fmt.Sprintf(":%d", m.prometheusListenAddr),
		serveMux,
	)
}
```

For this particular use-case, we run it as a goroutine that can error without failing the whole process, for the purposes of not impacting the main runtime for MEV-Boost

```
		go func() {
			log.Infof("Metric Server Listening on %d", opts.PrometheusListenAddr)
			if err := service.StartMetricsServer(); err != nil {
				log.WithError(err).Error("metrics server exited with error")
			}
		}()
```

### A standalone example

We'll break this down a little further in a practical runnable example, we'll use this as the starting point of the code, but ultimately implements the same pattern as we have explored in the mev-boost pull request above and should feel familiar.

```
package main

import (
	"fmt"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/collectors"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"golang.org/x/sync/errgroup"
	"log"
	"net/http"
)

func main() {
	prometheusRegistry := prometheus.NewRegistry()
	if err := prometheusRegistry.Register(collectors.NewGoCollector()); err != nil {
		log.Printf("Failed to register metrics for GoCollector")
	}
	if err := prometheusRegistry.Register(collectors.NewProcessCollector(collectors.ProcessCollectorOpts{})); err != nil {
		log.Printf("Failed to register ProcessCollector")
	}

	const (
		serverPort  = 8080
		metricsPort = 9000
	)

	server := &http.Server{
		Addr: fmt.Sprintf(":%d", serverPort),
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Header().Set("Content-Type", "application/json")
			_, _ = w.Write([]byte(`{"hello":"world"}`))
			return
		})}

	metricsServer := NewMetricsServer(prometheusRegistry, metricsPort)

	eg := errgroup.Group{}
	eg.Go(metricsServer.ListenAndServe)
	eg.Go(server.ListenAndServe)

	if err := eg.Wait(); err != nil {
		log.Fatalf("main process exited with error %v", err)
	}

}

type MetricsServer struct {
	registry          *prometheus.Registry
	metricsListenAddr int
}

func NewMetricsServer(registry *prometheus.Registry, metricsListenAddr int) *MetricsServer {
	return &MetricsServer{
		registry:          registry,
		metricsListenAddr: metricsListenAddr,
	}
}

func (s *MetricsServer) ListenAndServe() error {
	if s.registry == nil {
		return fmt.Errorf("nil metrics registry")
	}
	serveMux := http.NewServeMux()
	serveMux.Handle("/metrics", promhttp.HandlerFor(s.registry, promhttp.HandlerOpts{
		EnableOpenMetrics: true,
	}))
	return http.ListenAndServe(
		fmt.Sprintf(":%d", s.metricsListenAddr),
		serveMux,
	)
}
```

This gives us enough to now start our serve, and check our custom metrics port on `localhost:9000/metrics` to see our go runtime metrics, they should look something like the following:

We'll not spend too much time looking into these metrics, we'll explore them some more in the next post

```
# HELP go_gc_duration_seconds A summary of the wall-time pause (stop-the-world) duration in garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
go_gc_duration_seconds{quantile="0.5"} 0
go_gc_duration_seconds{quantile="0.75"} 0
go_gc_duration_seconds{quantile="1"} 0
go_gc_duration_seconds_sum 0
go_gc_duration_seconds_count 0
# HELP go_gc_gogc_percent Heap size target percentage configured by the user, otherwise 100. This value is set by the GOGC environment variable, and the runtime/debug.SetGCPercent function. Sourced from /gc/gogc:percent
# TYPE go_gc_gogc_percent gauge
go_gc_gogc_percent 100
# HELP go_gc_gomemlimit_bytes Go runtime memory limit configured by the user, otherwise math.MaxInt64. This value is set by the GOMEMLIMIT environment variable, and the runtime/debug.SetMemoryLimit function. Sourced from /gc/gomemlimit:bytes
# TYPE go_gc_gomemlimit_bytes gauge
go_gc_gomemlimit_bytes 9.223372036854776e+18
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 10
# HELP go_info Information about the Go environment.
# TYPE go_info gauge
go_info{version="go1.23.1"} 1
```


## Registering custom metrics

Now we have a baselined starting point, we can start looking at registering custom metrics following some pretty widely used design patterns i've used personally based on a few OSS examples.

A pattern i like for this is having a struct to manage in each module we want to register metrics, which takes the registry as a function argument and returns a `{Module}_metrics` package for use

We'll dive into this pattern in this section

Here, we register two metrics, note the difference between the `Counter` and `CounterVec`

As discussed more in the OpenMetrics Primer blog post, if we want to expose custom metrics we'll use labels in most cases, but there are some benefits to using the raw unlabelled `Counter`, but for the most part we'll be using `CounterVec` in most of our use-cases

An additional note on this, in terms of the metric `Name` property that's passed in via `CounterOpts`, there are some additional properties there for `Namespace` and `Subsystem` which i tend to prefer not to use, the main reason is so i can quickly find the exact metric in the code, and where it's used, particularly useful when you have a large codebase with lots of modules that register custom metrics

```
type HttpMetrics struct {
	requestCount   *prometheus.CounterVec
	requestCounter prometheus.Counter
}

func NewHttpMetrics(r prometheus.Registerer) *HttpMetrics {
	return &HttpMetrics{
		requestCount: promauto.With(r).NewCounterVec(
			prometheus.CounterOpts{
				Name: "http_request_count",
				Help: "the count of api requests",
			}, []string{"path"},
		),
		requestCounter: promauto.With(r).NewCounter(
			prometheus.CounterOpts{
				Name: "http_request_counter",
				Help: "the count of api requests",
			},
		),
	}
}
```

So we now look to make use of this custom metric, so we'll modify our code from before to include a few additions

We'll construct a HttpMetrics instance like so
```
	serverMetrics := NewHttpMetrics(prometheusRegistry)

```

We'll then add the following line to our handler to make use of our metric
```
			serverMetrics.requestCount.WithLabelValues(r.URL.Path).Add(1)
```

Our code should look something like this
```
	serverMetrics := NewHttpMetrics(prometheusRegistry)
	server := &http.Server{
		Addr: fmt.Sprintf(":%d", serverPort),
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			serverMetrics.requestCount.WithLabelValues(r.URL.Path).Add(1)
			w.Header().Set("Content-Type", "application/json")
			_, _ = w.Write([]byte(`{"hello":"world"}`))
			return
		}),
	}
```

Which gives us the additional metrics alongside our native go metrics

If we navigate to our server on `localhost:9000/metrics` we can see our additional metrics

```
# HELP http_request_count the count of api requests
# TYPE http_request_count counter
http_request_count{path="/"} 1
http_request_count{path="/favicon.ico"} 2
http_request_count{path="/hello"} 1
```

## Wrapping Up

In this post we covered a few areas for how i generally look to implement prometheus metrics across projects through a few practical examples. In the next post, we'll look into some metric patterns we can use to optimize at the query side for the purposes of broader observability.
