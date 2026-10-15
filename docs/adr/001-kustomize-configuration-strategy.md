# ADR 001: Kubernetes Configuration Strategy with Kustomize

**Status:** Accepted

## Context

The vProfile application is deployed across three environments: development, staging and production.

Each environment uses the same core application components:

- Tomcat application
- RabbitMQ
- Memcached

However, environments have different requirements for resource allocation, replica counts, namespaces and container image versions.

Maintaining separate copies of all Kubernetes manifests would introduce duplication and increase the risk of configuration drift.

We needed a configuration management approach that allowed environments to share common definitions while retaining independent configuration.

## Decision

Use **Kustomize with a shared base and environment-specific overlays**.

The repository is structured as follows:

```text
apps/vprofile/
├── base/
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   ├── rabbitmq-deployment.yaml
│   ├── rabbitmq-service.yaml
│   ├── memcached-deployment.yaml
│   ├── memcached-service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    ├── staging/
    └── prod/
```

The base contains reusable Kubernetes Deployments and Services without environment-specific namespaces, resource allocations or replica counts.

Each overlay defines its own:

- Namespace
- Replica counts
- CPU and memory requests/limits
- Immutable container image tag
- ALB Ingress configuration

Dev and staging run in separate namespaces within the non-production EKS cluster. Production runs in a dedicated EKS cluster.

Resource patches are maintained separately for each environment rather than referencing another environment's patches. This introduces some duplication but allows independent changes without unintended cross-environment effects.

Kustomize is used for application manifests, while Helm is reserved for installing third-party platform components.

## Alternatives Considered

### Separate YAML manifests per environment

Rejected because maintaining three copies of the same Deployments and Services would increase duplication and configuration drift.

### Helm for application configuration

Helm supports templating and environment-specific values, but introduces additional template complexity.

Kustomize was preferred because the application has a relatively small number of resources and does not currently require extensive templating.

### Sharing patches between environments

Considered allowing staging to reuse dev resource patches.

Rejected because changes to dev resource allocations could unintentionally affect staging.

## Consequences

### Benefits

- Common workload definitions are maintained in one location.
- Environment differences remain explicit and reviewable.
- Configuration changes can be validated using `kubectl kustomize`.
- Immutable image tags can be updated independently during promotion.
- Kubernetes manifests remain standard YAML without application-specific templating.

### Trade-offs

- Some duplication remains between environment overlays.
- Kustomize patches can become difficult to manage as configurations grow.
- Namespace separation does not provide complete isolation between dev and staging, since both share the same EKS cluster and underlying compute resources.

## Validation

All three overlays can be rendered locally:

```bash
kubectl kustomize apps/vprofile/overlays/dev
kubectl kustomize apps/vprofile/overlays/staging
kubectl kustomize apps/vprofile/overlays/prod
```

Rendered manifests should be checked for correct namespaces, replica counts, resource allocations, image references and Ingress routing before deployment.