# Troubleshooting: Spring Security and ALB Health Checks

**Date:** 2026-10-15  
**Component:** vProfile Java Application / Kubernetes / AWS ALB  
**Status:** Resolved locally — AWS deployment validation pending

## Problem

While configuring Kubernetes health probes and AWS Application Load Balancer (ALB) health checks, we needed to identify an endpoint that would reliably return HTTP 200 without authentication.

The inherited vProfile application uses Spring MVC, Spring Security and Apache Tomcat.

The application does not expose dedicated health-check endpoints, so we investigated whether existing routes could be used without modifying the Java code.

An incorrect health-check path could cause Kubernetes to restart otherwise healthy containers or the ALB to mark application targets as unhealthy.

## Investigation

### 1. Reviewed the application's authentication configuration

The application uses Spring Security form-based authentication.

Its configuration includes:

- A custom login page at `/login`.
- Form submission to `/login`.
- Protected routes such as `/` and `/welcome`.
- Session-based authentication.

Initially, `/login` appeared to be a suitable health-check candidate because it was associated with the login page.

However, this needed to be verified against the running application.

### 2. Tested the login endpoint

Executed:

```bash
curl -i http://localhost:8080/login
```

Result:

```http
HTTP/1.1 405
Allow: POST
```

Tomcat returned:

```text
HTTP Status 405 – Method Not Allowed
Message: Method 'GET' is not supported.
```

This showed that `/login` did not support the GET request used by the proposed health check.

The endpoint therefore could not be used with an expected success status of HTTP 200.

### 3. Tested the root endpoint

Executed:

```bash
curl -i http://localhost:8080/
```

Result:

```http
HTTP/1.1 200
Set-Cookie: JSESSIONID=...
Content-Type: text/html;charset=ISO-8859-1
```

The response contained the application's HTML login page.

This confirmed that the root endpoint could return HTTP 200 without an authenticated session.

The `JSESSIONID` cookie also provided evidence that the application was using Tomcat HTTP sessions.

### 4. Reviewed the implications for Kubernetes and ALB

Kubernetes HTTP probes treat successful HTTP status codes in the 200–399 range as healthy.

The ALB health check was configured to expect HTTP 200.

Using `/login` with an expected status of 200 would therefore fail.

Using `/` provided a verified endpoint that matched the required response.

However, the login page could still be served while downstream services such as MySQL were unavailable.

The endpoint therefore provided basic HTTP availability rather than complete application readiness.

## Resolution

Retained the root endpoint for the Kubernetes startup, readiness and liveness probes.

Configured the ALB Ingress with:

```yaml
alb.ingress.kubernetes.io/healthcheck-path: /
alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
alb.ingress.kubernetes.io/success-codes: '200'
```

The same configuration was applied to the development, staging and production Ingress manifests.

We deliberately avoided modifying the inherited Java application to introduce new health endpoints at this stage.

## Verification

The following checks were completed locally:

| Test | Result |
|---|---|
| `GET /login` | HTTP 405 |
| `GET /` | HTTP 200 |
| Root endpoint returns login page | Confirmed |
| Root response contains `JSESSIONID` | Confirmed |
| Ingress configured to check `/` | Completed |

Live ALB target health and Kubernetes probe behaviour still need to be verified after deployment to EKS.

## Lessons Learned

1. **Do not assume an endpoint is suitable for health checks based on its name.** Verify the HTTP method and response status against the running application.

2. **HTTP availability and application readiness are different.** A successful login-page response does not prove that critical downstream dependencies are functioning.

3. **Authentication can affect infrastructure health checks.** Spring Security routing and access rules must be considered when selecting probe endpoints.

4. **Health-check configuration should reflect actual application behaviour.** Changing ALB success codes to accept an error response would hide the underlying issue rather than establish a meaningful health signal.

5. **Avoid unnecessary changes to inherited application code.** Where an existing endpoint meets the immediate operational requirement, its limitations can be documented and improvements planned separately.

## Future Improvements

For stricter production requirements, work with the application development team to introduce dedicated liveness and readiness endpoints.

Readiness checks should reflect dependencies genuinely required to serve traffic, while liveness checks should avoid triggering unnecessary restarts during temporary downstream outages.

See also: [ADR 004 — Application Health Checks and Session Affinity](../adr/004-healthchecks-and-session-affinity.md).