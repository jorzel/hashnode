---
title: "The Idea of Synthetic Monitoring"
seoTitle: "The Idea of Synthetic Monitoring"
seoDescription: "In this post, I would like to show the process of building a basic synthetic monitoring service in Python that uses Prometheus to collect metrics."
datePublished: Thu Dec 12 2024 12:56:21 GMT+0000 (Coordinated Universal Time)
cuid: cm4lbp7r1000009mmbp42hexk
slug: the-idea-of-synthetic-monitoring
tags: programming, python, monitoring, devops, observability

---

## Introduction

Synthetic monitoring is a proactive strategy used to ensure the reliability, performance, and availability of systems by generating synthetic (artificial) traffic. It relies on scripted scenarios that mimic user interactions with your system. These scripted tests are run at regular intervals (e.g., every minute, five minutes, or hourly). The frequency of these tests depends on the criticality of the monitored paths and the desired level of monitoring granularity.

These scripted tests typically store data as metrics using traditional tools like Prometheus. By leveraging this data, we can analyze traffic patterns, identify early signs of issues, and configure additional alerts to notify us of any potential problems.

Synthetic monitoring does not replace traditional monitoring, which gathers data from real user traffic. Instead, it is a complementary approach that enhances your overall monitoring strategy. While conventional monitoring examines how the system behaves from an internal perspective, synthetic monitoring provides measurements from an external viewpoint (treating the system under observation as a black box).

The main benefits of this approach are:

* **Proactive Issue Detection**: Synthetic monitoring can detect issues before they impact real users by continuously running tests that simulate critical user interactions, helping you catch and fix problems early.
    
* **Continuous Monitoring**: Synthetic monitoring ensures your system is being tested and monitored even during periods of low or no real traffic. This is particularly useful outside of peak hours or during scheduled maintenance windows.
    
* **Testing Critical Paths**: You can focus on key user journeys or business processes, ensuring that the most important parts of your system are always functioning correctly.
    
* **Alerting for Low or Seasonal Traffic Services**: In scenarios where real traffic is disrupted or absent (e.g., a sudden drop in user activity or service with seasonal traffic), synthetic monitoring ensures that your system remains functional.
    

In the context of modern distributed systems, where failures can occur in many components, synthetic monitoring resembles a form of continuous testing. It is tough to perform a robust test for functionality that touches several components. This does not mean that we should abandon traditional testing. It is a great approach to proactively find issues in your codebase. However, it can sometimes be limited. Synthetic monitoring can be great supplementary technique to quickly detect problems that evade standard testing methods and regular observability approaches.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733991501155/e6323eee-3862-40d5-9193-12bb4cb84f37.png align="center")

In this post, I would like to show the process of building a basic synthetic monitoring service in Python that uses Prometheus to collect metrics.

## High-level Architecture

The high-level architecture consists of a simple reservations service built in Python with a REST API exposed, and an independently running monitor written in Python that periodically calls the `POST /reservations` endpoint. Both services are integrated with Prometheus, which collects the metrics.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1725027561267/bea81a4d-d752-4d5c-98b7-9686ff3a443e.png align="center")

## Prometheus Setup

To collect metrics from our services we need to configure the Prometheus instance. I presented the details of this process in one of the [previous posts](https://medium.com/gitconnected/how-to-use-prometheus-for-web-application-monitoring-524204e36413).

The first step is to define the Prometheus config file (`prometheus.yml`):

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 10s
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
```

At this point, the Prometheus instance will be scraping metrics only from itself (`localhost:9090`). When the Reservations Service and Synthetic Monitor are ready, we will have to update `scrape_configs` section to provide data from these services.

The Prometheus instance can be started using the Docker Compose file (`docker-compose.yml`):

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
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
```

We expose `9090` port to have access to Prometheus UI. The configuration file (`prometheus.yml`) which was defined several paragraphs above has to be mounted into the container using `volumes` section.

After hitting `docker compose up` in a terminal, we should see several Prometheus logs and information that the Prometheus container was created:

```bash
$ docker compose up
 ✔ Container synthetic-monitoring-prometheus-1  Created
```

We can visit `localhost:9090/targets` and check whether our scraping target (Prometheus instance) is recognized.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733931117647/da41e4da-08f9-4934-9044-1d87a504597b.png align="center")

Everything looks ok, the Prometheus instance (`prometheus`) is up and ready as a scraping target. We can go to the next step: implementing Reservations Service and connecting it to the running Prometheus instance.

## Service Under Observation

Reservations Service is a simple application written in Python `Flask` framework. It exposes a single HTTP endpoint: `/reservations`.

```python
# reservations-service/app.py
import random
import time

import structlog
from flask import Flask, request
from prometheus_client import Histogram, make_wsgi_app
from werkzeug import Response
from werkzeug.middleware.dispatcher import DispatcherMiddleware

app = Flask(__name__)
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {"/metrics": make_wsgi_app()})

import prometheus_client

# Disable unnecessary metrics
prometheus_client.REGISTRY.unregister(prometheus_client.GC_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PLATFORM_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PROCESS_COLLECTOR)

logger = structlog.get_logger()

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

@app.route("/reservations", methods=["POST"])
@observe_http
def reservations():
    logger.info("Reservation request", data=request.data)
    random_duration = (
        random.choice([1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3])
        * random.randint(1, 100)
        * 0.001
    )
    time.sleep(random_duration)

    response_code = random.choice(
        [
            200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200, 200,
            400, 401, 500
        ]
    )
    logger.info("Reservation response", code=response_code)
    return Response(str(response_code), status=response_code)
```

The implementation of the endpoint is not important for our topic. We only mimic the behavior using random duration times and response codes. The endpoint is wrapped into`observe_http` decorator that gathers `http_request_duration` histogram metric. We use `prometheus_client` library to expose stored metrics in `/metrics` endpoint.

We define a `Dockerfile` within `reservations-service` folder. It will be used to create an application image.

```bash
# reservations-service/Dockerfile
FROM python:3.9-alpine
COPY app.py requirements.txt /
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["flask", "run", "-h", "0.0.0.0", "-p", "8000"]
```

To build the image, we need to run within `reservations-service` folder following command:

```bash
$ docker build . reservations-service:v0 ./
```

Now, the service artifact is created and can be added to `docker-compose.yml` file.

```yaml
# docker-compose.yml
version: "3.7"

services:
  ...

  reservations-service:
    image: reservations-service:v0
    ports:
      - target: 8000
        published: 8080
```

The `prometheus.yml` file has to be updated with a new entry to gather metrics from Reservations Service:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 10s
rule_files:
  - "prometheus-rules.yml"
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "reservations-service"
    scrape_interval: 5s
    static_configs:
     - targets: ["reservations-service:8000"]
```

We have to use the hostname that was declared for Reservations Service in the `docker-compose.yml` . In our case it is `reservations-service`, so the target address will be `reservations-service:8000` . We can restart our Docker Compose and check whether the new target is visible in the Prometheus UI.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733938876964/5f3a67c0-fa6c-48a5-834d-b3e80f25a042.png align="center")

Reservations Service `/metrics` endpoint is visible for the Prometheus instance.

Now, we can hit `/reservations` endpoint several times, and check whether the metrics are collected.

```bash
$ curl -X POST http://localhost:8080/reservations
```

The Prometheus UI is not the greatest tool to visualize metrics. Nevertheless, using `localhost:9090/graphs` we can place a query (`sum(rate(http_request_duration_count[1m])) by (code, url, method)`) and present a request rate per second for the `/reservations` endpoint grouped by the response code (the method and URL should be intact).

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733939506052/f309e2c6-1412-4eb7-8367-d7b8fe6ca3e3.png align="center")

## Synthetic Monitor

In the previous section, we manually invoked `/reservations` endpoint several times. The idea of synthetic monitoring is to automate the process of probing an endpoint and storing the results as metrics.

Synthetic Monitor can be a different process or application that runs independently of other services. In our example, we implement the monitor as a Python service.

```python
# monotor/app.py
import os
import time
import threading

import prometheus_client
import structlog
import requests
from flask import Flask
from prometheus_client import Histogram, make_wsgi_app
from werkzeug.middleware.dispatcher import DispatcherMiddleware

logger = structlog.get_logger()

app = Flask(__name__)

# Add endpoint that will expose the metrics
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {"/metrics": make_wsgi_app()})

prometheus_client.REGISTRY.unregister(prometheus_client.GC_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PLATFORM_COLLECTOR)
prometheus_client.REGISTRY.unregister(prometheus_client.PROCESS_COLLECTOR)

INTERVAL_SEC = int(os.getenv("INTERVAL_SEC", 5))
RESERVATIONS_SERVICE_URL = os.getenv("RESERVATIONS_SERVICE_URL")
if not RESERVATIONS_SERVICE_URL:
    raise ValueError("RESERVATIONS_SERVICE_URL is not set")

HTTP_REQUEST_DURATION = Histogram(
    "synthetic_request_duration",
    "Synthetic requests durations",
    ["method", "url", "result"],
    buckets=[0.01, 0.1, 0.5, 2, float("inf")],
)

logger.info("Monitor")

def start_monitor():
    logger.info("Starting monitor")
    while True:
        _make_request()     
        time.sleep(INTERVAL_SEC)

    logger.info("Monitor terminated")

def _make_request():
    logger.info("Sending request to reservations service")
    endpoint = f"{RESERVATIONS_SERVICE_URL}/reservations"
    result = 'success'
    start = time.time()
    try:
        response = requests.post(endpoint, data='{"username": "synthetic"}')
        if response.status_code != 200:
            result = 'failure'
    except Exception as e:
        logger.warn(f"Error: {e}")
        result = 'failure'
    finally:
        end = time.time()
        HTTP_REQUEST_DURATION.labels(
            method='POST',
            url=endpoint,
            result=result
        ).observe(end - start)

# Start the monitor in a separate thread in the background
t = threading.Thread(target=start_monitor)
t.start()
```

The monitor is an infinite loop, that hits a desired endpoint (defined by `RESERVATION_SERVICE_URL` environment variable) at predefined intervals (`INTERVAL_SEC`). The endpoint response is analyzed and stored in the Prometheus histogram metric `synthetic_request_duration`. Instead of saving the response code, for each call we determine `result` value. The `success` is only if we get `200` response. For both client errors (`4..`) and internal server errors (`5..`) the `result` value is `failure`.

We used the `Flask` framework to set up an HTTP server for exposing Prometheus metrics from the monitor.

The monitor is run in a separate background thread (`threading.Thread(target=start_monitor)` to avoid blocking the HTTP server.

To build a service image, we defined a Dockerfile, similar to the one created for Reservation Service.

```python
# monitor/Dockerfile
FROM python:3.9-alpine
COPY app.py requirements.txt /
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["flask", "run", "-h", "0.0.0.0", "-p", "8000"]
```

The build command should be run within `monitor` folder:

```bash
$ docker build -t monitor:v0 ./
```

To start the monitor, we need to add it to `docker-compose.yml` file:

```yaml
# docker-compose.yml
version: "3.7"

services:
  ...

  reservations-service:
    image: reservations-service:v0
    ports:
      - target: 8000
        published: 8080

  monitor:
    image: monitor:v0
    ports:
      - target: 8000
        published: 8081
    environment:
      - RESERVATIONS_SERVICE_URL=http://reservations-service:8000
      - INTERVAL_SEC=5
```

The last step for the monitor is to add it as a target for the Prometheus instance.

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 10s
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "reservations-service"
    scrape_interval: 5s
    static_configs:
      - targets: ["reservations-service:8000"]
  - job_name: "monitor"
    scrape_interval: 5s
    static_configs:
      - targets: ["monitor:8000"]
```

As for Reservations Service, we use the same hostname we defined in the `docker-compose.yml` file for Synthetic Monitor: `monitor`.

After restarting Docker Compose, the monitor will periodically call Reservations Service, and we should be able to collect metrics from both services.

* Reservations Service
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733999433225/6b3f7971-1b9c-4fd0-8ceb-22c150ef4bef.png align="center")

* Synthetic Monitor
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1733999417713/3c7ede91-1f56-4e21-9cbd-60b599728d33.png align="center")

We can also examine the logs to verify whether the services are communicating with each other as expected.

```yaml
reservations-service-1  | 192.168.80.4 - - [12/Dec/2024 10:22:05] "POST /reservations HTTP/1.1" 200 -
monitor-1               | 2024-12-12 10:22:10 [info     ] Sending request to reservations service
reservations-service-1  | 2024-12-12 10:22:10 [info     ] Reservation request            data=b'{"username": "synthetic"}'
reservations-service-1  | 2024-12-12 10:22:10 [info     ] Reservation response           code=500
reservations-service-1  | 192.168.80.4 - - [12/Dec/2024 10:22:10] "POST /reservations HTTP/1.1" 500 -
monitor-1               | 192.168.80.3 - - [12/Dec/2024 10:22:12] "GET /metrics HTTP/1.1" 200 -
reservations-service-1  | 192.168.80.3 - - [12/Dec/2024 10:22:15] "GET /metrics HTTP/1.1" 200 -
monitor-1               | 2024-12-12 10:22:15 [info     ] Sending request to reservations service
reservations-service-1  | 2024-12-12 10:22:15 [info     ] Reservation request            data=b'{"username": "synthetic"}'
reservations-service-1  | 2024-12-12 10:22:15 [info     ] Reservation response           code=200
```

Beyond logs showing communication between services, we can also observe logs indicating that Prometheus is successfully reaching the `/metrics` endpoint: `"GET /metrics HTTP/1.1" 200`.

## Alerting

We collect metrics from both Reservations Service and Synthetic Monitor and while they are quite similar, they can be utilized in different ways with distinct alerting strategies.

For Reservations Service, we treat internal server errors (such as 500 errors) as failures, as they indicate issues within the service itself.

However, for Synthetic Monitoring, we treat even client errors (such as 400, 401, 403, and 404) as failures (not only 5xx errors). This is because we have full knowledge of the conditions and context of the request. For instance:

* **404 Errors**: We know that the user exists in the database, so a 404 error (resource not found) should never occur unless there is an issue with database retrieval.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734000971019/2637c4d9-8278-4fab-ae33-578857fafe84.png align="center")
    
* **401 and 403 Errors**: Since we pass the correct credentials, these errors should never happen unless there's an authentication issue.
    
* ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734000982629/afc36ecc-778a-4483-b21a-d2afbc08fa73.png align="center")
    
    **400 Errors**: As long as we send well-formed requests, a 400 error (bad request) should not occur. If it does, it suggests that there is a problem with request validation.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734000966571/2449b5a0-fa06-48c1-83f1-597b4d66a71e.png align="center")

Last but not least, in a situation where our `/reservations` endpoint ceased functioning (for example, because we were using a URL mapping that stopped working), synthetic monitoring can also detect the problem that traditional monitoring misses. In that scenario, synthetic monitoring will alert as it is unable to access a service, but internal monitoring won't since there won't be any traffic.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734006155878/dbf9531c-935c-4ead-8951-bd7796fcbb62.png align="center")

To set up different alerting strategies for Reservations Service and Monitor, we can define rules in `prometheus-rules.yml` file.

```yaml
groups:
  - name: reservations-service
    rules:
      - alert: Synthetic errors rate exceeds 10%
        for: 30s
        expr:  sum(rate(synthetic_request_duration_count{result=~"failure"}[1m]))/ sum(rate(synthetic_request_duration_count[1m])) * 100 > 10
        labels:
          severity: critical

      - alert: Errors rate exceeds 10%
        for: 30s
        expr:   sum(rate(http_request_duration_count{code=~"5.."}[1m])) / sum(rate(http_request_duration_count[1m])) * 100 > 10
        labels:
          severity: critical
```

The criteria for both alerts is the same: 10% of errors for at least 30 seconds should result in an alert. However, because they define error differently (for the Monitor, an error is any response code that is different than 200, but for Reservations Service, only requests that end in 5xx codes are considered errors) the alerts' sensitivity varies.

The rules will be enabled when we mount `prometheus-rules.yml` files into the correct location in the Prometheus instance

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
      - type: bind
        source: ./prometheus-rules.yml
        target: /etc/prometheus/rules.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
```

In the Prometheus config file `prometheus.yml` we also have to add a reference to the rules file. It is mounted as `/etc/prometheus/rules.yml` on the Prometheus instance and we only need to pass relative path name `rules.yml` in `rule_files` section.

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 10s
rule_files:
  - "rules.yml"
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "reservations-service"
    scrape_interval: 5s
    static_configs:
      - targets: ["reservations-service:8000"]
  - job_name: "monitor"
    scrape_interval: 5s
    static_configs:
      - targets: ["monitor:8000"]
```

Now, we can observe on `localhost:9090/alerts` page that we genuinely have situations when only the synthetic monitoring alert is firing.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1734002042468/b3b841cb-5460-4514-8d3c-d6009cb1c15c.png align="center")

## Conclusion

In this post, we demonstrated how to utilize synthetic monitoring as a complementary tool alongside traditional monitoring and testing to enhance system reliability. We walked through the implementation of a simple synthetic monitor and highlighted how this service can help identify issues in situations where other indicators might remain silent. By simulating user behavior and observing the system from an external perspective, synthetic monitoring provides proactive insights that traditional methods may miss, ensuring that potential problems are detected and addressed early.

The codebase covering the topic can be found [here](https://github.com/jorzel/synthetic-monitoring).