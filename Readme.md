# go-http-metrics [![Build Status][travis-image]][travis-url] [![Go Report Card][goreport-image]][goreport-url] [![GoDoc][godoc-image]][godoc-url]

go-http-metrics knows how to measure http metrics in different metric formats, it comes with a middleware that will measure metrics of a Go net/http handler. The metrics measured are based on [RED] and/or [Four golden signals], follow standards and try to be measured in a efficient way.

If you are using a framework that isn't directly compatible with go's `http.Handler` interface from the std library, do not worry, there are multiple helpers available to get middlewares fo the most used http Go frameworks. If there isn't you can open an issue or a PR.

## Metrics recorder implementations

go-http-metrics is easy to extend to different metric backends by implementing `metrics.Recorder` interface.

- [Prometheus][prometheus-recorder]

## Framework compatibility middlewares

The middleware is mainly focused to be compatible with Go std library using http.Handler, but it comes with helpers to get middlewares for other frameworks or libraries.

**The different helpers are on separate packages so when you import the project it doesn't import other framework packages and dependencies, for example if I don't use Negroni and instead I use std go net/http, it wouldn't be nice to import Negroni on my project.**

- [Negroni][negroni-example]
- [httprouter][httprouter-example]
- [go-restful][gorestful-example]

## Getting Started

A simple example that uses Prometheus as the recorder with the standard Go handler.

```golang
package main

import (
    "log"
    "net/http"

    "github.com/prometheus/client_golang/prometheus/promhttp"
    metrics "github.com/slok/go-http-metrics/metrics/prometheus"
    "github.com/slok/go-http-metrics/middleware"
)

func main() {
    // Create our middleware.
    mdlw := middleware.New(middleware.Config{
        Recorder: metrics.New(metrics.Config{}),
    })

    // Our handler.
    myHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("hello world!"))
    })
    h := mdlw.Handler("", myHandler)

    // Serve metrics.
    log.Printf("serving metrics at: %s", ":9090")
    go http.ListenAndServe(":9090", promhttp.Handler())

    // Serve our handler.
    log.Printf("listening at: %s", ":8080")
    if err := http.ListenAndServe(":8080", h); err != nil {
        log.Panicf("error while serving: %s", err)
    }
}
```

For more examples check the [examples]. [default][default-example] and [custom][custom-example] are the examples for Go net/http std library users.

## Metrics

The metrics obtained with this middleware are the [most important ones][red] for a HTTP service.

The middleware will measure the latency seconds of the requests using a histogram (latency), this will give us also the number of requests (rate), and the metric has the status codes (error rate):

### Query examples

Get the request rate by handler:

```text
sum(
    rate(http_request_duration_seconds_count[30s])
) by (handler)
```

Get the request error rate:

```text
rate(http_request_duration_seconds_count{code=~"5.."}[30s])
```

Get percentile 99 of the whole service:

```text
histogram_quantile(0.99,
    rate(http_request_duration_seconds_bucket[5m]))
```

Get percentile 90 of each handler:

```text
histogram_quantile(0.9,
    sum(
        rate(http_request_duration_seconds_bucket[10m])
    ) by (handler, le)
)
```

## Options

### Middleware Options

The factory options are the ones that are passed in the moment of creating the middleware factory using the `middleware.Config` object.

#### Recorder

This is the implementation of the metrics backend, by default it's a dummy recorder.

#### GroupedStatus

Storing all the status codes could increase the cardinality of the metrics, usually this is not a common case because the used status codes by a service are not too much and are finite, but some services use a lot of different status codes, grouping the status on the `\dxx` form could impact the performance (in a good way) of the queries on Prometheus (as they are already aggregated), on the other hand it losses detail. For example the metrics code `code="401"`, `code="404"`, `code="403"` with this enabled option would end being `code="4xx"` label. By default is disabled.

#### Custom handler ID

One of the options that you need to pass when wrapping the handler with the middleware is `handlerID`, this has 2 working ways.

- If an empty string is passed `mdwr.Handler("", h)` it will get the `handler` label from the url path. This will create very high cardnialty on the metrics because `/p/123/dashboard/1`, `/p/123/dashboard/2` and `/p/9821/dashboard/1` would have different `handler` labels. **This method is only recomended when the URLs are fixed (not dynamic or don't have parameters on the path)**.

- If a predefined handler ID is passed, `mdwr.Handler("/p/:userID/dashboard/:page", h)` this will keep cardinality low because `/p/123/dashboard/1`, `/p/123/dashboard/2` and `/p/9821/dashboard/1` would have the same `handler` label on the metrics.

There are different parameters to set up your middleware factory, you can check everything on the [docs] and see the usage in the [examples].

### Prometheus recorder options

#### Prefix

This option will make exposed metrics have a `{PREFIX}_` in fornt of the metric. For example if a regular exposed metric is `http_request_duration_seconds_count` and I use `Prefix: batman` my exposed metric will be `batman_http_request_duration_seconds_count`. By default this will be disabled or empty, but can be useful if all the metrics of the app are prefixed with the app name.

#### DurationBuckets

DurationBuckets are the buckets used for the request duration histogram metric, by default it will use Prometheus defaults, this is from 5ms to 10s, on a regular HTTP service this is very common and in most cases this default works perfect, but on some cases where the latency is very low or very high due the nature of the service, this could be changed to measure a different range of time. Example, from 500ms to 320s `Buckets: []float64{.5, 1, 2.5, 5, 10, 20, 40, 80, 160, 320}`. Is not adviced to use more than 10 buckets.

#### Registry

The Prometheus registry to use, by default it will use Prometheus global registry (the default one on Prometheus library).

## Benchmarks

```text
pkg: github.com/slok/go-http-metrics/middleware
BenchmarkMiddlewareHandler/benchmark_with_default_settings.-4            1000000              1202 ns/op             256 B/op          6 allocs/op
BenchmarkMiddlewareHandler/benchmark_with_grouped_status_code.-4         1000000              1356 ns/op             256 B/op          7 allocs/op
BenchmarkMiddlewareHandler/benchmark_with_predefined_handler_ID-4        1000000              1019 ns/op             256 B/op          6 allocs/op

```

[travis-image]: https://travis-ci.org/slok/go-http-metrics.svg?branch=master
[travis-url]: https://travis-ci.org/slok/go-http-metrics
[goreport-image]: https://goreportcard.com/badge/github.com/slok/go-http-metrics
[goreport-url]: https://goreportcard.com/report/github.com/slok/go-http-metrics
[godoc-image]: https://godoc.org/github.com/slok/go-http-metrics?status.svg
[godoc-url]: https://godoc.org/github.com/slok/go-http-metrics
[docs]: https://godoc.org/github.com/slok/go-http-metrics
[examples]: examples/
[red]: https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/
[four golden signals]: https://landing.google.com/sre/book/chapters/monitoring-distributed-systems.html#xref_monitoring_golden-signals
[default-example]: examples/default
[custom-example]: examples/custom
[negroni-example]: examples/negroni
[httprouter-example]: examples/httprouter
[gorestful-example]: examples/gorestful
[prometheus-recorder]: metrics/prometheus