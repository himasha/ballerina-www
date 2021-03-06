# How to Observe Ballerina Programs

## Introduction
Observability is a measure of how well internal states of a system can be inferred from knowledge of its external
outputs. Monitoring, logging, and distributed tracing are key methods that reveal the internal state of the system to
provide the observability. Therefore, Ballerina becomes fully observable by having default support for connecting with
external systems to monitor metrics such as request count and response time statistics, log processing and analysis,
and perform distributed tracing.

HTTP/HTTPS based Ballerina services and any client connectors are observable by default. HTTP/HTTPS and SQL client
connectors provide [semantic tags](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)
as well to make tracing and metrics monitoring more informative.

## Getting Started
A Ballerina service is by default observable. This section focuses on enabling observability with default systems such
as [Prometheus] and [Grafana] for metrics monitoring and [Jaeger] for distributed tracing. Ballerina logs can
be read and fed to any external log monitoring system like Elastic Stack to perform log monitoring and analysis.

**Pre-requisites**

Make sure you have already installed [Docker](https://www.docker.com/) to setup external products such as Jaeger,
Prometheus, etc. You can follow [official documentation](https://docs.docker.com/install/) to install Docker.

**Steps**

**Step 1:** Create Hello World Ballerina service as shown below and save it as `hello_world_service.bal`.

```ballerina
import ballerina/http;
import ballerina/log;

service<http:Service> hello bind { port:9090 } {
    
    sayHello (endpoint caller, http:Request req) {
        log:printInfo("This is a test Info log");
        log:printError("This is a test Error log");
        log:printWarn("This is a test Warn log");
        http:Response res = new;
        res.setStringPayload("Hello, World!");
        _ = caller -> respond(res);
    }
}
```

**Step 2:** The observability is disabled by default and it can be enabled either by adding `--observe` flag or updating
the configuration. This option enables metrics monitoring and distributed tracing. You will have to follow
step 3 and step 4, if you want to monitor logs as well.

Using `--observe` flag:

Observability for Ballerina service can be enabled with a flag `--observe` as shown below. This enables observability
for Ballerina service with default settings, and collect the distributed tracing information with [Jaeger]
and metrics with [Prometheus].

```bash
$ ballerina run --observe hello_world_service.bal

ballerina: started Prometheus HTTP endpoint localhost/127.0.0.1:9797
ballerina: started publishing tracers to Jaeger on localhost:5775
ballerina: initiating service(s) in 'hello_world_service.bal'
ballerina: started HTTP/WS endpoint 0.0.0.0:9090
```

Configuration file:

Observability can be enabled from the configuration. An example configuration that starts metrics monitoring and
distributed tracing with default configurations is given below.

```
[b7a.observability.metrics]
# Flag to enable Metrics
enabled=true

[b7a.observability.tracing]
# Flag to enable Tracing
enabled=true
```

The Ballerina program needs to be started as below with either `--config` or `--c` option and provide the path of the
configuration file to adhere to the configuration as shown below.

```bash
$ ballerina run --config <path-to-conf>/ballerina.conf hello_world_service.bal

ballerina: started Prometheus HTTP endpoint localhost/127.0.0.1:9797
ballerina: started publishing tracers to Jaeger on localhost:5775
ballerina: initiating service(s) in 'hello_world_service.bal'
ballerina: started HTTP/WS endpoint 0.0.0.0:9090
```

**Step 3:** Follow this step if you want to monitor Ballerina logs. In the Ballerina service, there are some
logs added to the service. The log package supports logging to the console only, therefore the logs need
to be redirected to a file if you want to perform log analysis. Those logs need to be pushed to
[Elastic Stack](#Distributed-Logging) to perform the log analysis.

Start the service as below along with the option that you have opted to start service in step 2 (`--observe` flag or
using configuration file).
   
Start Ballerina service with `--observe` flag:

```bash
$ nohup ballerina run --observe hello_world_service.bal > ballerina.log &
```

Start Ballerina service with configuration file:

```bash
$ nohup ballerina run --config <path-to-conf>/ballerina.conf hello_world_service.bal > ballerina.log &
```

**Step 4:** Install Elastic Stack as mentioned in section [Setting up Elastic Stack](#setting-up-elastic-stack). You may
skip this step, if you do not want to monitor logs.

**Step 5:** Install and configure Prometheus as mentioned in section [Setting up Prometheus](#Prometheus).

**Step 6:** Install and configure Grafana as mentioned in section [Setting up Grafana](#Grafana).

**Step 7:** Install Jaeger as mentioned in section [Setting up Jaeger](#jaeger-server).

**Step 8:** Send few requests to `http://localhost:9090/hello/sayHello`

Example cURL command:

```bash
$ curl http://localhost:9090/hello/sayHello
```

**Step 9:** Go to the imported dashboard in Grafana and check the metrics. Similarly, do this for Jaeger by going to
`http://localhost:16686/` and check the traces.

![Jaeger Sample Dashboard](images/jaeger-sample-dashboard.png "Jaeger Sample Dashboard")

**Step 10:** If you have followed step 4, then you can go to Elastic Stack dashboard and view logs.

![Kibana Sample Dashboard](images/kibana-sample-dashboard.png "Kibana Sample Dashboard")

## Metrics Monitoring
Metrics help to monitor the runtime behaviour of a service. Therefore, metrics is a vital part of monitoring
Ballerina programs. However, metrics is not the same as analytics. For example, you should not use metrics to do
something like per-request billing. Metrics are used to measure what Ballerina program does at runtime to make
better decisions using the numbers. The code generates business value when it is run in production.
Therefore, it is imperative to continuously measure the code in production.

Metrics, by default, supports Prometheus. In order to support Prometheus, an HTTP endpoint starts with the context
of `/metrics` in default port 9797 when starting the Ballerina program.

### Configure Ballerina
This section focuses on the Ballerina configurations that are available for metrics monitoring with Prometheus,
and the sample configuration is provided below.

```
[b7a.observability.metrics]
enabled=true
provider="micrometer"

[b7a.observability.metrics.micrometer]
registry.name="prometheus"

[b7a.observability.metrics.prometheus]
port=9797
hostname="0.0.0.0"
descriptions=false
step="PT1M"
```

The descriptions of each configuration above are provided below with possible alternate options.

Configuration Key | Description | Default Value | Possible Values 
--- | --- | --- | --- 
b7a.observability.metrics.enabled | Whether metrics monitoring is enabled (true) or disabled (false) | false | true or false
b7a.observability.metrics.provider | Provider name which implements Metrics interface. This is only required to be modified if a custom provider is implemented and needs to be used. | micrometer | micrometer or if any custom implementation, then name of the provider.
b7a.observability.metrics.micrometer.registry.name | Name of the registry used in micrometer | prometheus | prometheus 
b7a.observability.metrics.prometheus.port | The value of the port in which the service '/metrics' will be bind to. This service will be used by Prometheus to scrape the information of the Ballerina program. | 9797 | Any suitable value for port 0 - 0 - 65535. However, within that range, ports 0 - 1023 are generally reserved for specific purposes, therefore it is advisable to select a port without that range. 
b7a.observability.metrics.prometheus.hostname | The hostname in which the service '/metrics' will be bind to. This service will be used by Prometheus to scrape the information of the Ballerina program. | 0.0.0.0 | IP or Hostname or 0.0.0.0 of the node in which the Ballerina program is running.
b7a.observability.metrics.prometheus.descriptions | This flag indicates whether meter descriptions should be sent to Prometheus. Turn this off to minimize the amount of data sent on each scrape. | false | true or false
b7a.observability.metrics.prometheus.step | The step size to use in computing windowed statistics like max. To get the most out of these statistics, align the step interval to be close to your scrape interval. | PT1M (1 minute) | The formats accepted are based on the ISO-8601 duration format PnDTnHnMn.nS with days considered to be exactly 24 hours.

### Setting up External Systems
There are mainly two systems involved in collecting and visualizing the metrics. [Prometheus] is used to collect the
metrics from the Ballerina program and [Grafana] can connect to Prometheus and visualize the metrics in the dashboard.

#### Prometheus
[Prometheus] is used as the monitoring system, which pulls out the metrics collected from the Ballerina service
'/metrics'. There are many ways to install the Prometheus and you can find the possible options from
[installation guide](https://prometheus.io/docs/prometheus/latest/installation/).

This section focuses on the quick installation of Prometheus with Docker, and configure it to collect metrics from
Ballerina program with default configurations. Below provided steps needs to be followed to configure the Prometheus.

**Step 1:** Create a `prometheus.yml` file in `/tmp/` directory.

**Step 2:** Use following content for `/tmp/prometheus.yml`.

Go to [official documentation of Prometheus](https://prometheus.io/docs/introduction/first_steps/), if you need more
information. Please note that the targets should contain the host and port of the `/metrics` service
that's exposed from Ballerina program for metrics collection. Let's say if the IP of the host in which the Ballerina
program is running is a.b.c.d and the port is default 9797 (configured from `b7a.observability.metrics.prometheus.port`
configuration in Ballerina configuration file), then the sample configuration in Prometheus will be as below.

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['a.b.c.d:9797']
```

**Step 3:** Start the Prometheus server in a Docker container with below command.

```bash
$ docker run -p 19090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```
    
**Step 4:** Go to `http://localhost:19090/` and check Prometheus graph to see whether Ballerina metrics are available.

#### Grafana
Let’s use [Grafana] to visualize metrics in a dashboard. For this, we need to install Grafana, and configure
Prometheus as a datasource. Follow the below provided steps and configure Grafana.

**Step 1:** Start Grafana as docker container with below command. For more information, please go to [link](https://hub.docker.com/r/grafana/grafana/).

```bash
$ docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

**Step 2:** Go to http://localhost:3000/ from where the docker is running, and access the Grafana dashboard.

**Step 3:** Login to the dashboard with default user, username: `admin` and password: `admin`

**Step 4:** Add Prometheus as datasource with direct access configuration as provided below.

![Grafana Prometheus Datasource](images/grafana-prometheus-datasource.png "Grafana Prometheus Datasource")

**Step 5:** Now you can import the default Grafana dashboard which has some default graphs to visualize the request/response metrics.

## Distributed Tracing

Tracing provides information regarding the roundtrip of a service invocation based on the concept of spans, which are
structured in a hierarchy based on the cause and effect concept. Tracers propagate across several services that can be
deployed in several nodes, depicting a high-level view of interconnections among services as well, hence coining the
term distributed tracing.

A span is a logical unit of work, which encapsulates a start and end time as well as metadata to give more meaning to
the unit of work being completed. For example, a span representing a client call to an HTTP endpoint would give the
user the latency of the client call and metadata like the HTTP URL being called and HTTP method used. If the span
represents an SQL client call, the metadata would include the query being executed.

Tracing gives the user a high-level view of how a single service invocation is processed across several distributed
microservices.

* Identify service bottlenecks - The user can monitor the latencies and identify when a service invocation slows down,
pinpoint where the slowing down happens (by looking at the span latencies) and take action to improve the latency.
* Error identification - If an error occurs during the service invocation, it will show up in the list of tracers.
The user can easily identify where the error occurred and information of the error will be attached to the relevant
span as metadata.

Ballerina supports [OpenTracing](http://opentracing.io/) standards out of the box. This means that Ballerina services
can be traced using OpenTracing implementations like [Jaeger](http://www.jaegertracing.io/), and
[Zipkin](https://zipkin.io/). Jaeger is the default tracer of Ballerina.

### Configure Ballerina
Tracing can be enabled in Ballerina with `--observe` flag as mentioned in the [Getting Started](#getting-started) section, as well as configuration option. This section mainly focuses on the configuration options with description and possible values.

The sample configuration that enables tracing, and uses Jaeger as the sample tracer as provided below.

```yaml
[b7a.observability.tracing]
enabled=true
name="jaeger"
```

The below table provides the descriptions of each configuration option and possible values that can be assigned.

Configuration Key | Description | Default Value | Possible Values
--- | --- | --- | --- 
b7a.observability.tracing.enabled | Whether tracing is enabled (true) or disabled (false) | false | true or false
b7a.observability.tracing.name | Tracer name which implements tracer interface. | jaeger | jaeger or zipkin

#### Jaeger Client
Jaeger is the default tracer supported by Ballerina. Below is the sample configuration options that are available in
the Jaeger.

```yaml
[b7a.observability.tracing]
enabled=true
name="jaeger"

[b7a.observability.tracing.jaeger]
reporter.hostname="localhost"
reporter.port=5775
sampler.type="const"
sampler.param=1.0
reporter.flush.interval.ms=2000
reporter.max.buffer.spans=1000
```

The below table provides the descriptions of each configuration option and possible values that can be assigned.

Configuration Key | Description | Default Value | Possible Values 
--- | --- | --- | --- 
b7a.observability.tracing.jaeger.reporter.hostname | Hostname of the Jaeger server | localhost | IP or hostname of the Jaeger server. If it is running on the same node as the Ballerina, it can be localhost. 
b7a.observability.tracing.jaeger.reporter.port | Port of the Jaeger server | 5775 | The port which the Jaeger server is listening to.
b7a.observability.tracing.jaeger.sampler.type | Type of the sampling methods used in the Jaeger tracer. | const | const, probabilistic, or ratelimiting.
b7a.observability.tracing.jaeger.sampler.param | It is a floating value. Based on the sampler type, the effect of the sampler param varies. Const - 0 is no sampling and 1 is sample all spans, Probabilistic - must be between 1.0 and 0.0, and Ratelimiting - param specifies rate per second | 1.0 | For Const 0 or 1, for Probabilistic 0.0 to 1.0, for Ratelimiting any positive integer
b7a.observability.tracing.jaeger.reporter.flush.interval.ms | Jaeger client will be sending the spans to the server at this interval. | 2000 | Any positive integer value.
b7a.observability.tracing.jaeger.reporter.max.buffer.spans | Queue size of the Jaeger client. | 2000 | Any positive integer value.

#### Zipkin Client
The tracing of Ballerina program can be done via Zipkin as well, but the required dependencies are not included in
default Ballerina distribution. Follow the below steps to add the required dependencies to the Ballerina distribution.

**Step 1:** Go to [ballerina-observability](https://github.com/ballerina-platform/ballerina-observability) and clone
the GitHub repository in any preferred location.

**Step 2:** Make sure you have installed [Apache Maven](http://maven.apache.org/).

**Step 3:** Open the command line and build the repository by using [Apache Maven](http://maven.apache.org/) with below command, while being in the root project directory `ballerina-observability`.

```bash
$ mvn clean install
```

**Step 4:** Go to the path - `ballerina-observability/tracing-extensions/modules/ballerina-zipkin-extension/target/` and
extract `distribution.zip`.

**Step 5:** Copy all the JAR files inside the `distribution.zip` to 'bre/lib' directory in the Ballerina distribution.

**Step 6:** Change the following configuration name to Zipkin. This ensures that all tracers are sent to Zipkin instead
of the default Jaeger tracer.

```yaml
[b7a.observability.tracing]
name="zipkin"
```

**Step 7:** The following configuration is a sample configuration option available for Zipkin tracer.

```
[b7a.observability.tracing.zipkin]
reporter.hostname="localhost"
reporter.port=9411
```

The below table provides the descriptions of each configuration option and possible values that can be assigned. 

Configuration Key | Description | Default Value | Possible Values 
--- | --- | --- | --- 
b7a.observability.tracing.zipkin.reporter.hostname | Hostname of the Zipkin server | localhost | IP or hostname of the Zipkin server. If it is running on the same node as the Ballerina, it can be localhost. 
b7a.observability.tracing.zipkin.reporter.port | Port of the Zipkin server | 9411 | The port which the Zipkin server is listening to.

### Setting up External Systems
Ballerina by default supports Jaerger and Zipkin for distributed tracing. This section focuses on configuring the
Jaeger and Zipkin with dockers as a quick installation.

#### Jaeger Server
Jaeger is the default distributed tracing system that is supported. There are many possible ways to deploy Jaeger and you can find more information on this [link](https://www.jaegertracing.io/docs/deployment/). Here we focus on all in one deployment with Docker.

**Step 1:** Install Jaeger via docker and start the docker container by executing below command.

```bash
$ docker run -d -p5775:5775/udp -p6831:6831/udp -p6832:6832/udp -p5778:5778 -p16686:16686 -p14268:14268 jaegertracing/all-in-one:latest
```

**Step 2:** Go to `http://localhost:16686` and load the web UI of the Jaeger to make sure it is functioning properly.

The below image is the sample tracing information you can see from Jaeger.

![Jaeger Tracing Dashboard](images/jaeger-tracing-dashboard.png "Jaeger Tracing Dashboard")

#### Zipkin Server
Similar to Jaeger, Zipkin is another distributed tracing system that is supported by the Ballerina. There are many
different configurations and deployment exist for Zipkin, please go to [link](https://github.com/openzipkin/zipkin)
for more information. Here we focus on all in one deployment with Docker.

**Step 1:** Install Zipkin via docker and start the Docker container by executing following command.

```bash
$ docker run -d -p 9411:9411 openzipkin/zipkin
```

**Step 2:** Go to `http://localhost:9411/zipkin/` and load the web UI of the Zipkin to make sure it is functioning
properly. The below shown is the sample Zipkin dashboard for the hello world sample in the [Quick Start](#Quick-start)

![Zipkin Sample](images/zipkin-sample.png "Zipkin Sample")

## Distributed Logging
Ballerina distributed logging and analysis is supported by Elastic Stack. Ballerina has a log package for logging to the console. In order to monitor the logs, the Ballerina standard output need to be redirected to a file.

This can be done by running the Ballerina service as below.

```bash
$ nohup ballerina run ballerinaservice.bal > ballerina.log &
```

You can view the logs with below command.

```bash
$ tail -f ~/wso2-ballerina/workspace/ballerina.log
```


### Setting up Elastic Stack
The elastic stack comprises of the following components.

1. Beats - Multiple agents that ship data to Logstash or Elasticsearch. In our context, Filebeat will ship the Ballerina logs to Logstash. Filebeat should be a container running on the same host as the Ballerina service. This is so that the log file (ballerina.log) can be mounted to the Filebeat container.
2. Logstash - Used to process and structure the log files received from Filebeat and send Elasticsearch.
3. Elasticsearch - Storage and indexing of the logs received by Logstash.
4. Kibana - Visualizes the data stored in Elasticsearch

Elasticsearch and Kibana are provided as cloud services from https://www.elastic.co/cloud with a trial period.
We only have to set up Logstash and Filebeat containers on-premise if you are going ahead with cloud services.
If you are not opting for the cloud service, you can use Docker containers for all the tools.

**Step 1:** Download the docker images using the following commands.

```bash
# Elasticsearch Image
$ docker pull docker.elastic.co/elasticsearch/elasticsearch:6.2.2
# Kibana Image
$ docker pull docker.elastic.co/kibana/kibana:6.2.2
# Filebeat Image
$ docker pull docker.elastic.co/beats/filebeat:6.2.2
# Logstash Image
$ docker pull docker.elastic.co/logstash/logstash:6.2.2
```

**Step 2:** Start elastic search and Kibana containers with below commands.

```bash
$ docker run -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name elasticsearch docker.elastic.co/elasticsearch/elasticsearch:6.2.2
$ docker run -p 5601:5601 -h kibana --name kibana --link elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:6.2.2
```

**Note:** Linux users may have to increase the `vm.max_map_count` for the Elasticsearch container to start. Execute the following command to do that.

```bash
$ sudo sysctl -w vm.max_map_count=262144
```

In `above docker` run commands,
* `-h` flag sets the containers hostname.
* `--link` flag is used to connect Docker container to another container. The container can consume services using the
hostname specified.

The Kibana container links to the Elasticsearch container and can use the `elasticsearch` hostname to talk to that
container.

**Step 3:** Next step is, configuring Logstash to format the Ballerina logs. In order to do this, you need to create a file named `logstash.conf` with the following content.

For this example, let's save this file at `/tmp/pipeline/logstash.conf`. This should be taken note of because, when starting the Logstash container, this file should be bind-mounted onto the container. Logstash container is configured to read a configuration file named `logstash.conf` at startup.

```
input {
  beats {
    port => 5044
    }
}
filter {
  grok  {
    match => { "message" => "%{TIMESTAMP_ISO8601:date}%{SPACE}%{WORD:logLevel}%{SPACE}\[%{GREEDYDATA:package}\]%{SPACE}\-%{SPACE}%{GREEDYDATA:logMessage}"}
  }
}
output {
    elasticsearch {
        hosts => "elasticsearch:9200"
        index => "store"
      document_type => "store_logs"
    }
}
```

**Note:**
* 3 stages are specified in the pipeline .
* Input is specified as beats and listens to port 5044.
* A grok filter is used to structure the Ballerina logs.
* An output is specified to push to Elasticsearch. Note here that the host is specified as `elasticsearch:9200`.
Hence the Elasticsearch container should be linked to the Logstash container.

* Make sure the `logstash.conf` is being stored in the empty directory and not having other unwanted documents within it, because during the startup Logstash merges the configurations and make it as a single file, and this operation will fail if there are unrelated configuration files in it.

**Step 4:** Start the Logstash container by the following command.

```bash
$ docker run -h logstash --name logstash --link elasticsearch:elasticsearch -it --rm -v /tmp/pipeline:/usr/share/logstash/pipeline/ -p 5044:5044 docker.elastic.co/logstash/logstash:6.2.2
```

**Step 5:** Configure Filebeat to ship the Ballerina logs. For this, you need to create a file named filebeat.yml with the following content. As an example, let's save this file at `/tmp/filebeat.yml`.

```
filebeat.prospectors:
- type: log
  paths:
    - /usr/share/filebeat/ballerina.log
output.logstash:
  hosts: ["logstash:5044"]
```

**Note:**
* Here also the host is specified as `logstash:5044`. Hence the Logstash container should be linked to this container.

**Step 6:** Start the Filebeat container with the following command.

```bash
$ docker run -v /tmp/filebeat.yml:/usr/share/filebeat/filebeat.yml -v ballerina.log:/usr/share/filebeat/ballerina.log --link logstash:logstash docker.elastic.co/beats/filebeat:6.2.2 
```

**Note:**
* `-v` flag is used for bind mounting, where the container will read the file from the host machine.
Hence bind mounting the configuration file and log file means that Filebeat container should be set up in the same
host where the log file is being generated.

**Step 7:** Access Kibana to visualize the logs at `http://localhost:5601` and click on discover and perform more log analysis.

[Prometheus]: https://prometheus.io/
[Grafana]: https://grafana.com/
[Jaeger]: https://www.jaegertracing.io/
