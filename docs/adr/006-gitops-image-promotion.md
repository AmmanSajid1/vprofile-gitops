# ADR 006: Automated GitOps Image Promotion

**Status:** Accepted

## Context

The vProfile application and Flyway migration images are built and published to Amazon ECR through GitHub Actions.

Initially, deploying a new application version required manually updating image tags in the GitOps repository.

We wanted application releases to flow automatically from a successful CI build into the development environment while preserving Git as the source of truth.

The GitOps repository also uses branch protection, so automation should not bypass the normal pull request workflow.

## Decision

Use GitHub Actions to create automated image-update pull requests in the GitOps repository after successfully publishing container images.

### Authentication

A GitHub App provides authentication for the cross-repository workflow.

GitHub Actions generates a short-lived installation token rather than storing a long-lived personal access token.

This allows the workflow to authenticate to the GitOps repository without granting broad access through a personal account.

### Image versioning

Application and Flyway images are published to separate ECR repositories using immutable Git commit SHA tags.

Both images are associated with the same application source revision.

This makes deployments traceable to a specific commit and avoids relying on mutable tags such as `latest`.

### GitOps update workflow

After the CI pipeline successfully builds and publishes the images:

1. GitHub Actions authenticates to the GitOps repository using the GitHub App.
2. The workflow creates or updates a branch containing the image-tag changes.
3. The development Kustomize overlay is updated with the new application and Flyway image tags.
4. An automated pull request is opened against the protected `main` branch.
5. The pull request is reviewed and merged.
6. Argo CD detects the updated GitOps configuration.
7. Argo CD automatically synchronises the development environment.

Kustomize image transformations ensure the rendered manifests use the configured tags, even when the original resource manifests contain older image references.

### Deployment ordering

The existing Argo CD deployment process runs database bootstrap and Flyway migrations before synchronising the application workloads.

This allows application image updates and database schema changes to be coordinated through the same GitOps revision.

## Alternatives considered

### Direct deployment from GitHub Actions

GitHub Actions could update Kubernetes workloads directly using `kubectl` or Helm.

Rejected because this would introduce a second deployment controller and weaken the GitOps model.

### Direct pushes to the GitOps main branch

Simpler automation, but bypasses pull request review and branch protection.

Rejected to preserve an auditable change process.

### Personal access token

A PAT could authenticate the cross-repository workflow.

Rejected in favour of a GitHub App with scoped permissions and short-lived installation tokens.

### Mutable container image tags

Using tags such as `latest` would simplify image references but reduce deployment traceability.

Rejected in favour of immutable commit SHA tags.

## Consequences

### Benefits

- Automated promotion of successfully built images.
- Clear separation between CI and deployment reconciliation.
- Auditable deployment changes through pull requests.
- Branch protection remains enforced.
- Immutable images support traceability and rollback.
- Application and migration images remain aligned to the same source revision.

### Trade-offs and limitations

- A GitHub App and its credentials must be managed.
- Deployment depends on the availability of GitHub and Argo CD.
- Pull requests introduce a review step before deployment.
- CI must handle existing branches, duplicate pull requests and concurrent workflow runs.
- Automatic synchronisation is currently configured for development; promotion to other environments requires separate policies.

## Future improvements

- Add automated validation of GitOps pull requests.
- Introduce staging and production promotion workflows.
- Add deployment notifications and failure reporting.
- Implement automated smoke tests following successful Argo CD synchronisation.