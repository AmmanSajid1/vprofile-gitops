# ADR 002: ALB Ingress Architecture and Environment Routing

**Status:** Accepted  

## Context

The vProfile application needs to be accessible over HTTPS across development, staging and production environments.

The application runs on Amazon EKS, with worker nodes deployed in private subnets. Development and staging share a non-production EKS cluster, while production runs in a separate cluster.

We needed an ingress architecture that would:

- Expose the application securely without making worker nodes publicly accessible.
- Support custom domains and HTTPS.
- Minimise infrastructure costs in non-production.
- Maintain isolation between production and non-production.
- Integrate with Kubernetes Services and automatically track changes to application Pods.

## Decision

Use the **AWS Load Balancer Controller** to provision and manage internet-facing Application Load Balancers (ALBs) through Kubernetes Ingress resources.

### 1. ALB with IP target mode

Configure the Ingress with:

```yaml
alb.ingress.kubernetes.io/scheme: internet-facing
alb.ingress.kubernetes.io/target-type: ip
```

The ALB is deployed into public subnets and forwards requests directly to the private VPC IP addresses assigned to application Pods by the AWS VPC CNI.

This avoids exposing worker nodes directly to the internet.

The application remains behind a Kubernetes ClusterIP Service, which provides a stable service definition for the Ingress backend.

The AWS Load Balancer Controller is responsible for registering and deregistering Pod IPs as targets when workloads change.

### 2. Shared ALB for non-production

Development and staging share one internet-facing ALB using an IngressGroup:

```yaml
alb.ingress.kubernetes.io/group.name: vprofile-nonprod
```

Host-based routing directs traffic to the appropriate environment:

| Hostname | Namespace |
|---|---|
| dev.ammandevops.xyz | dev |
| staging.ammandevops.xyz | staging |

Each environment maintains its own Ingress and application Service.

This reduces the cost of running separate ALBs for environments that do not require complete infrastructure isolation.

Only trusted, repository-managed Ingress configurations should be allowed to join this group because group membership can affect shared ALB routing and configuration.

### 3. Dedicated production ALB

Production runs in a separate EKS cluster and uses its own ALB.

Its Ingress does not specify an IngressGroup because no ALB sharing is currently required.

Production traffic uses:

```text
app.ammandevops.xyz
```

Separating production from non-production reduces the risk of non-production configuration changes affecting production ingress infrastructure.

### 4. HTTPS termination using ACM

TLS certificates are provisioned through Terraform in the shared infrastructure layer.

Separate ACM certificates are used for:

- Non-production: `dev.ammandevops.xyz` and `staging.ammandevops.xyz`
- Production: `app.ammandevops.xyz`

Each Ingress references the appropriate ACM certificate ARN.

The ALB listens on ports 80 and 443, with HTTP requests redirected to HTTPS:

```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

TLS terminates at the ALB. Traffic from the ALB to the Tomcat application uses HTTP within the VPC.

This simplifies application configuration by avoiding certificate management inside Tomcat.

### 5. Application routing

Each Ingress routes requests to the `vprofile-app` ClusterIP Service on port 80.

The Service forwards traffic to Tomcat on port 8080.

The request flow is:

```text
Internet
   |
   v
Route 53 DNS
   |
   v
Public ALB (HTTPS :443)
   |
   | HTTP
   v
Private Pod IP (Tomcat :8080)
```

The Kubernetes Service is used to identify the application backend, while the ALB targets Pod IPs directly.

### 6. Session affinity

The inherited application uses HTTP sessions for authentication, with session state maintained locally by Tomcat.

ALB target-group stickiness is enabled:

```yaml
alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=86400
```

This helps direct subsequent requests from the same client to the same application target.

However, stickiness does not provide session replication or durable session storage.

If a Tomcat Pod terminates or becomes unavailable, users whose sessions were stored on that Pod may need to authenticate again.

This limitation is accepted for the current implementation.

### 7. Health checks

The ALB checks the application root endpoint:

```yaml
alb.ingress.kubernetes.io/healthcheck-path: /
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/success-codes: '200'
```

Local testing confirmed that `GET /` returns HTTP 200 without authentication.

This provides basic HTTP availability monitoring but does not verify connectivity to downstream dependencies such as MySQL, RabbitMQ or Memcached.

Dedicated health endpoints are deferred to future application development.

## Alternatives Considered

### NGINX Ingress Controller

Considered using NGINX as the Kubernetes ingress controller.

AWS Load Balancer Controller was selected because it integrates directly with AWS ALB, ACM, VPC networking and target groups, reducing the need to operate an additional ingress proxy layer.

### ALB instance target mode

Instance targeting routes traffic through worker-node IP addresses and NodePorts.

IP targeting was selected to route directly to Pod IPs, avoiding the NodePort requirement and providing Pod-level target registration.

### Separate ALBs for development and staging

This would provide stronger isolation between environments.

Rejected for the current non-production deployment because of the additional cost and limited benefit for this project.

### One shared ALB across all environments

Rejected because production runs in a separate EKS cluster and requires an independent ingress lifecycle.

### TLS termination inside Tomcat

Rejected to avoid distributing and managing TLS certificates within application containers.

ACM-managed certificates and ALB TLS termination simplify certificate management.

## Consequences

### Benefits

- Worker nodes and application Pods remain in private subnets.
- ALBs are provisioned declaratively through Kubernetes Ingress resources.
- IP target registration adapts to changes in application Pods.
- Non-production environments share ALB infrastructure to reduce costs.
- Production maintains a separate ingress lifecycle.
- HTTPS certificates are managed centrally through ACM.
- Host-based routing provides independent URLs for each environment.

### Trade-offs and Limitations

- ALBs introduce ongoing AWS charges.
- Development and staging share an ALB, creating a shared failure and configuration domain.
- TLS is terminated at the ALB; backend traffic is HTTP rather than end-to-end TLS.
- IP targeting requires correct VPC networking and security-group rules between the ALB and Pods.
- ALB target registration and deregistration are not instantaneous during scaling or deployments.
- Session stickiness does not guarantee session availability during Pod failures.
- The application does not currently expose dependency-aware readiness endpoints.

## Validation

Before considering this architecture operationally validated, the following checks must be completed after EKS deployment:

1. Confirm the AWS Load Balancer Controller successfully provisions the ALBs.
2. Confirm dev and staging share one ALB while production uses a separate ALB.
3. Verify that ALB targets register as private Pod IP addresses and become healthy.
4. Verify DNS resolution and HTTPS access for all three hostnames.
5. Confirm HTTP requests redirect to HTTPS.
6. Test application routing, authentication and session behaviour.
7. Verify that traffic is removed from unhealthy or terminating Pods.

These checks remain pending until the infrastructure is deployed.