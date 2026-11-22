
---
layout: post
title: "Building a Modern Freight Management System: From Architecture to CI/CD"
date: 2026-11-23 10:00:00 +0500
description: "An engineering journey through building a modern freight management platform, covering backend architecture, development tools, database engineering, authentication, testing, Testcontainers, and CI/CD."
tags: [software-engineering, backend, architecture, database, testing, testcontainers, docker, devops, freight]
image: /assets/img/zero-trust.png
categories: [Engineering]
---

# Introduction

Building a software platform from the ground up is more than writing code. It involves making architectural decisions, designing reliable data structures, establishing development workflows, implementing security, and creating a repeatable process for testing and deployment.

I am currently working on building a **freight management system**. Rather than focusing on product-specific functionality, I want to share the engineering practices, technologies, and decisions involved in developing a modern backend application.

This article covers the journey from architecture and project organization to database engineering, authentication, automated testing, and CI/CD.

The goal is to document the technical approach and lessons learned while keeping proprietary implementation details private.

## 1. Understanding the System Architecture

Architecture is one of the first important decisions when starting a backend project. The initial structure influences how easily the application can evolve as requirements become more complex.

There are several architectural approaches worth considering:

- **Monolithic architecture:** A single application containing the system's functionality.
- **Modular monolith:** A single deployable application organized into independent, well-defined modules.
- **Microservices architecture:** Multiple independently deployable services communicating through APIs or messaging.

For an early-stage product, a modular monolith can provide a practical balance between simplicity and maintainability.

It avoids the operational complexity of distributed services while allowing the codebase to grow through clearly separated modules.

### Separation of responsibilities

A maintainable backend should separate different responsibilities.

A typical layered approach includes:

- **Presentation layer:** Handles HTTP requests, validation, and responses.
- **Application layer:** Coordinates use cases and application workflows.
- **Domain layer:** Contains business rules and domain concepts.
- **Data access layer:** Handles database queries and persistence.
- **Infrastructure layer:** Integrates external services and technical dependencies.

Each layer should have a clear responsibility. This makes the application easier to understand, test, and maintain.

### Multi-tenancy considerations

Multi-tenancy introduces another architectural concern: data isolation.

In a multi-tenant application, multiple organizations share the same application infrastructure while their data remains logically separated.

The architecture should account for:

- Tenant identification.
- Data isolation.
- Access boundaries.
- Authorization.
- Consistent request context.
- Protection against cross-tenant access.

These concerns should be considered early rather than added as an afterthought.

## 2. Backend Project Structure

A good project structure makes it easier to navigate the codebase and understand where responsibilities belong.

Instead of organizing everything around technical file types, a modular structure can group related functionality together.

For example, a generic backend application might use a structure like:

```text
backend/
├── src/
│   ├── modules/
│   │   ├── users/
│   │   ├── organizations/
│   │   ├── operations/
│   │   ├── documents/
│   │   └── billing/
│   │
│   ├── common/
│   │   ├── guards/
│   │   ├── decorators/
│   │   ├── filters/
│   │   └── interceptors/
│   │
│   ├── infrastructure/
│   │   ├── database/
│   │   ├── identity/
│   │   ├── storage/
│   │   └── messaging/
│   │
│   ├── config/
│   └── main.ts/
│
├── database/
│   ├── migrations/
│   └── seeds/
│
├── test/
├── scripts/
└── package.json
```

This is an illustrative example, not a representation of any private project structure.

The important principle is separation of concerns.

A module should own its functionality and expose only what other modules need. Shared infrastructure should contain reusable technical capabilities rather than becoming a place for unrelated business logic.

The structure should remain understandable as the application grows. Adding unnecessary abstractions too early can make a project harder to maintain rather than easier.

## 3. Development Tools & Environment

Choosing a development stack is not just about selecting popular technologies. The tools should work together, support the application's requirements, and make development reproducible.

The technologies and tools relevant to this engineering approach include:

| Area | Technology / Tool |
|---|---|
| Runtime | [Node.js](https://nodejs.org/docs/latest/api/) |
| Language | [TypeScript](https://www.typescriptlang.org/docs/) |
| Backend framework | [NestJS](https://docs.nestjs.com/) |
| Database | [PostgreSQL](https://www.postgresql.org/docs/) |
| Containerization | [Docker](https://docs.docker.com/) |
| Local infrastructure | [Docker Compose](https://docs.docker.com/compose/) |
| Identity management | [Keycloak](https://www.keycloak.org/documentation) |
| API specification | [OpenAPI](https://spec.openapis.org/oas/latest.html) |
| Testing | [Jest](https://jestjs.io/docs/getting-started) |
| HTTP testing | [Supertest](https://github.com/ladjs/supertest) |
| Integration testing | [Testcontainers](https://testcontainers.com/) |
| Version control | Git and GitHub |
| CI/CD | [GitHub Actions](https://docs.github.com/en/actions) |

### Local development environment

A consistent local environment helps reduce the difference between individual development machines and automated environments.

Docker is useful for running supporting services such as databases, caches, and other infrastructure dependencies.

Docker Compose can define and start multiple services together, making it convenient to manage a local development environment.

The objective is to make the development environment easy to reproduce without requiring every developer to configure infrastructure manually.

### Configuration management

Application configuration should be separated from application code.

Environment variables are commonly used for settings such as:

- Database connection details.
- External service endpoints.
- Authentication configuration.
- Application environment.
- Logging settings.

Sensitive values should never be committed to source control. Development, testing, staging, and production environments should use their own configurations.

## 4. Database Engineering

Database design is one of the most important parts of building a reliable backend.

A database is not simply a place to store application data. It is responsible for maintaining relationships, enforcing constraints, and preserving data integrity.

For relational applications, PostgreSQL provides a strong foundation.

### Database-first thinking

A database-first approach means carefully considering the data model and its relationships before implementing application functionality.

Important considerations include:

- Primary keys and foreign keys.
- Unique constraints.
- Nullability.
- Data types.
- Indexes.
- Relationships.
- Transactions.
- Data integrity.

Database constraints should enforce important rules wherever appropriate rather than relying entirely on application-level validation.

### Normalization and relationships

Database normalization helps reduce unnecessary duplication and maintain consistency.

Tables should represent clear concepts, and relationships should be defined explicitly.

However, normalization should be applied pragmatically. The objective is not to create as many tables as possible, but to build a model that is understandable, consistent, and maintainable.

### Transactions

Transactions ensure that a group of related database operations succeeds or fails together.

For example, if an operation requires several database changes, those changes should not leave the database in a partially completed state.

Understanding transaction boundaries is essential for maintaining data consistency.

### Indexing and performance

Indexes improve query performance when designed around actual access patterns.

However, indexes also introduce storage and write overhead. Adding indexes without understanding query behavior can negatively affect performance.

Useful practices include:

- Identifying frequently executed queries.
- Reviewing query execution plans.
- Indexing appropriate filtering and joining columns.
- Avoiding unnecessary indexes.
- Monitoring slow queries.

PostgreSQL provides tools such as `EXPLAIN` and `EXPLAIN ANALYZE` to help understand query execution.

Official documentation: [PostgreSQL Documentation](https://www.postgresql.org/docs/current/).

## 5. Database Migrations

As an application evolves, its database schema changes.

New tables, columns, constraints, indexes, and relationships are introduced over time. Managing these changes manually becomes difficult, especially when multiple environments are involved.

Database migrations provide a controlled and versioned way to manage schema changes.

### Why migrations matter

Migrations help ensure that every environment follows a consistent database structure.

Instead of manually modifying databases, schema changes are recorded as version-controlled files.

A generic migration directory might look like:

```text
database/
└── migrations/
    ├── 001_initial_schema.sql
    ├── 002_create_users.sql
    ├── 003_add_indexes.sql
    └── 004_add_constraints.sql
```

Each migration represents a specific schema change.

### Migration best practices

A reliable migration process should follow a few principles:

1. Keep migrations version-controlled.
2. Apply migrations in a predictable order.
3. Avoid modifying migrations that have already been applied.
4. Test migrations against a real database engine.
5. Keep schema changes small and understandable.
6. Consider existing data before changing columns or constraints.
7. Have a recovery strategy for failed deployments.

### Safe schema changes

Database changes can become risky when an application is already running.

For example, renaming or removing a column immediately can break older application versions that still depend on it.

An approach commonly used for safer deployments is the **expand-and-contract pattern**:

1. Introduce the new schema without immediately removing the old one.
2. Update application code to support the new structure.
3. Migrate existing data if necessary.
4. Confirm that the new implementation works.
5. Remove the old structure in a later deployment.

This approach helps maintain compatibility during application updates.

## 6. Seed Data Management Across Environments

Migrations and seed data serve different purposes.

- **Migrations** define and evolve the database structure.
- **Seed data** inserts initial or supporting records into the database.

Separating these responsibilities makes database management easier.

Different environments also require different types of seed data.

### Development environment

Local development may require sample organizations, users, and records to make testing and development easier.

The data should be predictable and safe to recreate.

### Testing environment

Test data should be controlled and deterministic.

Tests should not depend on manually created records or data left behind by previous test executions.

Each test should have a clear understanding of the data it needs.

### Staging environment

Staging should resemble production as closely as practical.

Synthetic or properly anonymized data can help validate application behavior without exposing real customer information.

### Production environment

Production seed data should be limited to approved initial or reference records.

It should not contain development examples or test accounts.

### Keep seed responsibilities separate

A useful approach is to distinguish:

- Reference data.
- Development sample data.
- Test fixtures.
- Staging data.
- Production bootstrap data.

This prevents accidental mixing of test data with production records.

Seed scripts should also be designed carefully so that running them more than once does not unintentionally create duplicate records.

## 7. Authentication & Identity Management

Authentication and authorization are fundamental parts of a secure application.

Although these concepts are related, they solve different problems.

- **Authentication:** Determines who the user is.
- **Authorization:** Determines what the user is allowed to do.

Separating these responsibilities makes security easier to reason about.

### Identity management

An identity provider can manage authentication-related responsibilities such as:

- User login.
- Password management.
- Session management.
- Single sign-on.
- Multi-factor authentication.
- Token issuance.
- Identity federation.

[Keycloak](https://www.keycloak.org/documentation) is an open-source identity and access management solution that supports standard authentication protocols.

It supports OAuth 2.0 and OpenID Connect, which are commonly used for modern application authentication.

Useful references:

- [OAuth 2.0](https://oauth.net/2/)
- [OpenID Connect](https://openid.net/developers/how-connect-works/)

### Token validation

After authentication, the client receives a token that can be presented to the backend.

The backend should validate the token rather than blindly trusting its contents.

Important checks include:

- Token signature.
- Issuer.
- Audience.
- Expiration.
- Relevant claims.

A valid token establishes an authenticated identity, but it does not automatically grant access to every resource.

### Authorization

Application authorization should be designed independently from authentication.

A backend may need to evaluate:

- User permissions.
- Organization membership.
- Role assignments.
- Resource ownership.
- Access scope.
- Tenant boundaries.

These checks should be enforced on the server side.

Frontend visibility rules can improve the user experience, but they should never be treated as a security boundary.

### Security configuration

Identity provider configuration should be environment-specific.

Secrets, client credentials, signing keys, and other sensitive configuration must be protected and managed outside source control.

## 8. Test Strategy & Test Case Development

Testing is not something that should be added only when an application becomes large.

A good test strategy helps catch problems early and gives developers confidence when changing existing functionality.

Different testing levels serve different purposes.

### Unit testing

Unit tests validate small pieces of application logic in isolation.

They are useful for testing:

- Business rules.
- Validation logic.
- Utility functions.
- Calculations.
- Individual service behavior.

Unit tests should generally be fast and independent of external infrastructure.

### Integration testing

Integration tests verify that different parts of the application work together.

For backend applications, this often includes testing interactions between:

- Application services.
- Database access code.
- PostgreSQL.
- External service adapters.

Integration tests are especially useful for catching issues that unit tests cannot detect.

### API testing

API tests validate HTTP behavior.

They can cover:

- Request validation.
- Response structure.
- HTTP status codes.
- Authentication.
- Authorization.
- Error handling.
- Pagination.
- Filtering.

[Supertest](https://github.com/ladjs/supertest) is a useful library for testing HTTP endpoints in Node.js applications.

### End-to-end testing

End-to-end tests validate complete workflows from the perspective of an application client.

They help verify that multiple components work together as expected.

However, end-to-end tests can be slower and more expensive to maintain, so they should focus on important application flows.

### Test case development

A good test case should consider more than the successful scenario.

For an API endpoint, useful scenarios include:

| Scenario | What to verify |
|---|---|
| Valid request | Expected successful response |
| Invalid input | Validation errors |
| Missing fields | Required-field validation |
| Boundary values | Limits and edge cases |
| Duplicate records | Uniqueness behavior |
| Unauthenticated request | Authentication enforcement |
| Unauthorized request | Permission enforcement |
| Missing resource | Correct not-found response |
| Invalid relationships | Referential integrity |
| Database failure | Error handling |
| Failed transaction | Rollback behavior |
| Cross-tenant access | Tenant isolation |

Testing should cover both expected behavior and failure conditions.

### Testcontainers: Testing Against Real Dependencies

One particularly useful tool for backend integration testing is [Testcontainers](https://node.testcontainers.org/).

Testcontainers allows tests to start disposable infrastructure dependencies using containers.

Instead of relying entirely on mocked databases, integration tests can use an actual PostgreSQL instance.

A typical test lifecycle looks like this:

1. Start a PostgreSQL container.
2. Wait until the database is ready.
3. Connect the application to the database.
4. Apply database migrations.
5. Insert the required test data.
6. Execute the tests.
7. Clean up the container.

This approach makes integration tests more realistic.

For example, testing against a real PostgreSQL instance can catch issues involving:

- SQL syntax.
- Foreign key constraints.
- Unique constraints.
- Transactions.
- Database-specific functions.
- Migration compatibility.
- Data type behavior.

Mocks still have their place, especially in unit testing. However, they cannot fully reproduce the behavior of a real database.

Testcontainers helps bridge that gap.

It is particularly useful when the application depends on external infrastructure such as PostgreSQL, Redis, or message brokers.

## 9. Continuous Integration (CI)

Continuous Integration is the practice of automatically validating code changes as they are introduced into a shared repository.

The objective is to identify problems early and keep the main codebase in a healthy state.

A typical backend CI pipeline can include:

1. Check out the repository.
2. Install dependencies.
3. Run formatting checks.
4. Run linting.
5. Run TypeScript type checks.
6. Start required infrastructure.
7. Apply database migrations.
8. Run unit tests.
9. Run integration tests.
10. Run API or end-to-end tests.
11. Generate test coverage.
12. Build the application.

### Why CI matters

Without automated checks, developers may unintentionally introduce:

- Broken builds.
- Type errors.
- Failed migrations.
- Regressions.
- Security issues.
- Incompatible changes.

Automated checks help catch these problems before changes reach production.

### GitHub Actions

[GitHub Actions](https://docs.github.com/en/actions) provides a way to automate workflows directly within GitHub repositories.

It can run tests, build applications, validate code, and trigger deployment workflows.

For Testcontainers-based integration tests, the CI runner must provide a working Docker-compatible environment.

A reliable CI pipeline should also use isolated test data and clean up temporary infrastructure after execution.

## 10. Continuous Delivery & Deployment (CD)

Continuous Delivery and Continuous Deployment are related practices, but they are not identical.

- **Continuous Delivery:** Changes are automatically built and validated, then made ready for deployment. Production release may require approval.
- **Continuous Deployment:** Changes that pass the required checks are automatically released to production.

A typical deployment pipeline can include:

1. Build the application.
2. Create a versioned artifact or container image.
3. Deploy to a staging environment.
4. Run smoke tests.
5. Validate application health.
6. Obtain approval when required.
7. Deploy to production.
8. Monitor the application.

### Environment promotion

Applications commonly have separate environments:

- Development.
- Testing.
- Staging.
- Production.

The goal is to promote validated application artifacts between environments rather than rebuilding different versions for each environment.

Configuration and secrets should remain environment-specific.

### Database deployment considerations

Deploying application code and database changes requires careful coordination.

A database migration can affect application compatibility, especially when multiple application versions may be running during deployment.

Safer schema changes, migration validation, and a recovery strategy are important parts of deployment planning.

### Monitoring and recovery

A deployment is not complete simply because the application starts successfully.

It is also important to monitor:

- Application health.
- Error rates.
- Response times.
- Database connectivity.
- Resource usage.
- Background job failures.

Backups, recovery procedures, and rollback strategies should be considered before production deployment.

Useful documentation:

- [Docker Build](https://docs.docker.com/build/)
- [Docker Compose](https://docs.docker.com/compose/)
- [GitHub Actions](https://docs.github.com/en/actions)

## 11. Putting Everything Together

Architecture, database design, authentication, testing, and deployment should not be treated as isolated activities.

They form a connected engineering workflow.

A repeatable development process might look like this:

1. Understand the requirement.
2. Define the expected behavior.
3. Consider the architectural impact.
4. Plan the required data changes.
5. Implement the application functionality.
6. Apply the appropriate authorization rules.
7. Write unit and integration tests.
8. Validate the API behavior.
9. Run automated checks.
10. Review and merge the changes.
11. Deploy to staging.
12. Validate the deployment.
13. Release to production when ready.

The process should be consistent, but not unnecessarily complicated.

A few principles are especially important:

**Keep architecture understandable.**

Choose an approach that solves current problems without introducing unnecessary complexity.

**Treat data integrity seriously.**

Use database constraints, transactions, and carefully designed relationships.

**Separate authentication from authorization.**

Knowing who a user is does not automatically determine what they can access.

**Test against realistic dependencies.**

Mocks are useful, but important integration behavior should also be tested against real infrastructure.

**Automate repetitive work.**

Formatting, linting, testing, building, and deployment validation should not depend entirely on manual execution.

**Build for maintainability.**

Code should be understandable by someone who did not originally write it.

**Improve incrementally.**

Architecture and engineering practices should evolve as requirements and experience grow.

## References & Further Reading

The following official documentation and resources are useful for exploring the technologies and practices discussed in this article.

### Backend Development

- [Node.js Documentation](https://nodejs.org/docs/latest/api/)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [NestJS Documentation](https://docs.nestjs.com/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)

### Database Engineering

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [PostgreSQL SQL Commands](https://www.postgresql.org/docs/current/sql.html)

### Infrastructure & Identity

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [OAuth 2.0](https://oauth.net/2/)
- [OpenID Connect](https://openid.net/developers/how-connect-works/)

### Testing

- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Supertest](https://github.com/ladjs/supertest)
- [Testcontainers for Node.js](https://node.testcontainers.org/)
- [Testcontainers](https://testcontainers.com/)

### CI/CD

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Docker Build Documentation](https://docs.docker.com/build/)

## Conclusion

Building a modern freight management system is an ongoing engineering process that involves much more than implementing individual features.

Architecture determines how the application is organized. Database engineering protects data integrity. Authentication and authorization establish security boundaries. Automated testing helps maintain confidence in changes. CI/CD makes delivery more predictable and repeatable.

Tools such as Docker, PostgreSQL, Keycloak, Jest, and Testcontainers can help establish a solid engineering foundation, but tools alone do not guarantee a good system.

The real value comes from using them thoughtfully, keeping the design understandable, and improving the development process over time.

This is an ongoing journey, and there is always something new to learn, improve, and refine.
