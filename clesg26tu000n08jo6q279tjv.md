---
title: "How to use Prometheus for web application monitoring"
seoTitle: "How to use Prometheus for web application monitoring"
seoDescription: "In this post, I would like to show how we can implement web application monitoring using Prometheus"
datePublished: Fri Mar 03 2023 11:20:27 GMT+0000 (Coordinated Universal Time)
cuid: clesg26tu000n08jo6q279tjv
slug: how-to-use-prometheus-for-web-application-monitoring
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1677842405923/869817c4-e36e-4b72-a9e7-05f20acce48b.webp
tags: python, learning, monitoring, devops, prometheus

---

## Intro

The ability to comprehend and assess a system's state using the data it produces is known as observability. It is one of the most important traits of system reliability that gives an opportunity to take action. Observability is crucial, especially in distributed systems where end-to-end testing is really difficult or even impossible. There are three pillars of observability:

* logging (record events: what + when)
    
* metrics (provide a state of a system using numeric values)
    
* tracing (follow operations that traverse through different systems/services)
    

In this post, I would like to cover only the metrics part and show how we can implement web application monitoring using [Prometheus](https://prometheus.io/). We discuss how to choose some metrics as Service Level Indicators (SLIs) and also how to ensure system reliability by setting proper targets as System Level Objectives (SLOs).

## About Prometheus

The aim of Prometheus is quite simple: to have a metrics server that gathers data from several systems or applications and aggregates them in one place. Prometheus collects metrics by scraping targets that expose metrics using HTTP endpoints.

The Prometheus server consists of three components:

* HTTP server - accepts queries for metrics
    
* Time-series database (TSDB) - store for metrics
    
* Retrieval - data scraping worker responsible for retrieving metrics from sources that expose metrics
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677588844183/dc3d52dd-e029-4b95-98e2-6ecfbb450509.png align="center")

Exporters are processes that run on Prometheus targets responsible for serving metrics and presenting them in an easily consumable format for Prometheus. Prometheus fetches the data from exporters in a pull-based manner (there is also [pushgateway](https://prometheus.io/docs/practices/pushing/) approach, but we would not cover it here). There are a lot of native exporters that can be used, e.g. [Node exporter](https://prometheus.io/download/#node_exporter) (Linux Server), [MySQL exporter](https://prometheus.io/download/#mysqld_exporter), etc. Each service that exposes an HTTP endpoint with properly formatted metric data can be an exporter.

Prometheus is using PromQL language for defining queries for metrics. Each metric has the following structure:

```bash
metric_name{label1=<label1>, ..., labelN=<labelN>} metric_value
```

For example:

```bash
up{instance="localhost:9090", job="prometheus"} 1
```

We can define a set of labels for a metric that can be used for filtering. However, each metric has two labels provided by default: `instance` and `job`. When Prometheus scrapes a target it also registers the timestamp (UNIX timestamp) when the metric was scraped. So in the database, we have stored the time series of metric values. Prometheus defines four different types of metrics:

* Counter - how many X occurs (can only increase), e.g total number of requests
    
* Gauge - what is the current value of X (can change over time), e.g. number of active requests
    
* Histogram - group observations in discrete buckets, e.g. request responses time below 1s
    
* Summary - similar to the Histogram and tell how many percent of observations are below the given value
    

## Prometheus server setup

Prometheus server has to be run, in order to start gathering data needed in system monitoring. To deploy it, we utilize a simple docker-compose file (using the latest `prom/prometheus` image and exposing `9090` port to get access to the Prometheus User Interface).

```yaml
# docker-compose.yml
version: "3.7"

services:
  prometheus:
    image: prom/prometheus
    ports:
      - target: 9090
        published: 9090
    volumes:
      - type: bind
        source: ./prometheus.yml
        target: /etc/prometheus/prometheus.yml
```

To make it works, we must also provide a Prometheus configuration file `prometheus.yml` that would be bound to `/etc/prometheus/prometheus.yml` inside running Prometheus container.

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
```

The `scrape_configs` section defines a list of targets for metrics pulling. In the beginning, the only source of metrics will be our running Prometheus server. So the Prometheus server would be scraping itself.

Now we can run our docker-compose file and test how it is working:

```bash
$ docker-compose up
```

As a metrics exporter, the Prometheus server exposes data at the default `/metrics` endpoint. Here (at `http://localhost:9090/metrics`), we have a piece of how Prometheus server metrics look alike:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677580273826/7ca7c8bf-1706-4ee7-822b-251c49360761.png align="center")

By hitting `http://localhost:9090` , we can access the Prometheus UI and check whether our configuration is working properly.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677582368209/fec4de25-9c1e-4976-b3fd-3cb0c0796805.png align="center")

There is only one metrics target that is up and running - and it is the Prometheus server.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677582431680/6c945fdf-5eeb-4c00-ac36-8c0df2c5b254.png align="center")

The next step will be adding our web application as a new metrics exporter.

## Web application exporter

Our new exporter will be a simple web application built in the Python [Flask](https://flask.palletsprojects.com/en/2.2.x/) framework. For Prometheus integration, we take the [prometheus\_client](https://github.com/prometheus/client_python#flask) library.

```python
# app.py
from flask import Flask
from prometheus_client import make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware

app = Flask(__name__)
app.wsgi_app = DispatcherMiddleware(
    app.wsgi_app, {"/metrics": make_wsgi_app()}
)

@app.route("/")
@app.route("/up")
def up():
    return "I am running!"
```

Here are the packages that will be used by the application:

```bash
# requirements.txt
Flask==2.2.3
prometheus-client==0.16.0
```

We have a simple HTTP `/up` endpoint and also `/metrics` endpoint provided by the library. The next step will be defining `Dockerfile`:

```bash
FROM python:alpine
COPY app.py requirements.txt /
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["flask", "run", "-h", "0.0.0.0", "-p", "8000"]
```

and building a container image:

```bash
$ docker build -t web-application-monitoring:v0 .
```

To integrate it with the Prometheus server, we have to do two things:

* add a new scrape target in `prometheus.yml` config file
    

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "web-app"
    scrape_interval: 5s
    static_configs:
      - targets: ["web-app:8000"]
```

* add our application docker image to the docker-compose file:
    

```yaml
# docker-compose.yml
version: "3.7"

services:
  prometheus:
    image: prom/prometheus
    ports:
      - target: 9090
        published: 9090
    volumes:
      - type: bind
        source: ./prometheus.yml
        target: /etc/prometheus/prometheus.yml
  web-app:
    image: web-app-monitoring:v0
    ports:
      - target: 8000
        published: 8080
```

The application will be exposed to the external world on port `8080`, so we can test that everything is working by hitting `http://localhost:8080`. We have also some metrics exposed (at `http://localhost:8080/metrics` ) by default (provided by `promethus_client` library):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677677254682/5bed7656-4efd-4327-a7b7-2dd503b63466.png align="center")

Our Prometheus server have also a new target running and accessible for scraping:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677677833445/80125216-14d5-4560-94b2-eb9d1413fd98.png align="center")

## HTTP traffic metrics

We have some metrics provided for our exporter by `prometheus_client`. But these metrics are not tell anything about our web HTTP endpoints. For better readability we disable it for now:

```python
# app.py 

import prometheus_client

prometheus_client.REGISTRY.unregister(prometheus_client.GC_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PLATFORM_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PROCESS_COLLECTOR)
```

As far as HTTP traffic is concerned, we are mostly interested in the following data:

* how many requests it is handling
    
* how many errors occur
    
* what is requests latency
    

For the purpose of this article, I would like to introduce a kind of synthetic endpoint that would return different HTTP responses and durations to produce some data for metrics gathering (It is of course redundant and unnecessary in a working system).

```python
# app.py

import random
from time import sleep

@app.route("/synthetic")
def synthetic():
    random_duration = random.randint(1, 50) * 0.001
    sleep(random_duration)
    response_code = random.choice([200, 200, 200, 200, 200, 400, 401, 500])
    return Response(str(response_code), status=response_code)
```

The next step will be preparing some code that monitors our endpoint:

```python
from prometheus_client import Histogram

HTTP_REQUEST_DURATION = Histogram(
    "http_request_duration",
    "Requests durations",
    ["method", "url", "code"],
    buckets=[0.01, 0.1, 0.5, 2, float("inf")],
)

def observe_http(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        response = func(*args, **kwargs)
        end = time.time()
        HTTP_REQUEST_DURATION.labels(
            method=request.method,
            code=response.status_code,
            url=request.url,
        ).observe(end - start)
        return response
    return wrapper

@app.route("/synthetic")
@observe_http
def synthetic():
    ...
```

The HTTP `/synthetic` endpoint is decorated by `observe_http` function that collects `HTTP_REQUEST_DURATION` metric. For our histogram we declared:

* three labels: `method`, `code`, `url` (that metric can be grouped by)
    
* five buckets:
    
    * less than 10ms
        
    * less than 100ms
        
    * less than 500ms
        
    * less than 2s
        
    * less than infinity
        

The above histogram provides us with a metric for each bucket with `_bucket` suffix but also `_count` and `_sum` metrics.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1677792035819/abcff24c-ed0d-463f-87b5-8f06bfde6f7d.png align="center")

## Service Level Indicator

When designing a solution it is important to specify measurable targets/goals of the working system. These targets can help to determine the level of reliability of the system (e.g. by users).

Service Level Indicator (SLI) is a quantitative measure of some aspects of the level of service that is provided, e.g. number of errors, latency, availability, etc. The RED (Rate, Errors, Duration) method is one of the widely known techniques that can serve as a source of SLIs. It consists of:

* **Rate** - number of requests per second
    

```bash
sum(rate(http_request_duration_count{job="web-app"}[1m]))
```

* **Errors** - the number of those requests that are failing
    

```bash
sum(rate(http_request_duration_count{job="web-app",
status_code~=”5..”}[1m]))
```

* **Duration** - the amount of time those requests take
    

```bash
histogram_quantile(0.99, sum(rate(http_request_duration_bucket
{job="web-app"}[1m])) by (le))
```

If you want to get more understanding of this approach, I would highly recommend [Tom Wilkie talk](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/) about how to instrument services.

While the RED method can function as an SLI, it is usually recommended (see [Google SRE book](https://sre.google/sre-book/service-level-objectives/)) to take SLI as the ratio of two numbers: the number of good events (e.g. with a response code different from `5xx`) divided by the total number of events. Thanks to it our indicator is always between 0 and 1, and we can easily set a target for our system - Service Level Objective (SLO), e.g 98% requests without errors. It enables us to trigger alerts when our system reaches a level that is below our SLO. The value for SLOs must be carefully chosen to avoid excessive alert calls but guarantee that system is stable at a promised level.

## Conclusion

The article has covered the Prometheus server setup, web application exporter, and HTTP traffic metrics. Additionally, the article has introduced the Service Level Indicator (SLI) and Service Level Objective (SLO) concepts, which can help determine the level of reliability of a system.

Although this post covers some aspects of system monitoring based on metrics, there is a lot more that should be done to have an application production ready. Prometheus is a really good tool for metrics gathering. However, it may not best choice for visualization and alerting. For that need, we should probably turn to [Grafana](https://grafana.com/) and [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/).

All stuff covering Prometheus setup and web application code can be found in the GitHub [repo](https://github.com/jorzel/web-app-monitoring). If you want to dive deeper into Prometheus I strongly recommend this [course](https://kodekloud.com/courses/prometheus-certified-associate-pca/).

I hope it would be helpful.