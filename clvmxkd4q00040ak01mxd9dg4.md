---
title: "How to improve the visibility of system internals"
seoTitle: "How to improve the visibility of system internals"
seoDescription: "How to increase the visibility of system internals using RED metrics, Grafana visualizations and dashboard as code approach"
datePublished: Tue Apr 30 2024 21:58:51 GMT+0000 (Coordinated Universal Time)
cuid: clvmxkd4q00040ak01mxd9dg4
slug: how-to-improve-the-visibility-of-system-internals
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714513164707/1eb134a0-cc7b-4310-a297-318bd73f70ee.png
tags: programming, python, devops, dashboard, grafana

---

## Overview

The aim of observability in a system is to gain deep insights into its internal state and behavior to facilitate troubleshooting, understanding, and improving its performance and reliability. In this post, I would like to focus on gathering and visualizing metrics that allow us to answer the following questions:

* How is the system performing?
    
* Are there any trends or patterns in the system behavior? Do we have any traffic/error anomalies?
    
* How are individual components of the system interacting with each other?
    

RED (Rate, Errors, Duration) method provides a comprehensive view of component (or whole service) performance, covering aspects such as throughput, reliability, and latency. By monitoring and analyzing these metrics, you can gain valuable insights into your components's behavior, diagnose issues, and make informed decisions to optimize performance and enhance user experience.

1. **Rate (Throughput)**: Rate refers to the traffic or request throughput that your component is handling.
    
2. **Errors (Reliability)**: Errors represent the number of failed requests or operations within a given time frame.
    
3. **Duration (Latency)**: Duration refers to the time it takes for your component to process individual requests or operations.
    

While RED metrics are commonly used to monitor HTTP traffic (I showed it in [one of the previous posts](https://medium.com/gitconnected/how-to-use-prometheus-for-web-application-monitoring-524204e36413)), they can also provide valuable insights into the performance of I/O operations (e.g. database, external service RPC calls, etc.).

It is a straightforward and universal way to answer fundamental questions about system components: how large is traffic that reaching a compoent, how many errors it returns, and how long do requests take?

In this post, I will show how to:

* setup Prometheus server to gather metrics,
    
* collect RED metrics for HTTP server requests and Redis database calls,
    
* setup Grafana to visualize Prometheus metrics,
    
* automate generating Grafana dashboards.
    

## Service Architecture

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714484383842/cfec5f2a-a8e9-4975-897a-65307065ab89.png align="center")

To illustrate the concept of boosting observability with RED metrics, we have established a web application built in Python (Flask) featuring a sole HTTP endpoint responsible for retrieving user reservations. To minimize relational database load, we've implemented request caching using Redis.

```python
import random
import time
import redis
import os
import json

from flask import Flask, request
from werkzeug import Response

app = Flask(__name__)

redis_client = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=os.environ["REDIS_PORT"],
    db=0,
)

@app.route("/reservations/<user_id>")
def reservations(user_id):
    if len(user_id) > 5:
        return Response("400", status=400)

    rand_problem = random.randint(1, 100)
    if rand_problem > 90:
        return Response("500", status=500)

    results = _get_from_cache(user_id)
    if not results:
        results = _fetch_reservations(user_id)
        _store_in_cache(user_id, results)

    return Response("200", status=200)


def _user_reservations_key(user_id):
    USER_RESERVATION_PREFIX = 'userReservations:'
    return f"{USER_RESERVATION_PREFIX}{user_id}"


def _store_in_cache(user_id, reservations):
    value = json.dumps(reservations)
    redis_client.set(_user_reservations_key(user_id), value)


def _get_from_cache(user_id):
    raw_value = redis_client.get(_user_reservations_key(user_id))
    if not raw_value:
        return None
    return json.loads(raw_value)


def _fetch_reservations(user_id):
    random_duration = (
        random.choice([1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3])
        * random.randint(1, 100)
        * 0.001
    )
    time.sleep(random_duration)
    return [{"id": random.randint(0, 12200), "user_id": user_id}]

@app.route("/")
@app.route("/up")
def up():
    return "I am running"
```

The service architecture is straightforward: it processes HTTP requests. Initially, it checks if user reservations are cached. If they are, it retrieves and returns them. Otherwise, it simulates a relational database query in our code, utilizing sleep and generating random results (without an actual database connection), before returning the fetched data.

To build the application we define a Dockerfile:

```python
FROM python:alpine
COPY app.py requirements.txt /
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["flask", "run", "-h", "0.0.0.0", "-p", "8000"]
```

To run it along with the Redis database, we start it using the Docker Compose file:

```yaml
# docker-compose.yaml
version: "3.7"

services:
  redis:
    image: redis:latest
    command: redis-server --appendonly yes

  myapp:
    image: myapp:v0
    ports:
      - target: 8000
        published: 8080
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
```

## Prometheus setup

The next step is to configure the Prometheus server to gather metrics. I described this process in detail [in the web application monitoring post](https://medium.com/gitconnected/how-to-use-prometheus-for-web-application-monitoring-524204e36413), so I will not elaborate here.

We need a `prometheus.yaml` config file:

```yaml
# prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 10s
scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "myapp"
    scrape_interval: 5s
    static_configs:
      - targets: ["myapp:8000"]
```

And put it in the Docker Compose file:

```yaml
# docker-compose.yml
version: "3.7"

services:
  ...
  prometheus:
    image: prom/prometheus
    ports:
      - target: 9090
        published: 9090
    volumes:
      - type: bind
        source: ./prometheus/prometheus.yml
        target: /etc/prometheus/prometheus.yml
      - ./prometheus/data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
```

## HTTP requests metrics

To collect any metrics, we need to have `prometheus_client` library installed. The incoming HTTP traffic can be measured by applying a simple decorator to HTTP endpoints. The same approach was implemented in the aforementioned post about web monitoring using Prometheus.

```python
import time
from prometheus_client import Histogram

HTTP_REQUEST_DURATION = Histogram(
    "http_request_duration",
    "Http requests durations",
    ["method", "path", "code"],
    buckets=[0.01, 0.1, 0.5, 2, float("inf")],
)

def _map_url(incoming_path):
    RESERVATIONS_REGEX = '^\/reservations\/(?P<user_id>.*)'
    _path_mapper = {
        RESERVATIONS_REGEX: "/reservations/:user_id"
    }
    for pattern, outgoing_path in _path_mapper.items():
        if re.match(pattern, incoming_path):
            return outgoing_path
    return incoming_path

def observe_http(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        try:
            response = func(*args, **kwargs)
            response_code = response.status_code
            return response
        except Exception:
            response_code = 500
            raise
        finally:
            end = time.time()
            parts = urlparse(request.url)
            HTTP_REQUEST_DURATION.labels(
                method=request.method,
                code=response_code,
                path=_map_url(parts.path),
            ).observe(end - start)
        return response
    return wrapper
```

Using `observe_http` decorator we collect `http_request_duration` histogram with labels:

* `method` - a method of the HTTP request (e.g. `GET`, `POST`, etc.)
    
* `path` - the relative path of the HTTP request handler (e.g. `/reservations/:user_id`)
    
* `code` - response status for HTTP request (e.g. 200, 400, etc.)
    

The reservation endpoint has a dynamic URL path because there is a `user_id` parameter. However, we want to store each request reaching this endpoint under the same `path` value: `/reservations/:user_id` instead of creating a new value for each `user_id` value, e.g. `/reservations/1234`, `/reservations/23`, etc. The `_map_url` helper function is responsible for the transformation: from dynamic URL dependent on `user_id` to stable `/reservations/:user_id` path.

Adding a decorator to the HTTP endpoint will ensure that incoming request metrics will be registered:

```python
@app.route("/reservations/<user_id>")
@observe_http
def reservations(user_id):
   ...
```

In the next step, we provide a way to gather metrics from Redis operations.

## Measure Redis metrics

We have already had Redis Client initialized for our service.

```python
import redis

redis_client = redis.Redis(
    host=os.environ["REDIS_HOST"],
    port=os.environ["REDIS_PORT"],
    db=0,
)
```

However, this client does not provide metrics collection. The most elegant way to start tracking Redis calls is the introduction of a client wrapper:

```python
from prometheus_client import Histogram

REDIS_CALL_DURATION = Histogram(
    "redis_call_duration",
    "Redis calls durations",
    ["operation", "status"]
)

def observe_redis(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        status = 'success'
        try:
            result = func(*args, **kwargs)
            return result
        except Exception:
            status = 'error'
            raise
        finally:
            end = time.time()
            REDIS_CALL_DURATION.labels(
                status=status,
                operation=func.__name__
            ).observe(end - start)
    return wrapper

class InstrumentedRedisClient:
    def __init__(self, client):
        self._client = client

    @observe_redis
    def set(self, *args, **kwargs):
        return self._client.set(*args, **kwargs)

    @observe_redis
    def get(self, *args, **kwargs):
        return self._client.get(*args, **kwargs)
```

Our new `InstumentedRedisClient` implements Redis methods that we use and decorate them with `observe_redis`. It is a function that registers measurements for the Prometheus histogram `redis_call_duration`. It has two labels:

* `operation` which allows us to differentiate different methods (e.g. `set`, `get`, etc.)
    
* `status` that determines whether the operation was a `success` or an `error`.
    

Our new client can be initialized in the following way:

```python
redis_client = InstrumentedRedisClient(
    client=redis.Redis(
         host=os.environ["REDIS_HOST"],
         port=os.environ["REDIS_PORT"],
         db=0,
    )
)
```

The code that uses `redis_client` can stay intact, because `InstrumentedRedisClient` provide the same interface as `RedisClient` for methods we employ.

In the next sections, we show how to visualize metrics in Grafana.

## Grafana setup

Setting up Grafana to provide Prometheus metrics is relatively simple. We just need to create a `datasources.yml` file.

```yaml
# grafana/datasources.yml
datasources:
-  access: 'proxy'
   editable: true
   is_default: true
   name: 'prometheus'
   org_id: 1
   type: 'prometheus'
   url: 'http://prometheus:9090'
   version: 1
```

The `url` value should conform to the configuration we provided for Prometheus in the Docker Compose file (`http://<host>:<port>`).

The `datasources.yml` file should be mounted to Grafana `volumes` in the Docker Compose file:

```yaml
# docker-compose.yaml
version: "3.7"

services:
  ...
  grafana:
    volumes:
      - ./grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yaml
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    image: grafana/grafana:latest
    ports:
      - target: 3000
        published: 3000
```

To check whether Grafana is running we can start `docker-compose up` and visit `http://localhost:3000` (`3000` is the port we published). We should see the Grafana front page without any dashboards:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714510816940/67ac1fa1-4f1e-4af1-833e-cd4c41bd8f1d.png align="center")

While Grafana's user interface enables users to configure and create dashboards, this method presents several challenges. It struggles with scalability and complicates dashboard maintenance, posing significant issues. We turn to a more robust approach: "dashboard as code".

## Dashboards as code

The "dashboard as code" approach involves defining and managing Grafana dashboards using code, typically in a high-level, human-readable format such as YAML, JSON, or a domain-specific language. Instead of manually creating or editing JSON files to define dashboards, you use tools or libraries that allow you to define dashboards programmatically, often using familiar programming languages or configuration formats.

There are several reasons why the dashboard as code approach can be tempting and advantageous:

* **Version Control**: By defining dashboards as code, you can version control them alongside your application code using Git or other version control systems.
    
* **Reproducibility**: Storing dashboards as code makes it easy to reproduce and deploy them in different environments, such as development, staging, and production.
    
* **Automation and Templating**: Dashboards as code allow you to automate the creation and management of dashboards using scripts, templates, or configuration files.
    
* **Abstraction and Reusability**: Tools or libraries for defining dashboards as code often provide higher-level abstractions and reusable components that simplify dashboard creation and maintenance.
    

## Dashboard generation

There are several options for managing Grafana as code. The most popular option is [Grafonnet](https://github.com/grafana/grafonnet), a [jsonnet](https://jsonnet.org/) library providing a high-level abstraction over the Grafana API. However, for the purpose of this article, I would like to show [Grafanalib](https://github.com/weaveworks/grafanalib), a Python library to create dynamic and reusable dashboards.

We need to write a Python script `dashboard.py` that execution results in Grafana dashboard JSON file generation.

```python
# grafana/dashboard.py

import json
import os

from grafanalib.prometheus import PromGraph
from grafanalib.core import (
    Dashboard, Row, Graph, Target, Histogram, OPS_FORMAT, single_y_axis, YAxes,
    YAxis, MILLISECONDS_FORMAT, SHORT_FORMAT,
)
from grafanalib._gen import DashboardEncoder

def duration_panel(
    metric, title='Response times', y_axes_format=MILLISECONDS_FORMAT
):
    return PromGraph(
        title=title,
        data_source='prometheus',
        expressions=[
            ('p99', f'histogram_quantile(0.99, sum(rate({metric}_bucket[1m])) by (le))'),
            ('p90', f'histogram_quantile(0.90, sum(rate({metric}_bucket[1m])) by (le))'),
            ('p50', f'histogram_quantile(0.5, sum(rate({metric}_bucket[1m])) by (le))'),
        ],
        yAxes=single_y_axis(format=y_axes_format),
    )


def http_requests_panel(
    metric, title="Requests", y_axes_format=OPS_FORMAT, legend_format="{{path}} {{method}} {{code}}"
):
    return Graph(
        title=title,
        dataSource='prometheus',
        targets=[
            Target(
                expr=f'rate({metric}_count[1m])',
                legendFormat=legend_format,
                refId='A',
            ),
        ],
        yAxes=single_y_axis(format=y_axes_format),
    )

def redis_requests_panel(
    metric, title="Requests", y_axes_format=OPS_FORMAT, legend_format="{{operation}} {{status}}"
):
    return Graph(
        title=title,
        dataSource='prometheus',
        targets=[
            Target(
                expr=f'rate({metric}_count[1m])',
                legendFormat=legend_format,
                refId='A',
            ),
        ],
        yAxes=single_y_axis(format=y_axes_format),
    )

http_requests_duration = duration_panel("http_request_duration", "HTTP Requests Response Times") 
http_requests = http_requests_panel("http_request_duration", "HTTP Requests")

redis_calls_duration = duration_panel("redis_call_duration", "Redis Call Response Times")
redis_calls = redis_requests_panel("redis_call_duration", "Redis Calls")

http_row = Row(title="RED - HTTP", panels=[http_requests, http_requests_duration])
redis_row = Row(title="RED - Redis", panels=[redis_calls, redis_calls_duration])

# Create a dashboard containing the RED panel
dashboard = Dashboard(
    title='Service instrumenation',
    rows=[http_row, redis_row],
    refresh='10s',
).auto_panel_ids()

# Convert the dashboard object to JSON
def get_dashboard_json(dashboard, overwrite=False, message="Updated by grafanlib"):
    '''
    get_dashboard_json generates JSON from grafanalib Dashboard object

    :param dashboard - Dashboard() created via grafanalib
    '''

    # grafanalib generates json which need to pack to "dashboard" root element
    return json.dumps(
        dashboard.to_json_data(),
        sort_keys=True,
        indent=2,
        cls=DashboardEncoder,
    )

def save_dashboard(dashboard):
    current_dir = os.path.dirname(__file__)
    output_path = os.path.join(current_dir, "dashboards/dashboard.json")
    dashboard_json = get_dashboard_json(dashboard)
    with open(output_path, "w") as outfile:
        outfile.write(dashboard_json)

save_dashboard(dashboard)
```

We define two RED metrics rows: HTTP metrics (`http_row`) and Redis metrics (`redis_row`). We use high-level `grafanalib` building blocks `Graph` , `Row` and `Dashboard` to implement it. Each Grafana row consists of a requests panel (Rate + Errors) and a requests latency panel (Duration). The `save_dashboard` function transforms `Dashboard` object into JSON file and saves it under `grafana/dashbaords/dashboard.json` location. To generate a dashboard JSON file we just need to execute a script:

```python
$ cd grafana
$ python dashboard.py
```

The last step is mounting the generated `grafana/dashbaords/dashboard.json` file to the Grafana `volumes`. We need an additional file `dashboards.yml`:

```yaml
# grafana/dashboards/dashboards.yml

- name: 'default'
  org_id: 1
  folder: ''
  type: 'file'
  options:
    folder: '/var/lib/grafana/dashboards'
```

Now we can modify the Docker Compose file and mount `./grafana/dashboards` local directory to `/var/lib/grafana/dashboards` Grafana location on Docker Compose environment:

```yaml
 # docker-compose.yaml
version: "3.7"

services:
  ...  
  grafana:
    volumes:
      - ./grafana/dashboards:/var/lib/grafana/dashboards
      - ./grafana/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yaml
    environment:
      - GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH=/var/lib/grafana/dashboards/dashboard.json
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    image: grafana/grafana:latest
    ports:
      - target: 3000
        published: 3000
```

We also added the env variable: `GF_DASHBOARDS_DEFAULT_HOME_DASHBOARD_PATH`. Thanks to it we will be redirected to our dashboard when we log into Grafana.

We can restart the Docker Compose environment and check `http://localhost:3000` again.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1714512060479/97179d31-9951-4e09-9c34-52956163bb0e.png align="center")

We should see two Grafana Rows without data. Now we can simulate some traffic in our application to see how our metrics are visualized.

## Metrics visualization

To get some data, we can hit `http://localhost:3000/<user_id/reservations` endpoint several times.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713437896625/be929f9b-f474-4321-8d56-b1ce2cdec818.png align="center")

We should see a nice visualization of the HTTP request rate (the number of requests per second) grouped by `path`, `method` and `status`. On the right-hand side, we have requests latency for 99, 90, and 50 percentiles. Similar data are visible for Redis calls (grouped by `operation` and `status`).

## Conclusion

RED metrics, coupled with Grafana visualizations, serve as powerful tools that enhance our system's visibility. They can be seamlessly applied to monitor HTTP traffic, internal service operations, and interactions with external systems. During incidents, we can quickly assess whether a specific component is functioning as expected or if anomalies such as spikes in errors or prolonged latency are occurring. Furthermore, generating dashboards through code adds to the robustness and maintainability of this approach, enabling easier management and updates.

The codebase covering this post can be found [here](https://github.com/jorzel/red-instrumentation).