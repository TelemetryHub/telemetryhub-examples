# Logs in TH
Here we’ll take a look at one method of sending logs to TelemetryHub using an OpenTelemetry Collector. Using a [Filelog Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) with a collector is a simple way to export any existing log files your app already produces to an Otel backend. If you haven’t already, check out our [Quick Start guide](https://app.telemetryhub.com/docs#quickStart) to set up your TelemetryHub account.

Regardless of how you plan to deploy it, the first step in using a collector is creating the configuration file.

```yml
# otel-collector-config.yml
receivers:
  filelog:
    include: [ /var/log/*.log ]

processors:
  batch:

exporters:
  logging:
  otlp:
    endpoint: otlp.telemetryhub.com:4317
    headers:
      "X-TELEMETRYHUB-KEY": “your-telemetryhub-key”

service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch]
      exporters: [logging, otlp]
```

Add your own TelemetryHub app key to the headers in the `otlp` exporter. In this configuration, we assume the app’s log files reside in `/var/log` and the collector has access to that directory.

That’s a great starting point for the collector configuration to get things flowing, but you can tailor it exactly to your needs. You can add `operators` to the filelog receiver to parse and manipulate the logs and filter processors to the collector pipeline.

The filelog receiver has many options to parse and process your logs. This is done through `operators`. You can apply regex and json parsers, manipulate attributes, and even define operator paths to handle multiple formats. Here’s an example:
```yml
filelog:
  include: [ /var/log/*.log ]
  operators:
    - type: router
      id: get-format
      routes:
        - output: my-json-parser
          expr: 'body matches "^\\{"'
        - output: my-regex-parser
          expr: 'body matches "^.*[ ]"'
    - type: regex_parser
      id: my-regex-parser
      regex: '^(?P<time>) (?P<sev>[A-Z]*) (?P<msg>.*)$'
      timestamp:
        parse_from: attributes.time
        layout: '%Y-%m-%dT%H:%M:%S.%LZ'
      severity:
        parse_from: attributes.sev
    - type: json_parser
      id: my-json-parser
      timestamp:
        parse_from: attributes.time
        layout: '%Y-%m-%dT%H:%M:%S.%LZ'
```

The first operator, the `router`, checks each log and routes it to the appropriate operator based on the regex expression. The following two operators will actually parse the log lines and output for the exporter.
In this example, the router uses the ID of the other operators to route the logs, but if you want to send logs through a series of operators, you can omit IDs and the logs will waterfall through the operators in the order they are defined.

You may not want to ship off every single log entry from your app. This is where [Filter processors](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/processor/filterprocessor/README.md) come into play. For logs, a filter can be applied to resource attributes or the log data to capture only the important logs.

```yml
processors:
  filter/regexp:
    logs:
      include:
        match_type: regexp
        resource_attributes:
          - key: host.name
            value: prefix.*
  filter/severity_number:
     logs:
       include:
         severity_number:
           min: "WARN"
           match_undefined: true
   filter/bodies:
     logs:
       include:
         match_type: regexp
         bodies:
         - ^IMPORTANT RECORD
```

The first filter includes all logs with an attribute that matches `prefix.*`. The second captures all logs with a minimum severity of `WARN`. And the third looks for a regex match on the body of the log. Instead of `include` you also have the option to `exclude` logs that match the filter. Remember, for these processors to go into effect, you need to add them to the service pipeline. 

```yml
service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch, filter/severity_number]
      exporters: [logging, otlp]
```

Sidenote: filter processors can also be applied to traces and metrics if you’ve already instrumented your app.

Now let’s run the collector. You have a few options here, and one might suit your environment better than the others. The collector can be used as an `agent` running alongside the application on the same host, or as a `gateway` where the collector is deployed independently, such as in its own Docker container where all services in a cluster can reach it.

First, you can run the collector directly as a stand alone process. You can download the latest release from [https://github.com/open-telemetry/opentelemetry-collector-releases/releases](https://github.com/open-telemetry/opentelemetry-collector-releases/releases). Be sure to get the appropriate `otel-contrib` release for your machine.

```bash
$ wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.75.0/otelcol-contrib_0.75.0_darwin_arm64.tar.gz
$ gunzip -c otelcol-contrib_0.75.0_darwin_arm64.tar.gz | tar xopf -
$ ./otelcol-contrib --config ./otel-collector-config.yml
```

OpenTelemetry also provides Docker images. You can use the image `otel/opentelemetry-collector-contrib` and run it with the configuration file you created:

```bash
docker run \
	-v “${PWD}/otel-collector-config.yml”:/etc/otel-collector-config.yml \
	-v /var/log:/var/log
	otel/opentelemetry-collector-contrib \
	--config /etc/otel-collector-config.yml
```

Here, we mount the volume `/var/log:/var/log` so the collector can access the files. 

Of course the collector can also be deployed with Docker Compose, which is probably the easiest if you are already using compose for the application.

Add the `otel-collector` service to the compose file:

```
otel-collector:
  image: otel/opentelemetry-collector-contrib
  command: [--config=/etc/otel-collector-config.yml]
  volumes:
    - ./config/otel-collector-config.yml:/etc/otel-collector-config.yml
    - /var/log/:/var/log
```

Similar to running the stand alone docker container, we mount volumes for the configuration file and the log director. It’s important here that the log files the app container produces are accessible from the collector container.

If you use Kubernetes for your production environment, the OpenTelemetry collector can be [added to your cluster with Helm](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collectorhttps://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector).

Now, go ahead and run your app. The collector will tail the specified files and export your logs to TelemetryHub.

