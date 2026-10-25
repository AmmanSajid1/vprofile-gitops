# vProfile GitOps

Kubernetes deployment configuration for the vProfile application, managed using Kustomize and Argo CD.

This repository separates application deployment configuration from source code and infrastructure provisioning. It defines the desired Kubernetes state for development and staging, with Argo CD responsible for reconciling changes when the EKS cluster is running.

## Architecture

The deployment uses the following components:

- **Amazon EKS:** Hosts the Kubernetes workloads.
- **Argo CD:** Manages GitOps deployment reconciliation.
- **Kustomize:** Maintains shared manifests and environment-specific configuration.
- **AWS Load Balancer Controller:** Provisions Application Load Balancers from Kubernetes Ingress resources.
- **External Secrets Operator:** Synchronises secrets from AWS Secrets Manager and Systems Manager Parameter Store.
- **Flyway:** Applies database schema migrations before application deployment.
- **Prometheus and Grafana:** Provide monitoring and observability.

The application consists of a Java application running on Tomcat, Memcached for caching, RabbitMQ for messaging, and an external Amazon RDS MySQL database.

## Repository Structure

```text
.
├── apps/
│   └── vprofile/
│       ├── base/               # Shared Kubernetes resources
│       └── overlays/
│           ├── dev/            # Development configuration
│           ├── staging/        # Staging configuration
│           └── prod/           # Production overlay scaffold
├── argocd/
│   └── applications/           # Argo CD Application definitions
├── platform/
│   └── monitoring/             # Monitoring secret integration
└── docs/
    ├── adr/                    # Architecture Decision Records
    └── troubleshooting/        # Investigation and resolution notes
```

## Environments

Development and staging run in separate Kubernetes namespaces within the same non-production EKS cluster.

Each environment has its own configuration, database resources and secrets. This allows development and staging to be managed independently while sharing the underlying cluster infrastructure.

| Environment | URL | Status |
|---|---|---|
| Development | `dev.ammandevops.xyz` | Implemented |
| Staging | `staging.ammandevops.xyz` | Implemented |
| Production | — | Overlay scaffold only |

The non-production infrastructure is provisioned through Terraform and can be destroyed when not required to avoid ongoing AWS costs.

The production overlay is not currently connected to an operational production Argo CD deployment.

## Kubernetes Deployment

Shared application resources are defined under `apps/vprofile/base/`.

Environment-specific overlays configure:

- Container image versions and resource requirements.
- Namespace and environment variables.
- Application Load Balancer Ingress and TLS.
- External Secrets and AWS integrations.
- Database bootstrap and Flyway migration Jobs.
- Memcached and RabbitMQ resources.

Argo CD uses deployment ordering to coordinate database initialisation and migrations before the application workloads are synchronised.

This keeps database schema changes and application releases aligned.

## CI/CD and Image Promotion

Application images are built in the [vprofile-app repository](https://github.com/AmmanSajid1/vprofile-app) using GitHub Actions.

The release process follows a build-once, promote-the-same-artifact approach.

### Development

1. CI tests and builds the application and Flyway images.
2. Images are published to Amazon ECR using immutable Git commit SHA tags.
3. GitHub Actions opens a pull request updating the development Kustomize overlay.
4. The pull request is reviewed and merged into the protected GitOps `main` branch.
5. Argo CD detects the change and synchronises development when the cluster is running.

### Staging

1. A manually triggered GitHub Actions workflow reads the approved development image versions.
2. It verifies that the application and Flyway images exist in ECR.
3. It updates the staging Kustomize overlay through a separate pull request.
4. After review and merge, Argo CD synchronises staging when the cluster is running.

Staging promotion reuses the same container images without rebuilding them.

The promotion workflow is idempotent: if development and staging already use the same image tags, no new pull request is created.

For implementation details, see the [CI/CD Release Pipeline documentation](https://github.com/AmmanSajid1/vprofile-app/blob/main/docs/release-pipeline.md).

## Secrets Management

Application credentials are not stored directly in Kubernetes manifests.

External Secrets Operator retrieves the required values from AWS Secrets Manager and Systems Manager Parameter Store.

Kubernetes service accounts use AWS IAM integration to access the required AWS resources without embedding long-lived AWS access keys in application configuration.

Development and staging use separate secret resources.

## Monitoring

The monitoring stack uses Prometheus, Grafana and Alertmanager.

Kubernetes ServiceMonitor resources support metrics collection from application dependencies, including RabbitMQ.

Monitoring configuration and deployment are managed separately from application workloads.

## Related Repositories

- [vprofile-app](https://github.com/AmmanSajid1/vprofile-app) — Application source, Dockerfiles, database migrations and CI/CD workflows.
- [vprofile-infra](https://github.com/AmmanSajid1/vprofile-infra) — Terraform infrastructure, EKS platform components and deployment runbooks.

## Documentation

### Architecture Decision Records

- [ADR 001 — Kustomize Configuration Strategy](docs/adr/001-kustomize-configuration-strategy.md)
- [ADR 002 — ALB Ingress Architecture](docs/adr/002-alb-ingress-architecture.md)
- [ADR 003 — Workload Security and Availability](docs/adr/003-workload-security-and-availability.md)
- [ADR 004 — Healthchecks and Session Affinity](docs/adr/004-healthchecks-and-session-affinity.md)
- [ADR 005 — Database Bootstrap and Migration Strategy](docs/adr/005-database-bootstrap-and-migration-strategy.md)
- [ADR 006 — GitOps Image Promotion](docs/adr/006-gitops-image-promotion.md)
- [ADR 007 — Monitoring Resource Ownership](docs/adr/007-monitoring-resource-ownership.md)

### Troubleshooting

- [RabbitMQ Prometheus Scrape Discovery](docs/troubleshooting/rabbitmq-prometheus-scrape-discovery.md)
- [Spring Security Healthcheck Investigation](docs/troubleshooting/spring-security-healthcheck-investigation.md)