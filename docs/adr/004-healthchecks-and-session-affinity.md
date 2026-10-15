# ADR 004: Application Health Checks and Session Affinity

**Status:** Accepted  

## Context

The vProfile application is an inherited Java web application running on Apache Tomcat with Spring Security authentication.

When deploying the application to Kubernetes behind an AWS Application Load Balancer (ALB), we needed to determine:

- Which HTTP endpoint should be used for Kubernetes and ALB health checks.
- Whether the application requires session affinity when running multiple Tomcat replicas.
- Whether modifying the application to introduce dedicated health endpoints was justified.

The application does not currently expose dedicated liveness and readiness endpoints.

Our objective is to deploy and operate the existing application without introducing unnecessary changes to its Java codebase.

## Decision

### 1. Use the root endpoint for health checks

Local testing produced the following results:

| Request | HTTP status | Observation |
|---|---|---|
| `GET /login` | 405 | Endpoint does not support GET |
| `GET /` | 200 | Returns the login page |
 
The root endpoint is therefore used for the existing Kubernetes startup, liveness and readiness probes, as well as ALB health checks.

The ALB Ingress uses:

```yaml
alb.ingress.kubernetes.io/healthcheck-path: /
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/success-codes: '200'
```

This provides a basic HTTP availability check without modifying the application.

However, an HTTP 200 response from `/` does not establish that the application can successfully access MySQL, RabbitMQ or Memcached.

The readiness probe therefore does not currently represent complete application readiness.

### 2. Retain existing application code

We considered implementing dedicated endpoints such as:

```text
/health/live
/health/ready
```

These could distinguish between an application process being alive and being ready to serve requests that depend on external services.

However, implementing them would require changes to Spring MVC and Spring Security configuration, along with application-specific dependency checks.

We decided to defer this work because:

- The inherited application already exposes a working HTTP endpoint.
- Introducing Java changes creates additional testing and maintenance responsibilities.
- The current project focuses on platform engineering, infrastructure automation and GitOps.
- Dedicated health endpoints would ideally be developed with the application development team.

The limitation is documented rather than hidden.

### 3. Enable ALB session stickiness

During local testing, the root endpoint returned a `JSESSIONID` cookie.

The application uses Spring Security form-based authentication and Tomcat HTTP sessions.

Because session state is maintained locally by Tomcat, requests routed to different replicas may not have access to the same authenticated session.

We therefore enabled ALB target-group stickiness:

```yaml
alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=86400
```

This configures an ALB-generated stickiness cookie with a duration of 86,400 seconds (24 hours).

Subsequent requests from a client presenting a valid stickiness cookie are generally directed to the same healthy target.

This reduces the likelihood of users being routed between Tomcat replicas during normal operation.

### 4. Accept the limitations of local HTTP sessions

ALB stickiness is not equivalent to distributed session management.

If a Tomcat Pod terminates, becomes unhealthy or is replaced during deployment, sessions stored only on that Pod may be lost.

The ALB can redirect subsequent requests to another healthy Pod, but the new Pod will not automatically have the previous Pod's session data.

For the current deployment, this behaviour is accepted.

## Alternatives Considered

### Dedicated liveness and readiness endpoints

Would provide better separation between application availability and dependency readiness.

Deferred because the necessary application changes are outside the immediate scope of the platform deployment.

### Using `/login` for health checks

Rejected after testing confirmed that `GET /login` returns HTTP 405.

### No session affinity

Would allow the ALB to distribute requests without attempting to preserve client-to-target affinity.

Not selected because the inherited application uses local HTTP sessions, making authentication behaviour potentially inconsistent across replicas.

### External session storage or session replication

Using shared session storage, such as Redis, or Tomcat session replication would allow sessions to survive routing changes and potentially Pod failures.

Not selected because it requires additional application integration and operational complexity.

## Consequences

### Benefits

- Health checks work with the existing application without Java modifications.
- Kubernetes and ALB use a consistent HTTP availability endpoint.
- The implementation remains simple to configure and troubleshoot.
- ALB stickiness supports the application's existing session-based authentication model.
- Application changes are avoided unless a clear operational requirement justifies them.

### Trade-offs and Limitations

- HTTP 200 from `/` does not guarantee downstream dependencies are healthy.
- The current liveness and readiness checks provide similar signals rather than distinct health semantics.
- Session stickiness does not preserve sessions when a Pod fails.
- A 24-hour stickiness cookie duration does not guarantee that a session remains valid for 24 hours.
- Application availability during dependency failures and rolling deployments still requires runtime testing.

## Future Improvements

If stronger production availability requirements are introduced:

1. Implement a lightweight liveness endpoint that does not depend on external services.
2. Implement readiness checks based on dependencies genuinely required to serve requests.
3. Ensure health endpoints are accessible without authentication but do not expose sensitive information.
4. Evaluate shared session storage if seamless failover and session continuity are required.
5. Test rolling deployments, Pod termination and dependency outages to validate actual user impact.

## Validation

The following behaviour has been verified locally:

- `GET /` returns HTTP 200 and serves the login page.
- `GET /login` returns HTTP 405.
- The application issues a `JSESSIONID` cookie.

The Kubernetes probes and ALB configuration have been defined in manifests.

Actual ALB target health, session affinity across multiple replicas and failure recovery remain to be tested after deployment to EKS.