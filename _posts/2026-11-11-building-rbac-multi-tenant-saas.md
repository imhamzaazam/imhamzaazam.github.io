---
layout: post
title: "Building RBAC for a Multi-Tenant SaaS: Permissions, Roles, and Scopes"
date: 2026-11-11 06:00:00 +0500
tags: [backend, rbac, saas, software-architecture]
description: "How I'm approaching role-based access control for a multi-tenant freight-forwarding SaaS."
image: /assets/img/zero-trust.png
comments: false
---

![Zero Trust](/assets/img/zero-trust.png)

Authentication is usually the easy part. A user logs in, gets a token, and starts using the application.

Authorization is where things get interesting.

I'm building a multi-tenant freight-forwarding SaaS, and one of the things I've been thinking carefully about is how users should access different parts of the system.

At first, role-based access control (RBAC) sounds straightforward. Create a few roles, assign permissions, and you're done.

But real businesses aren't that simple.

A tenant might have multiple companies. Each company might have multiple branches. Someone working in one branch shouldn't automatically have access to everything happening in another.

And then there's the platform itself, which has its own users and responsibilities.

I wanted an authorization model that could handle these requirements without becoming unnecessarily complicated.

Here's the approach I'm currently taking.

## 1. Separating authentication from authorization

I'm using Keycloak for authentication and PostgreSQL as the source of truth for application permissions.

I don't want to build password management, login flows, token issuance, and identity management myself. That's what an identity provider is for.

Keycloak handles:

- User authentication
- Password management
- Login flows
- Access token issuance
- Identity information

My application handles:

- Users and tenant membership
- Roles and permissions
- Role assignments
- Company and branch access
- Authorization decisions

The distinction is intentional.

A valid access token tells me who the user is. It doesn't automatically mean they're allowed to perform every action in the application.

I want application permissions to remain under my control, rather than duplicating the entire permission model inside Keycloak.

## 2. Understanding the hierarchy

Before designing the tables, I needed to be clear about the structure of the application.

The hierarchy I'm working with is:

```text
Tenant
  └── Company
       └── Branch
```

A tenant can have multiple companies, and each company can have multiple branches.

This matters because access can be assigned at different levels.

For example:

- A user might have access to an entire company.
- Another user might only have access to one branch.
- A platform user might manage platform-level operations independently of tenant access.

I don't want to solve these requirements by creating a separate role for every possible combination.

That would become difficult to maintain very quickly.

Instead, I'm treating **permissions, roles, and scope as separate concepts**.

## 3. Permissions describe what a user can do

A permission should describe an action, not where that action is allowed.

For example:

```text
shipment:create
shipment:read
shipment:update
shipment:delete

invoice:view
invoice:create

user:create
user:read
user:update
```

These permission codes answer one question:

**What is the user allowed to do?**

They don't answer which company or branch the user can access.

I deliberately want to keep scope out of permission names.

I don't want permissions such as:

```text
shipment:read:company
shipment:read:branch
shipment:read:tenant
```

That mixes two different responsibilities.

The permission says what action is allowed. The assignment determines where that permission applies.

This keeps the permission catalog relatively small and easier to manage.

## 4. Roles group permissions

Creating individual permissions for every user would be difficult to manage.

That's where roles come in.

A role is simply a collection of permissions.

For example:

**Operations**

```text
shipment:create
shipment:read
shipment:update
```

**Accountant**

```text
invoice:view
invoice:create
```

**Sales**

```text
customer:create
customer:read
quotation:create
quotation:read
```

A user receives permissions through their assigned roles.

I'm also planning to support both system-defined roles and custom roles created by tenants.

System roles provide sensible defaults, while custom roles let businesses adapt the application to their own workflows.

Both should use the same permission catalog. There's no reason to introduce a completely separate permission system for custom roles.

## 5. Scope belongs to the role assignment

This is probably the most important decision in my current design.

A role tells us what a user can do. An assignment tells us where they can do it.

Consider this example:

Hamza has the Operations role.

Does that mean he can manage shipments across every company and branch?

Not necessarily.

He might only be responsible for the Dubai branch of a particular company.

Another user with exactly the same Operations role might have access to the entire company.

The role is identical. The scope is different.

My current assignment model looks something like this:

```sql
CREATE TABLE user_role (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    company_id UUID NULL REFERENCES company(id),
    branch_id UUID REFERENCES branch(id),
    user_id UUID NOT NULL REFERENCES user_account(id),
    role_id UUID NOT NULL REFERENCES role(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

The important part is `company_id` and `branch_id`.

- `branch_id IS NULL` means the assignment applies to the entire company.
- A specific `branch_id` means the assignment applies only to that branch.
- `company_id IS NULL` means the assignment applies to entire tenant and `tenant_id` should not be null.

Notice that this represents company-level and branch-level assignments. It does not represent tenant-wide access across multiple companies.

If I need tenant-wide assignments, I'll model that explicitly rather than treating a missing branch ID as tenant-wide access.

This separation gives me flexibility without creating unnecessary role variations.

## 6. The frontend selects context, but the backend authorizes access

In the frontend, users need a way to select the company and branch they're currently working in.

I'm thinking of handling this through request headers:

```http
x-company-id: <company-id>
x-branch-id: <branch-id>
```

For example, a user might belong to multiple companies and switch between them using a company selector.

The selected company and branch represent the user's current working context.

But there's an important distinction here:

**A frontend-selected context is not proof of authorization.**

A user can modify request headers. So the backend must validate that the selected company and branch belong to the tenant and that the user's assignments permit access to that context.

The frontend tells the backend where the user wants to work. The backend decides whether they're allowed to work there.

I prefer this approach because it keeps the frontend simple while keeping authorization decisions on the server.

## 7. Keeping NestJS authorization simple

I don't want every controller to contain its own authorization logic.

That would quickly lead to duplicated checks and inconsistent behavior.

Instead, I'm planning to use a small permission decorator:

```typescript
@RequirePermission('shipment:read')
@Get()
findAll() {
  return this.shipmentService.findAll();
}
```

The decorator declares what permission an endpoint requires.

A central authorization layer handles the actual decision:

1. Identify the authenticated user.
2. Resolve the user's role assignments.
3. Check whether the required permission is available.
4. Validate the selected company and branch against the assignment scope.
5. Allow or reject the request.

The decorator shouldn't need to know whether the user has company-wide or branch-specific access.

That's not its responsibility.

It only declares the required permission.

The authorization layer handles the rest.

## 8. Authorization must reach the database

Passing an authorization check doesn't mean the database can return everything.

The data access layer still needs to enforce the correct tenant, company, and branch boundaries.

For example, a branch-specific shipment query might look like this:

```sql
SELECT *
FROM shipment
WHERE tenant_id = $1
  AND company_id = $2
  AND branch_id = $3;
```

For a company-wide assignment, the branch condition would be handled differently because the user is allowed to access multiple branches within that company.

This is important because authorization shouldn't depend only on whether an endpoint is accessible.

The actual records returned by the database must also respect the user's scope.

I'm keeping these filters close to the data access layer so that they aren't accidentally forgotten in individual endpoints.

## 9. Separating platform and tenant access

The SaaS platform itself has users with responsibilities that are different from tenant users.

For example, platform users might handle:

- Platform ownership
- Finance
- Customer support

Tenant users, on the other hand, work inside their own business environment.

I don't want these two contexts mixed together.

My current direction is to keep platform and portal identities separate using distinct Keycloak realms:

```typescript
export const REALM = {
  PLATFORM: 'platform',
  PORTAL: 'portal',
} as const;
```

The application can then resolve the correct user context and authorization rules based on the realm.

Platform permissions and tenant permissions remain separate concerns.

This also avoids giving platform users tenant-level access just because they have a platform role.

## 10. What I'm trying to avoid

While designing this, there are a few things I deliberately don't want to introduce.

**Hardcoding roles everywhere**

Roles should be manageable data, not assumptions scattered throughout the codebase.

**Putting scope inside permission names**

Permissions should describe actions. Assignments should describe access boundaries.

**Trusting frontend context**

Headers help select the working context, but the backend must validate every selection.

**Duplicating authorization logic**

Controllers shouldn't each implement their own permission and scope checks.

**Overengineering too early**

I don't need a complicated policy engine or a distributed authorization service to get started. A clear permission catalog, role assignments, and centralized authorization checks should give me a solid foundation.

The goal is to keep the design flexible without building more than the product currently needs.

## Final thoughts

I'm still building this system, so these decisions will evolve as the application grows.

But I want to get the fundamentals right early.

The model I'm working towards is fairly simple:

| Concept | Responsibility |
|---|---|
| User | Who is making the request? |
| Permission | What can they do? |
| Role | Which permissions are grouped together? |
| Assignment | Which role does the user have? |
| Scope | Where does that role apply? |
| Context | Which company or branch are they currently working in? |

Keeping these concepts separate makes the authorization model easier to reason about.

And more importantly, it gives me room to support different business structures without turning every new requirement into another special case.

That's the direction I'm taking for RBAC in my freight-forwarding SaaS.

---

## References

- [NIST: Role Based Access Control](https://csrc.nist.gov/projects/role-based-access-control)
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [PostgreSQL Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
