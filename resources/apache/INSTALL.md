# Installing the Apache exporter
Apache provides metics in its own format via its [ServerStatus module](https://httpd.apache.org/docs/2.4/mod/mod_status.html).
To enable this module, be sure to include (or uncomment) this line in your apache configuration file:
```xml
LoadModule status_module modules/mod_status.so
<Location "/server-status">
  SetHandler server-status
</Location>
```
The native statistics page will be available on http://your-ip/server-status/?auto.
This is a basic configuration.
You can find more information on how to configure access control to this endpoint
in the [apache documentation for this module](https://httpd.apache.org/docs/2.4/mod/mod_status.html).
To translate these metrics to Prometheus format we will use the [Apache Prometheus Exporter](https://github.com/Lusitaniae/apache_exporter).

In the deployment we will use this exporter as a sidecar of the Apache server instance,
passing as parameter the endpoint to scrape the metrics in the _--scrape_uri_ parameter

# Installing the Grok exporter
We will use the [Grok exporter](https://hub.docker.com/r/palobo/grok_exporter) to parse the logs of Apache and obtain metrics of errors and requests codes.
The configuration used here is based in the one published in this [blog post of Robust Perception](https://www.robustperception.io/getting-metrics-from-apache-logs-using-the-grok-exporter).
This configuration relies on Apache to produce common standard logs. Other log format may require other Grok configuration.

To configure Apache server to produce common logs, include (or uncomment) this in your Apache configuration file:
```xml
<IfModule log_config_module>
       LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
       CustomLog /usr/local/apache2/logs/accesss.log common
</IfModule>
 ```

To the Grok exporter we will pass this configuration with a ConfigMap:
```yaml
global:
    config_version: 2
input:
    type: file
    path: /tmp/logs/accesss.log
    readall: true
    fail_on_missing_logfile: false
grok:
    patterns_dir: ./patterns
metrics:
    - type: counter
      name: apache_http_response_codes_total
      help: HTTP requests to Apache
      match: '%{COMMONAPACHELOG}'
      labels:
          method: '{{.verb}}'
          code: '{{.response}}'
    - type: gauge
      name: apache_http_response_bytes_total
      help: Size of HTTP responses
      match: '%{COMMONAPACHELOG}'
      cumulative: true
      value: '{{.bytes}}'
      labels:
          method: '{{.verb}}'
          code: '{{.response}}'
    - type: gauge
      name: apache_http_last_request_seconds
      help: Timestamp of the last HTTP request
      match: '%{COMMONAPACHELOG}'
      value: '{{timestamp "02/Jan/2006:15:04:05 -0700" .timestamp}}'
      labels:
          method: '{{.verb}}'
          code: '{{.response}}'
server:
    port: 9144
```

Next, we will deploy the Grok exporter as a sidecar of the Apache server and we will create a shared
volume to share the access log so the Grok exporter can parse it and generate metrics.

# Merging the two exporters
As there is not a way to annotate two ports to be scrape by Prometheus, we will use another sidecar
to merge both exporters into a single endpoint that we will expose and annotate.

We will use the [Exporter merger](https://github.com/rebuy-de/exporter-merger) and we will pass
the url address of both exporter in the _MERGER_URLS_ environment variable.

# Deploy in Kubernetes
To deploy the configuration example in Kubernetes, just download the config file and run:
```
kubectl apply -f apache-deploy.yaml
```

# Sysdig Agent configuration
In the Sysdig Agent configuration, you have to add the following code to include the label 'app' as a metric label.
```yaml
process_filter:
  - include:
      kubernetes.pod.annotation.prometheus.io/scrape: true
      conf:
        path: "{kubernetes.pod.annotation.prometheus.io/path}"
        port: "{kubernetes.pod.annotation.prometheus.io/port}"
        tags:
          app: "{kubernetes.pod.label.app}"
```

You can download the sample configuration file below and apply it by:
```bash
kubectl apply -f sysdig-agent-config.yaml
```