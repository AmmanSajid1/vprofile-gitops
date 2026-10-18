# ADR 007: Monitoring Resource Ownership

**Status:** Accepted

## Context

The vProfile monitoring stack is installed using Terraform and Helm, while application workloads are deployed through Argo CD.

During Grafana configuration, we needed to manage administrator credentials securely using AWS Secrets Manager and External Secrets Operator (ESO).

We also needed to introduce application-specific monitoring resources, such as RabbitMQ ServiceMonitors.

This raised an ownership question: should Terraform manage all monitoring-related Kubernetes resources, or should application and namespace-level configuration remain under GitOps?

Managing the same resources through both Terraform and Argo CD would create conflicting reconciliation responsibilities.

## Decision

Separate monitoring resource ownership according to responsibility.

### Terraform: Platform infrastructure

Terraform manages:

- Installation and configuration of `kube-prometheus-stack` through Helm.
- Prometheus, Grafana and Alertmanager platform settings.
- The EBS CSI driver and gp3 StorageClass.
- Prometheus persistent storage configuration.
- Resource requests, limits and retention settings.
- AWS IAM permissions required for monitoring-related secret access.

These resources belong to the infrastructure lifecycle and are managed in `vprofile-infra`.

### Argo CD: Kubernetes monitoring configuration

Argo CD manages namespace-level monitoring configuration, including:

- Monitoring namespace `SecretStore`.
- Grafana administrator `ExternalSecret`.
- Application-specific `ServiceMonitor` resources.
- Future application monitoring configuration.

These resources are version-controlled in `vprofile-gitops`.

The monitoring configuration has a dedicated Argo CD Application pointing to `platform/monitoring`.

Application-specific ServiceMonitors, such as RabbitMQ's, remain alongside their respective application overlays.

### Grafana administrator credentials

Grafana's administrator password is stored in AWS Secrets Manager under:

`vprofile/nonprod/grafana_admin`

The secret is managed in the shared Terraform configuration to preserve it independently of temporary non-production infrastructure.

External Secrets Operator retrieves the password and creates the Kubernetes Secret `grafana-admin-credentials` in the monitoring namespace.

The Grafana Helm release references this existing Kubernetes Secret rather than embedding the password in Terraform configuration or Git.

ESO accesses AWS Secrets Manager through EKS Pod Identity with scoped IAM permissions.

## Alternatives considered

### Manage everything through Terraform

This would centralise monitoring configuration but require Terraform to own application-specific Kubernetes resources.

Rejected because application monitoring changes should follow the GitOps workflow.

### Manage the entire monitoring stack through Argo CD

This would provide a single Kubernetes reconciliation mechanism.

Not selected because the existing platform installation, storage and resource configuration were already managed through Terraform.

### Manage the same resources through Terraform and Argo CD

Rejected because competing controllers could overwrite changes and introduce configuration drift.

### Store Grafana credentials directly in Helm values

Simpler initially but risks exposing sensitive values through configuration and Terraform state.

Rejected in favour of AWS Secrets Manager and ESO.

## Consequences

### Benefits

- Clear ownership boundaries between infrastructure and application configuration.
- Reduced risk of Terraform and Argo CD reconciliation conflicts.
- Application monitoring changes follow the normal GitOps review process.
- Credentials remain outside Git and Helm values.
- Shared secrets can survive non-production infrastructure teardown.

### Trade-offs and limitations

- Monitoring depends on multiple components: Terraform, Helm, Argo CD, ESO and AWS Secrets Manager.
- Bootstrap ordering matters because Grafana requires its Kubernetes Secret to exist.
- Teardown must account for resources managed by different controllers.
- IAM permissions and ExternalSecret reconciliation introduce additional troubleshooting considerations.

## Future improvements

- Automate monitoring bootstrap ordering more explicitly.
- Add validation for ServiceMonitor discovery and ExternalSecret readiness.
- Define consistent monitoring conventions for staging and production.
- Document disaster recovery and credential rotation procedures.