# Demo OpenTelemetry setup

This directory contains all the OpenTelemetry collector configuration that I described in my presentation.
It assumes you have Loki (for logs), Tempo (for traces), and Mimir (for metrics), but these are simple to replace
by changing the collector exporters.

Also included are some nice grafana dashboards for RED metrics and drilling down into traces.

Enjoy!

# Instructions

1. Go through the yamls and replace anything in `<angular brackets>` with your values
2. Install helm charts with 
   ```shell
   helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
   ```
3. Install the collectors with:
    ```shell
    kubectl create ns otel
    helm install open-telemetry/opentelemetry-collector -f opentelemetry-collector/values.yaml --version=0.69.2
    helm install open-telemetry/opentelemetry-collector -f opentelemetry-collector-lb/values.yaml --version=0.69.2
    helm install open-telemetry/opentelemetry-collector -f opentelemetry-collector-daemonset/values.yaml --version=0.69.2
    kubectl apply -k ./opentelemetry-lb-role
    ```
4. Label your app pods with `app` and `team`
5. Choose if you want to use auto-instrumentation or not
    1. If so, run:
       ```shell
       helm install open-telemetry/opentelemetry-operator -f opentelemetry-operator/values.yaml --version=0.69.2    
       kubectl apply -k ./opentelemetry-instrumentation 
       ```
       and add an annotation to your pods:
       ```yaml
       instrumentation.opentelemetry.io/inject-python: "otel/instrumentation"
       ```
       replacing `inject-python` with any of:
       - `inject-dotnet`
       - `inject-go` (needs a bit more config: https://opentelemetry.io/docs/kubernetes/operator/automatic/#opt-in-a-go-service)
       - `inject-java`
       - `inject-nodejs`
   2. Otherwise, [instrument your app yourself](https://opentelemetry.io/docs/instrumentation/) add these env vars to your pods:
       ```yaml
            - name: OTEL_RESOURCE_ATTRIBUTES # special values: otel-collector-lb replaces them with the real values
              value: service.name=namespace/app,service.instance.id=pod_ip:container
            - name: OTEL_EXPERIMENTAL_RESOURCE_DETECTORS
              value: otel,process,container
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://otel-collector-lb.otel.svc.cluster.local:4317
            - name: OTEL_PROPAGATORS
              value: tracecontext,baggage
            - name: OTEL_TRACES_SAMPLER
              value: always_on
            - name: OTEL_METRICS_EXPORTER
              value: otlp
            - name: OTEL_TRACES_EXPORTER
              value: otlp
            - name: OTEL_LOGS_EXPORTER
              value: none
            - name: OTEL_METRIC_EXPORT_INTERVAL
              value: "10000"
       ```
6. Import the grafana dashboards from `./grafana`. Change the datasource variables to suit your deployment
   (we're using an `environment` variable to choose them, e.g. our metrics datasource is `mimir-$environment`)
   1. From spanmetrics dashboard graphs you can see RED metrics. 
   2. Click on a line on any graph then `Matching Traces` to drilldown into matching traces 