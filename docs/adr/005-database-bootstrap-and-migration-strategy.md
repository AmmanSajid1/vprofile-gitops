# ADR 005: Database Bootstrap and Migration Strategy

**Status:** Accepted

## Context

The vProfile application uses Amazon RDS for MySQL. The database requires initial provisioning of the application schema and database users, followed by version-controlled schema migrations.

Previously, database setup depended on manual configuration. This was unsuitable for a GitOps deployment because application releases needed a repeatable way to prepare the database before starting the application.

We also wanted to avoid using the RDS master account for normal application operations or schema migrations.

## Decision

Use two separate Kubernetes Jobs, managed through Argo CD, for database bootstrap and schema migrations.

### Database bootstrap

The `vprofile-mysql-bootstrap` Job:

- Retrieves the RDS endpoint and master secret ARN from AWS Systems Manager Parameter Store.
- Retrieves the RDS master credentials and dedicated application/migration credentials from AWS Secrets Manager.
- Creates the `accounts` database if it does not exist.
- Creates or updates two MySQL users:
  - `vprofile_app`: application data access.
  - `vprofile_migrator`: application data access and schema modification.
- Uses a dedicated Kubernetes service account with AWS permissions provided through EKS Pod Identity.

The application user receives `SELECT`, `INSERT`, `UPDATE` and `DELETE` privileges. The migration user additionally receives `CREATE`, `ALTER`, `DROP`, `INDEX` and `REFERENCES` privileges.

The Job is designed to tolerate repeated execution by using `CREATE DATABASE IF NOT EXISTS`, `CREATE USER IF NOT EXISTS` and explicit privilege grants.

### Database migrations

A separate Flyway Job runs after database bootstrap.

Flyway migrations are packaged into a dedicated container image and published to Amazon ECR through the application CI pipeline.

The migration Job retrieves the RDS endpoint and migrator credentials from AWS, constructs a temporary Flyway configuration file, and executes `flyway migrate` over a TLS-required MySQL connection.

Flyway tracks applied migrations in the database, allowing schema changes to be version-controlled and applied incrementally.

### Deployment ordering

Argo CD sync hooks and waves enforce the sequence:

1. **Wave -2:** Bootstrap the database and configure users.
2. **Wave -1:** Execute Flyway migrations.
3. **Normal application sync:** Deploy application workloads.

Both Jobs use `BeforeHookCreation` and `HookSucceeded` cleanup policies.

### Credential handling

Credentials are not committed to Git.

The Jobs retrieve credentials using AWS APIs and place temporary files in memory-backed Kubernetes `emptyDir` volumes. The application and migration Jobs use different database identities.

## Alternatives considered

### Manual database provisioning

Simpler initially, but requires operator intervention and introduces configuration drift.

Rejected because database setup should be reproducible.

### Running migrations during application startup

Avoids a separate Job but couples application availability to migration execution and can create concurrency issues when multiple replicas start.

Rejected in favour of an independently controlled migration stage.

### Using the RDS master account for application and migrations

Reduces the number of database identities but grants unnecessary administrative permissions.

Rejected in favour of least-privilege database users.

## Consequences

### Benefits

- Repeatable database provisioning.
- Version-controlled schema migrations.
- Reduced application database privileges.
- Explicit ordering between bootstrap, migrations and application deployment.
- AWS-managed credential storage without plaintext secrets in Git.

### Trade-offs and limitations

- More Kubernetes Jobs, service accounts and IAM permissions to maintain.
- Bootstrap currently installs PyMySQL at runtime, introducing a dependency on package availability.
- Bootstrap uses a master database credential and therefore requires tightly scoped AWS permissions.
- Versioned Job names and Argo CD hook behaviour require care during future changes.
- Database bootstrap and migration failures can block application deployment.

## Future improvements

- Build a dedicated bootstrap image with pinned dependencies.
- Add automated verification of database privileges.
- Introduce migration rollback and recovery procedures.
- Improve visibility into Job failures and migration duration.