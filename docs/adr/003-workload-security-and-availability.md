# ADR 003: Kubernetes Workload Security and Availability

**Status:** Accepted  

## Context

The vProfile application consists of three Kubernetes workloads:

- **Tomcat:** Hosts the Java web application.
- **RabbitMQ:** Provides message-broker functionality.
- **Memcached:** Provides in-memory caching.

MySQL is hosted externally using Amazon RDS rather than running inside Kubernetes.

The application must support development, staging and production environments with different resource and availability requirements.

We needed to establish sensible defaults for container security, resource allocation, scaling and workload persistence without introducing unnecessary complexity or AWS costs.

## Decision

### 1. Run application containers as non-root

Configure workloads to run using non-root user IDs.

The Tomcat application uses UID `999`, matching the dedicated user created in the application container image.

RabbitMQ and Memcached also use non-root execution, with Memcached configured to use UID `11211`.

Where supported, containers drop Linux capabilities to reduce unnecessary privileges.

This limits the impact of container compromise compared with running processes as root.

These controls are applied through Kubernetes Pod and container security contexts.

### 2. Define resource requests and limits

Each environment specifies CPU and memory requirements through Kustomize overlays.

CPU requests are used to inform Kubernetes scheduling and provide a baseline allocation.

Memory requests and limits are configured to reduce the risk of workloads consuming excessive node memory.

CPU limits are intentionally omitted to avoid unnecessary CPU throttling during bursts of activity.

Production receives larger resource allocations than development and staging.

These settings are initial estimates and should be reviewed using actual resource-utilisation metrics after deployment.

### 3. Environment-specific replica counts

The initial deployment uses the following replica counts:

| Workload | Dev | Staging | Production |
|---|---|---|---|
| Tomcat | 1 | 1 | 2 |
| RabbitMQ | 1 | 1 | 1 |
| Memcached | 1 | 1 | 1 |

Two Tomcat replicas are configured in production to support basic application redundancy and rolling deployments.

However, replica count alone does not guarantee high availability.

Without scheduling constraints, both replicas could be placed on the same worker node or in the same Availability Zone.

Pod topology spread constraints or anti-affinity should be considered to distribute production replicas across failure domains.

The initial configuration also does not include a Horizontal Pod Autoscaler. Scaling decisions will be revisited after collecting workload metrics.

### 4. Keep Memcached stateless

Memcached is deployed as a single-replica Deployment without persistent storage.

Its contents are temporary cached data and are expected to be lost when the Pod restarts or is replaced.

Persistent volumes are therefore unnecessary.

This assumes the application can tolerate cache loss, although its behaviour during Memcached outages still needs to be validated.

A single replica is accepted for the initial deployment.

### 5. Run RabbitMQ without persistent storage initially

RabbitMQ is deployed as a single-replica Deployment without a PersistentVolumeClaim.

This simplifies deployment and reduces storage-management complexity.

However, messages and broker state stored only within the container filesystem may be lost when the Pod is replaced or rescheduled.

RabbitMQ is therefore **not configured for durable messaging or high availability**.

This is an accepted limitation of the current portfolio deployment, not a recommended production configuration for workloads requiring reliable message delivery.

If durable messaging becomes a requirement, alternatives include:

- RabbitMQ with persistent volumes and appropriate recovery procedures.
- A highly available RabbitMQ deployment using suitable queue replication.
- A managed messaging service, where compatible with application requirements.

### 6. Use Amazon RDS for persistent relational data

MySQL is hosted on Amazon RDS rather than inside the EKS cluster.

This separates database persistence and lifecycle management from application Pods.

The intended infrastructure configuration is:

- Non-production: Single-AZ RDS deployment.
- Production: Multi-AZ RDS deployment.

This provides stronger database availability in production without requiring the team to manage MySQL replication directly in Kubernetes.

Database connectivity, credentials and access controls must be configured separately.

## Alternatives Considered

### Running containers as root

Rejected because the application workloads do not require unrestricted root privileges during normal operation.

### Setting CPU limits on all workloads

Considered for strict resource isolation.

Not selected initially because CPU limits can cause throttling, even when spare CPU capacity exists on a node.

CPU requests and memory limits provide a starting point, with monitoring required to determine whether further restrictions are appropriate.

### Deploying RabbitMQ as a StatefulSet

A StatefulSet with persistent storage would better support stable broker identity and durable data.

Deferred because the current deployment does not require guaranteed message persistence and the additional operational complexity is not justified at this stage.

### Running MySQL inside Kubernetes

Rejected because RDS provides managed backups, storage and database availability features, reducing the operational responsibilities of the EKS platform.

### Deploying multiple replicas of every component

Rejected because increasing replica counts does not automatically provide high availability for stateful or session-dependent services.

RabbitMQ clustering, cache distribution and application session handling each require additional design considerations.

## Consequences

### Benefits

- Non-root execution reduces container privileges.
- Shared workload definitions provide consistent security settings.
- Environment-specific resource allocation supports different workload requirements.
- Omitting CPU limits avoids unnecessary CPU throttling.
- Two production Tomcat replicas provide a foundation for application redundancy.
- RDS separates persistent database storage from the Kubernetes application lifecycle.
- Stateless supporting services keep the initial deployment relatively simple and inexpensive.

### Trade-offs and Limitations

- Resource allocations have not yet been validated against real workload metrics.
- Production Tomcat replicas are not yet guaranteed to run across separate nodes or Availability Zones.
- Local Tomcat HTTP sessions are not replicated between Pods.
- Memcached data is lost when its Pod is replaced.
- RabbitMQ does not provide durable messaging or broker high availability.
- RabbitMQ and Memcached remain single points of failure.
- The current configuration does not provide automatic horizontal scaling.
- Non-root execution alone does not provide complete container isolation.

## Future Improvements

Before treating the platform as production-ready for a business-critical application:

1. Introduce topology spread constraints for production Tomcat replicas.
2. Validate resource requests and memory limits using Prometheus/Grafana metrics.
3. Test application behaviour during RabbitMQ and Memcached outages.
4. Evaluate persistent or managed messaging if message durability is required.
5. Consider HPA once appropriate application metrics and scaling thresholds are established.
6. Validate RDS failover behaviour and application reconnection handling.

## Validation

The security contexts, resource allocations and replica counts can be inspected using the rendered Kustomize manifests:

```bash
kubectl kustomize apps/vprofile/overlays/dev
kubectl kustomize apps/vprofile/overlays/staging
kubectl kustomize apps/vprofile/overlays/prod
```

Runtime validation, including Pod scheduling, resource utilisation, restart behaviour and failure recovery, remains pending until the EKS environments are deployed.