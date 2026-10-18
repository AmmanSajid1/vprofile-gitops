# RabbitMQ Prometheus Metrics Discovery

## Problem

While reviewing the Grafana Kubernetes dashboards, RabbitMQ was consistently consuming approximately 170–180 millicores of CPU, despite having a CPU request of 100m.

We wanted to investigate RabbitMQ's behaviour using application-specific metrics rather than relying solely on Kubernetes container metrics.

However, querying `rabbitmq_queue_messages` in Grafana initially returned no data.

## Investigation

### 1. Check RabbitMQ health

Reviewed the RabbitMQ pod and application logs.

RabbitMQ appeared healthy, with no obvious errors explaining its CPU consumption.

The observed CPU usage was relatively consistent rather than a sudden spike.

### 2. Check Prometheus metrics availability

RabbitMQ's Prometheus plugin was enabled, exposing metrics on port `15692`.

However, the Kubernetes Service `vpromq01` only exposed the AMQP port `5672`.

The metrics endpoint was therefore not available through the Service.

### 3. Check Prometheus discovery configuration

The Prometheus instance was configured to discover ServiceMonitors carrying the Helm release label:

```yaml
release: monitoring
```

The RabbitMQ Service also needed a stable label that a ServiceMonitor could select.

## Resolution

### Expose RabbitMQ metrics through its Service

Updated the `vpromq01` Service in the GitOps repository to expose both ports:

```yaml
metadata:
  labels:
    app: vprofile-rabbitmq

spec:
  ports:
    - name: amqp
      port: 5672
      targetPort: 5672
      protocol: TCP

    - name: metrics
      port: 15692
      targetPort: 15692
      protocol: TCP
```

The existing Service selector was retained.

### Create a ServiceMonitor

Added a `ServiceMonitor` to the development Kustomize overlay:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vprofile-rabbitmq
  namespace: dev
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - dev
  selector:
    matchLabels:
      app: vprofile-rabbitmq
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

The ServiceMonitor was included in the overlay's `kustomization.yaml`.

Changes were merged through a GitHub pull request and applied automatically by Argo CD.

## Verification

In Grafana Explore, the following query returned a RabbitMQ metric series:

```promql
rabbitmq_queue_messages
```

The reported value was `0`, indicating no queued messages at the time of the query.

The returned labels included:

- `namespace="dev"`
- `service="vpromq01"`
- `endpoint="metrics"`
- `job="vpromq01"`

This confirmed that Prometheus was successfully collecting RabbitMQ application metrics.

An additional useful scrape-health query is:

```promql
up{namespace="dev",service="vpromq01"}
```

A value of `1` indicates a successful scrape.

## Root cause

The missing metrics were caused by incomplete Prometheus discovery configuration, not a confirmed RabbitMQ application failure.

The RabbitMQ metrics endpoint was enabled, but the Kubernetes Service did not expose port `15692`, and a matching ServiceMonitor had not yet been configured.

## Lessons learned

- Kubernetes resource metrics and application metrics provide different levels of visibility.
- Enabling an application's metrics endpoint does not automatically make it discoverable by Prometheus.
- ServiceMonitor selectors must match Service labels.
- ServiceMonitor metadata must satisfy the Prometheus instance's discovery selectors.
- GitOps-managed monitoring resources should be committed and reconciled through Argo CD.
- CPU usage above a Kubernetes CPU request does not, by itself, indicate an application fault.