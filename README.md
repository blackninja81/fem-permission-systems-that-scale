# Permission Systems That Scale - Comprehensive Guide

## Table of Contents

1. [Introduction](#introduction)
2. [Authentication vs Authorization](#authentication-vs-authorization)
3. [Goals of a Permission System](#goals-of-a-permission-system)
4. [Common Permission Mistakes](#common-permission-mistakes)
5. [Clean Architecture: Services Layer](#clean-architecture-services-layer)
6. [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
7. [RBAC Limitations](#rbac-limitations)
8. [Attribute-Based Access Control (ABAC)](#attribute-based-access-control-abac)
9. [Advanced ABAC Features](#advanced-abac-features)
10. [ABAC Limitations](#abac-limitations)
11. [Permission Libraries](#permission-libraries)
12. [CASL Implementation](#casl-implementation)
13. [Choosing the Right Permission Model](#choosing-the-right-permission-model)
14. [Decision Flowchart](#decision-flowchart)

---

## Introduction

### What You'll Learn

This course teaches how to build robust, maintainable permission systems for real-world web applications. By the end, you'll understand the trade-offs between different authorization approaches and know how to implement them from scratch.

**Key Topics Covered:**
- How to implement **Role-Based Access Control (RBAC)** with clean, type-safe code
- How to build **Attribute-Based Access Control (ABAC)** for complex, dynamic permissions
- How popular libraries like **CASL** and **Casbin** handle permissions
- How to structure permission logic so it's easy to maintain and extend

### The Progressive Learning Approach

The course uses a document management application that evolves through multiple iterations:

1. **Basic Permissions** → Inline checks with simple role comparisons
2. **Structured RBAC** → Centralized role-permission mappings
3. **Full ABAC** → Dynamic rules based on user attributes, resource properties, and context
4. **Library Integration** → Migrating to CASL for production-ready permissions

Each upgrade demonstrates the strengths and limitations of different approaches, helping you choose the right tool for your specific needs.

---

## Authentication vs Authorization

### The Critical Distinction

Before diving into permission systems, it's essential to understand two concepts that are often confused:

| Concept            | Question It Answers | Example                             |
| ------------------ | ------------------- | ----------------------------------- |
| **Authentication** | "Who are you?"      | Logging in with email/password      |
| **Authorization**  | "What can you do?"  | Can this user delete this document? |

### The Building Security Analogy

Think of it like entering a secure building:

- **Authentication** is showing your ID badge at the entrance, proving you are who you claim to be
- **Authorization** is the access levels on your badge, determining which floors and rooms you can enter

### Why Authorization Is More Complex

Authorization is generally more complex than authentication for several reasons:

#### 1. Deeply Tied to Business Logic

Every application has unique rules that can't be outsourced to a generic provider:

```text
"Editors can modify documents, but only in their department"
"Authors can publish their own drafts, but need approval for others"
"Admins can do everything, except delete locked documents"
```

#### 2. Touches Every Part of Your Codebase

Permission checks appear everywhere:

- **API routes**: Can this user call this endpoint?
- **Database queries**: Which records should be returned?
- **UI components**: Should this button be visible?
- **Business logic**: Is this state transition allowed?

#### 3. Evolves Constantly

Business requirements change frequently:

```text
"We need to add a new role for external contractors"
"Documents should be locked after 30 days"
"Department heads can now approve cross-department access"
```

A rigid permission system becomes a bottleneck for feature development.

### The Cost of Getting It Wrong

Poor authorization leads to real problems:

| Issue                | Consequence                                                  |
| -------------------- | ------------------------------------------------------------ |
| **Over-permissive**  | Data breaches, unauthorized actions, compliance violations   |
| **Under-permissive** | Frustrated users, support tickets, lost productivity         |
| **Inconsistent**     | Security holes where checks are missing, confusing UX        |
| **Unmaintainable**   | New features require touching dozens of files, bugs multiply |

### Workshop Focus

This course is **100% about authorization**. We assume you already have authentication handled. Users can log in and you know who they are. The challenge is: given a known user, how do we efficiently and maintainably determine what they're allowed to do?

---

## Goals of a Permission System

A well-designed permission system should achieve the following goals:

### 1. Prevent Unauthorized Access

**Impossible to access unauthorized data**: All data access should go through auth checks before any data is returned or modified. This is the most fundamental security requirement.

### 2. Single Source of Truth

Permission logic should be centralized in one place, not scattered throughout your codebase. This makes it easier to:
- Audit permissions
- Update rules
- Reason about security

### 3. Automatic Enforcement

Authorization should be automatically applied whenever data is accessed or mutated. Developers shouldn't need to remember to add permission checks manually. They should happen by default.

### 4. Consistent Across Frontend and Backend

The same permission rules should produce the same results whether evaluated on the client or server. This prevents confusing UX where buttons appear enabled but actions fail.

### 5. Never Trust the Client

All permission checks must be enforced on the server. Client-side checks are only for UX (hiding buttons, disabling fields) and never for actual security.

### 6. Fail Closed

When in doubt, deny access. If something goes wrong or a check is missing, the default should be to block access rather than allow it.

---

## Common Permission Mistakes

When building permission systems, developers often make these critical mistakes:

### Problem 1: Scattered Permission Checks

Permission logic duplicated across files creates inevitable bugs:

```typescript
// In one file
if (user.role !== "admin" && user.role !== "author") {
  // deny access
}

// In another file - slightly different
if (user.role === "viewer" || user.role === "editor") {
  // deny access
}
```

When the same logic exists in multiple places, a developer might update it in one place but forget others, leading to inconsistent behavior that's incredibly hard to debug.

**Real-world example**: A common bug is allowing a user to access an "Edit" page but blocking them from actually saving changes. Or worse, hiding the UI button but not blocking the API endpoint, so a savvy user can still perform unauthorized actions via the browser console.

### Problem 2: Missing Permission Checks

This is the most dangerous mistake. Applications often have pages that can be accessed by **anyone who knows the URL**, even if the UI doesn't show a link to that page.

For example, if you're logged in as a `viewer` and manually navigate to:

```text
/projects/[projectId]/documents/[documentId]
```

You might be able to access that page even if you normally wouldn't be able to see that document.

**The golden rule**: Never trust the client. If a permission matters, check it on the server.

### Problem 3: Inconsistent Permission Logic

Complex permission checks that are hard to read lead to errors:

```typescript
// What does this actually check?
if (user.role !== "admin" && user.role !== "author") {
  // deny access
}

// Same logic, different implementation
if (user.role === "viewer" || user.role === "editor") {
  // deny access
}
```

The problems with this approach:
- **Duplicated** across multiple files
- **Error-prone** (easy to miss a condition)
- **Hard to read** (what does this actually mean?)
- **Inconsistent** (some places might check slightly differently)

### Why This Matters

These issues compound over time:

1. **Security vulnerabilities** - Missing checks = unauthorized access
2. **Maintenance nightmare** - Changing permission logic requires updating many files
3. **Testing difficulty** - Hard to verify all permission paths work correctly
4. **Onboarding friction** - New developers don't know where all the checks live

---

## Clean Architecture: Services Layer

### The Problem

Even after fixing basic permission bugs, authorization logic often lives in multiple places:

```text
📁 src/
├── 📁 actions/
│   ├── documents.ts    ← Auth checks here
│   └── projects.ts     ← Auth checks here
├── 📁 dal/
│   └── documents.ts    ← And here
└── 📁 app/
    └── (pages)/
        ├── documents/  ← And here too!
        └── projects/
```

This creates problems:
1. **Duplication** - Same checks written multiple ways
2. **Inconsistency** - Easy to forget a check somewhere
3. **Difficult testing** - Auth logic tied to framework code
4. **Performance** - Potentially extra database queries

### The Solution: Services Layer

Create a dedicated layer for business logic which includes authorization checks:

```text
📁 src/
├── 📁 services/        ← NEW: All auth lives here
│   ├── documents.ts
│   └── projects.ts
├── 📁 actions/         ← Just calls services and handles errors
├── 📁 dal/             ← Just database access
└── 📁 app/             ← Just UI concerns
```

### A Basic Service Implementation

Each function in the service layer should take into account all authorization checks and only return data the user has access to:

```typescript
export async function createDocumentService(
  projectId: string,
  data: DocumentFormValues,
) {
  const user = await getCurrentUser()
  if (user == null) throw new Error("Unauthenticated")

  // Step 1: Authorization - can they perform this action?
  if (user.role !== "admin" && user.role !== "author") {
    throw new AuthorizationError()
  }

  // Step 2: Validation - is the data valid?
  const result = documentSchema.safeParse(data)
  if (!result.success) throw new Error("Invalid data")

  // Step 3: Execute - perform the actual operation
  return await createDocument({
    ...result.data,
    creatorId: user.id,
    lastEditedById: user.id,
    projectId,
  })
}
```

The action becomes a simple wrapper:

```typescript
export async function createDocumentAction(
  projectId: string,
  data: DocumentFormValues,
) {
  const [error, document] = await tryFn(() =>
    createDocumentService(projectId, data),
  )

  if (error) return error

  redirect(`/projects/${projectId}/documents/${document.id}`)
}
```

Notice what's **not** in the action:
- No `if (!canUpdate)` checks
- No field validation
- No permission imports

All that complexity lives in the service.

### Refactoring Page Components

Pages delegate to services and no longer require direct permission checks:

```typescript
export default async function ProjectDocumentsPage({
  params,
}: PageProps<"/projects/[projectId]">) {
  const { projectId } = await params

  // Returns null if not found or not authorized
  const project = await getProjectByIdService(projectId)
  if (project == null) return notFound()

  // Only fetch documents the user is authorized to see
  const documents = await getProjectDocumentsService(projectId)

  return (
    // UI
  )
}
```

### Architecture Benefits

#### 1. Single Responsibility

| Layer        | Responsibility                    |
| ------------ | --------------------------------- |
| **Pages**    | Routing, layout, calling services |
| **Actions**  | Redirection, error handling       |
| **Services** | Business logic, authorization     |
| **DAL**      | Pure database operations          |

This is important because:
- The data access layer can get data without worrying about authorization
- Many times you need all records regardless of the current user's permissions
- Each layer has a clear, single purpose

#### 2. Testability

Since the data access layer is no longer tied to the user or permissions, you can easily test it in isolation. It's also easier to test authorization logic since it mostly lives in one place.

#### 3. Security by Default

As long as all calls go through the service layer, it's impossible to accidentally bypass authorization checks or incorrectly code an authorization check since it's all handled automatically by the services layer.

---

## Role-Based Access Control (RBAC)

### The Core Concept

**Role-Based Access Control (RBAC)** is the most common permission system you'll encounter. If you've ever assigned a user to an "admin" or "editor" role, you've used a version of RBAC.

RBAC is built on a simple idea: **permissions are assigned to roles, and roles are assigned to users**.

```text
User → Role → Permissions
```

### The RBAC Model

A typical RBAC implementation has three components:

| Component       | Description                            | Example                          |
| --------------- | -------------------------------------- | -------------------------------- |
| **Users**       | The people using your system           | Alice, Bob                       |
| **Roles**       | Named groups of permissions            | admin, editor, viewer            |
| **Permissions** | Specific actions that can be performed | document:create, document:delete |

### Why RBAC Works

#### 1. Centralized Permission Logic

Instead of hunting through dozens of files to find every `if (user.role === "admin")` check, all your permission logic lives in one place.

#### 2. Single Point of Change

When business requirements change, you update the role definition once. Compare this to finding and updating every scattered permission check.

#### 3. Cleaner, More Readable Code

Your application code becomes about _what_ you're checking, not _how_:

```typescript
// Ad-hoc: What does this even mean?
if (user.role === "admin" || user.role === "editor")

// RBAC: Clear intent
if (can(user, "document:update"))
```

#### 4. Reduced Bugs

Ad-hoc checks are error-prone. Forget one check? Security hole. Copy-paste the wrong condition? Security hole. With RBAC, the logic is defined once and reused everywhere.

### RBAC Implementation

A basic RBAC implementation looks like this:

```typescript
// Define your permissions
type Permission =
  | "document:create"
  | "document:read"
  | "document:update"
  | "document:delete"
  | "project:create"
  | "project:read"
  | "project:update"
  | "project:delete"

// Map roles to permissions
const permissionsByRole: Record<UserRole, Permission[]> = {
  admin: [
    "document:create",
    "document:read",
    "document:update",
    "document:delete",
    "project:create",
    "project:read",
    "project:update",
    "project:delete",
  ],
  editor: ["document:read", "document:update"],
  viewer: ["document:read"],
}

// Check if a user can perform an action
function can(user: { role: UserRole }, permission: Permission) {
  return permissionsByRole[user.role].includes(permission)
}
```

Usage is clean and readable:

```typescript
if (can(user, "document:create")) {
  // Show create button
}

if (!can(user, "project:delete")) {
  throw new AuthorizationError()
}
```

### What RBAC Provides

| Benefit                    | Description                                           |
| -------------------------- | ----------------------------------------------------- |
| **Single source of truth** | All permissions defined in one place                  |
| **Readable code**          | `can(user, "document:create")` is self-documenting    |
| **Type safety**            | TypeScript ensures we only use valid permission names |
| **Easy updates**           | Change a role's permissions in one file               |

---

## RBAC Limitations

### Where RBAC Excels

RBAC is the right choice when your permissions are:

| Characteristic                  | Example                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| **Role-centric**                | "Admins can delete, viewers can read"                          |
| **Coarse-grained**              | A few roles cover all use cases                                |
| **Limited Attribute Awareness** | Permissions depend on few or no resource attributes or context |

### Where RBAC Struggles

RBAC breaks down when permissions depend on:

#### 1. Attributes Other Than User Role

As soon as you need to create multiple permutations of permissions for different attributes (e.g., ownership, document status), RBAC starts to struggle.

**Example requirement:**
> "Authors can edit documents **they created**, that are **not locked**, in **draft status**"

This permission depends on:
- The **user's role** (must be author)
- The **user's ID** (must be the creator)
- The **document's isLocked** attribute
- The **document's status** attribute

Pure RBAC struggles to express this.

#### 2. Environmental Factors

RBAC cannot take into consideration environmental factors such as time of day, location, or other contextual information. For example, a user may only be able to push code to production during certain hours or from specific IP addresses.

### The Symptom: Permission Explosion

When RBAC is the wrong fit, you'll notice your permission list growing uncontrollably:

```typescript
// Started simple
type Permission = "document:read" | "document:update" | "document:delete"

// Then requirements came in...
type Permission =
  | "document:read:all"
  | "document:read:own"
  | "document:read:non-draft"
  | "document:read:published"
  | "document:update:all"
  | "document:update:unlocked"
  | "document:update:own"
  | "document:update:own-unlocked"
  | "document:update:own-unlocked-draft"
  | "document:delete:all"
  | "document:delete:own"
// ... and it keeps growing
```

### The Hybrid Trap

When teams hit RBAC limits, they often create hybrids:

```typescript
// "RBAC" that's actually policy functions
export function canUpdateDocument(user, document) {
  if (user.role === "admin") return !document.isLocked
  if (user.role === "editor") return !document.isLocked
  if (user.role === "author") {
    return (
      document.creatorId === user.id &&
      !document.isLocked &&
      document.status === "draft"
    )
  }
  return false
}
```

This works, but:
- You've abandoned the simplicity of RBAC
- Permission logic is scattered across helper functions
- The `can(user, permission)` pattern is inconsistent
- You're essentially hand-building ABAC without the structure

### When to Graduate from RBAC

Consider moving beyond RBAC when you find yourself:

1. **Creating permission names with multiple conditions**
   - `"document:update:own-unlocked-draft"` is a red flag

2. **Writing helper functions for every permission check**
   - If `canDoX(user, resource)` is the norm, RBAC isn't helping

3. **Duplicating attribute checks across the codebase**
   - The same `document.isLocked` check appearing everywhere

4. **Needing different permissions for the same action based on environment**
   - Same "edit" action has different rules on weekday vs weekend

---

## Attribute-Based Access Control (ABAC)

### The Core Concept

**Attribute-Based Access Control (ABAC)** is a permission system that makes decisions based on **attributes** of users, resources, and the environment. Instead of asking "Does this role have permission?", ABAC asks "Do the attributes of this request satisfy the policy?"

### The Four Categories of Attributes

ABAC evaluates permissions using four categories:

| Category        | Description                 | Examples                                                     |
| --------------- | --------------------------- | ------------------------------------------------------------ |
| **Subject**     | Who is making the request   | `user.role`, `user.department`, `user.id`                    |
| **Resource**    | What is being accessed      | `document.status`, `document.creatorId`, `document.isLocked` |
| **Action**      | What operation is requested | `read`, `create`, `update`, `delete`                         |
| **Environment** | Contextual factors          | Time of day, IP address, device type                         |

### How ABAC Differs from RBAC

```text
RBAC:  User → Role → Permission
ABAC:  (Subject + Resource + Action + Environment) → Policy → Decision
```

In RBAC, you ask: "Does the admin role have the document:update permission?"

In ABAC, you ask: "Can a user with `role=editor` and `department=Engineering` update a document with `status=draft` and `isLocked=false`?"

### A Simple ABAC Policy

Here's what an ABAC policy looks like conceptually:

```typescript
// "Authors can edit their own unlocked draft documents"
{
  action: "update",
  resource: "document",
  conditions: {
    "user.role": "author",
    "document.creatorId": "user.id",  // Dynamic comparison!
    "document.isLocked": false,
    "document.status": "draft"
  }
}
```

### Why ABAC Solves Our Problems

Remember the permissions that broke our RBAC system?

| Requirement          | RBAC Approach                                               | ABAC Approach                             |
| -------------------- | ----------------------------------------------------------- | ----------------------------------------- |
| "Edit own documents" | Create `document:update:own` permission + manual check      | `condition: { creatorId: user.id }`       |
| "Edit unlocked only" | Create `document:update:unlocked` permission + manual check | `condition: { isLocked: false }`          |
| "View non-drafts"    | Create `document:read:non-draft` permission + manual check  | `condition: { status: { $ne: "draft" } }` |

ABAC expresses these naturally as **conditions**, not encoded permission strings.

### ABAC Implementation

Here's a simplified ABAC implementation:

```typescript
function getUserPermissions(user: User) {
  const builder = new PermissionBuilder()

  if (user.role === "author") {
    builder
      .allow("document", "read", { status: "published" })
      .allow("document", "read", { status: "archived" })
      .allow("document", "read", { status: "draft", creatorId: user.id })
      .allow("document", "create")
      .allow("document", "update", {
        creatorId: user.id,
        isLocked: false,
        status: "draft",
      })
  }

  return builder.build()
}

// Usage
const permissions = getUserPermissions(user)
permissions.can("document", "update", document) // Checks all conditions!
```

### What ABAC Provides

| Benefit                    | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| **Unified API**            | One `can()` function for all checks                          |
| **Declarative policies**   | Conditions describe what's allowed                           |
| **No helper functions**    | No more `canUpdateDocument`, `canReadProject`, etc.          |
| **Type safety**            | TypeScript enforces valid resources, actions, and conditions |
| **Composable**             | Easy to add new conditions without new permission strings    |

---

## Advanced ABAC Features

### 1. Field-Level Permissions

Different roles should see or modify different fields on the same resource:

#### Field-Level Read Permissions

| Field            | Admin | Editor | Author | Viewer |
| ---------------- | ----- | ------ | ------ | ------ |
| `title`          | ✅    | ✅     | ✅     | ✅     |
| `content`        | ✅    | ✅     | ✅     | ✅     |
| `createdAt`      | ✅    | ✅     | ✅     | ❌     |
| `updatedAt`      | ✅    | ✅     | ✅     | ❌     |

#### Field-Level Write Permissions

| Field      | Admin | Editor | Author |
| ---------- | ----- | ------ | ------ |
| `title`    | ✅    | ✅     | ✅     |
| `content`  | ✅    | ✅     | ✅     |
| `status`   | ✅    | ✅     | ❌     |
| `isLocked` | ✅    | ❌     | ❌     |

### Implementing Field-Level Permissions

The `allow` function needs to accept an optional fields array:

```typescript
function addEditorPermissions(
  builder: PermissionBuilder,
  user: Pick<User, "department">,
) {
  builder.allow("document", "read") // All fields by default

  builder.allow(
    "document",
    "update",
    { isLocked: false },
    ["content", "title", "status"], // Restricted to these fields
  )
}
```

### Enhanced Permission API

```typescript
// Check if user can read a specific field
permissions.can("document", "read", document, "isLocked")

// Check if user can update a specific field
permissions.can("document", "update", document, "status")
```

### 2. Environment-Based Rules

Rules that depend on contextual factors like time of day:

```typescript
const builder = new PermissionBuilder()

const dayOfWeek = new Date().getDay()
const isWeekend = dayOfWeek === 0 || dayOfWeek === 6

builder.allow("document", "read")

// Only allow updates on weekdays
if (!isWeekend) {
  builder.allow("document", "update", { isLocked: false })
}
```

### 3. Automatic Database Filtering

Convert ABAC conditions directly to SQL WHERE clauses:

```typescript
// ABAC condition → SQL query
{ status: "published", department: user.department }
// becomes
WHERE status = 'published' AND department = 'Engineering'
```

This is powerful because:
- We only fetch data the user can see
- The database does the filtering (efficient!)
- No chance of data leaks from forgetting a filter

---

## ABAC Limitations

### Where ABAC Excels

ABAC is the right choice when your permissions are:

| Characteristic               | Example                                                             |
| ---------------------------- | ------------------------------------------------------------------- |
| **Attribute-driven**         | "Users can edit their own unlocked draft documents"                 |
| **Fine-grained**             | Permissions depend on resource state, ownership, or context         |
| **Environment-aware**        | Rules change based on time, location, or device                     |
| **Field-level restrictions** | Different roles see or modify different fields on the same resource |

### Where ABAC Struggles

#### 1. TypeScript Complexity

As your application grows and you add more resources/conditional types, maintaining the type system becomes increasingly difficult:

```typescript
// Started manageable
type Resources = {
  project: {
    action: "create" | "read" | "update" | "delete"
    data: Project
    condition: Pick<Project, "department">
  }
  document: {
    action: "create" | "read" | "update" | "delete"
    data: Document
    condition: Pick<Document, "projectId" | "creatorId" | "status" | "isLocked">
  }
}

// Then you add complex condition operators
type Condition<T> = { $gte: T } | { $lte: T } | { $between: [T, T] } | { $ne: T } | { $in: T[] }

// The permission builder now needs to handle all these extra condition operators
type Permission<Res extends keyof Resources> = {
  action: Resources[Res]["action"]
  condition?: // ??? This gets massive
  fields?: (keyof Resources[Res]["data"])[]
}
```

Every new condition operator or resource type requires updating multiple type definitions, and the cognitive overhead grows with each addition.

#### 2. Over-Engineering for Simple Use Cases

Many applications don't actually need advanced ABAC features:
- Field-level permissions
- Complex conditions
- Automatic query generation

If your application only needs simple attribute checks, a **basic ABAC implementation** (without field filtering and complex operators) dramatically reduces complexity while still being more flexible than RBAC.

```typescript
// Full ABAC with all the bells and whistles
builder.allow(
  "document",
  "update",
  { creatorId: user.id, isLocked: false, status: { $in: ["draft", "review"] } },
  ["title", "content", "status"],
)

// Basic ABAC - still powerful, much simpler
builder.allow("document", "update", { creatorId: user.id, isLocked: false })
```

Basic ABAC gives you:
- Attribute-based conditions (ownership, status checks)
- The unified `can()` API
- Type safety for resources and actions

Without the overhead of:
- Field-level permission tracking
- Complex condition operators
- Automatic query generation
- Multiple layers of TypeScript generics

---

## Permission Libraries

### When to Use a Library

Building your own permission system makes sense when:
- Learning how permissions work
- You need maximum control over your permission logic
- Your permissions aren't too complex

Using a library makes sense when:
- You want well-tested, production-ready code
- You need advanced features
- You want to follow established patterns
- Your team needs documentation and community support

### Two Types of Permission Libraries

#### 1. In-Code Libraries

These libraries define permissions using the same programming language as your application:

```typescript
// Permissions defined in JavaScript/TypeScript
const ability = defineAbility((can, cannot) => {
  can("read", "Document", { status: "published" })
  can("update", "Document", { creatorId: user.id })
  cannot("delete", "Document", { isLocked: true })
})
```

**Pros:**
- Full IDE support (autocomplete, type checking)
- Easy to debug (same language, same tools)
- No context switching between languages
- Can use application logic directly

**Cons:**
- Policies are embedded in application code
- Harder to externalize or share policies
- Changes require redeploying the application

**Popular in-code libraries:**
- **CASL** (JavaScript/TypeScript)
- **Pundit** (Ruby)
- **Cancan** (Ruby)

#### 2. DSL Libraries (Domain-Specific Language)

These libraries define permissions using a dedicated policy language:

```ini
# model.conf - Defines the permission structure
[request_definition]
r = sub, obj, act

[policy_definition]
p = sub, obj, act

[matchers]
m = r.sub == p.sub && r.obj == p.obj && r.act == p.act
```

```csv
# policy.csv - The actual rules
p, admin, document, read
p, admin, document, update
p, admin, document, delete
p, editor, document, read
p, editor, document, update
```

**Pros:**
- Policies are separate from application code
- Can update policies without redeploying
- Same policies work across different services
- Easier to audit (policies in one place)

**Cons:**
- Must learn a new language
- Limited IDE support (usually)
- Debugging across language boundary is harder
- Extra infrastructure to manage policy files

**Popular DSL libraries:**
- **Casbin** - Supports multiple languages
- **Open Policy Agent (OPA)** - Uses Rego language
- **Cedar** - AWS's policy language

---

## CASL Implementation

### Installing CASL

```bash
npm install @casl/ability
```

### Key Changes from Custom ABAC

```typescript
// BEFORE (custom): resource first, action second
builder.allow("document", "read")
builder.allow("document", "read", { status: "published" }, ["title", "content"])

// AFTER (CASL): action first, resource second, fields before conditions
allow("read", "document")
allow("read", "document", ["title", "content"], { status: "published" })
```

### Type Setup

CASL needs types to understand your resources:

```typescript
type FullCRUD = "create" | "read" | "update" | "delete"
type ProjectSubject = "project" | Pick<Project, "department">
type DocumentSubject =
  | "document"
  | Pick<Document, "projectId" | "creatorId" | "status" | "isLocked">

// [Actions, Resource] tuple
type MyAbility = MongoAbility<
  [FullCRUD, ProjectSubject] | [FullCRUD, DocumentSubject]
>
```

### Using CASL in Components

```typescript
// Custom: pass object directly
if (permissions.can("document", "update", document)) {
  // ...
}

// CASL: wrap object with subject() helper
import { subject } from "@casl/ability"

if (permissions.can("update", subject("document", document))) {
  // ...
}
```

**Note the differences:**
1. Resource and action order is swapped: `can("update", "document")` vs `permissions.can("document", "update")`
2. Use `subject()` helper to wrap data for condition checking

**Why the `subject()` wrapper?** CASL needs to know what _type_ of object you're checking. Our custom implementation knew because we passed the resource name first. CASL uses class names by default, but since we're using plain objects, we need to tell it explicitly.

### CASL Benefits

#### 1. Advanced Conditions

CASL supports MongoDB-style query operators:

```typescript
can("read", "document", { status: { $ne: "archived" } }) // not equal
can("read", "document", { status: { $in: ["published", "draft"] } }) // in array
```

#### 2. No Complicated TypeScript

CASL handles all the complex TypeScript for you, so you don't have to manually define types for each resource and action combination.

#### 3. No Maintenance Headaches

CASL is actively maintained and widely used, so you don't have to worry about maintaining a custom permission system.

### CASL Drawbacks

#### 1. Reliance on Classes

CASL uses class names to determine which permissions to check on an object. Since we're using plain objects (not class instances), permissions don't work by default which is why we need the `subject()` wrapper everywhere.

#### 2. React Server Component Compatibility

The `subject()` function mutates our objects in a way that is incompatible with React Server components. There are two workarounds:

**Option A: Spread into a new object**

```typescript
// Always create a new object before passing to subject()
ability.can("update", subject("document", { ...document }))
```

**Option B: Use `detectSubjectType`**

Configure CASL to detect subject types without mutation:

```typescript
build({ detectSubjectType: (object) => object.__caslType })

// In your queries, add __caslType to returned objects
const document = await db.query.documents.findFirst({
  where: eq(documents.id, id),
})

return { ...document, __caslType: "document" }
```

This approach adds a small overhead but makes consuming CASL permissions easier.

---

## Choosing the Right Permission Model

### The Evolution Summary

| Stage | Approach | Single Source of Truth | Automatic Enforcement | Consistent Front/Back |
|-------|----------|----------------------|----------------------|----------------------|
| 1 | Scattered Checks | ❌ | ❌ | ❌ |
| 2 | Fixed Bugs | ❌ | ❌ | ❌ |
| 3 | Services Layer | ⚠️ | ✅ | ❌ |
| 4 | Basic RBAC | ⚠️ | ✅ | ✅ |
| 5 | RBAC + Helpers | ⚠️ | ✅ | ✅ |
| 6 | Basic ABAC | ⚠️ | ✅ | ✅ |
| 7 | Advanced ABAC | ✅ | ✅ | ✅ |
| 8 | CASL | ✅ | ✅ | ✅ |

### RBAC (Role-Based Access Control)

```typescript
const permissions = {
  admin: ["create", "read", "update", "delete"],
  editor: ["read", "update"],
  viewer: ["read"],
}
```

#### Pros

| Benefit                    | Description                                        |
| -------------------------- | -------------------------------------------------- |
| **Single source of truth** | All permissions defined in one file                |
| **Readable code**          | `can(user, "document:create")` is self-documenting |
| **Type safety**            | TypeScript ensures valid permission names          |
| **Easy updates**           | Change role permissions in one place               |

#### Cons

| Limitation                 | Description                                     |
| -------------------------- | ----------------------------------------------- |
| **No attribute awareness** | Can't express "edit own documents"              |
| **No resource context**    | Can't check document status or ownership        |
| **Permission explosion**   | Complex rules require many specific permissions |

#### When to Use RBAC

- Simple app with admin/user roles
- Manageable number of roles
- No/minimal attribute-based rules needed

#### When to Avoid RBAC

- Permissions depend on resource data
- Rules vary by context or environment
- Field-level restrictions needed

### ABAC (Attribute-Based Access Control)

```typescript
builder.allow("document", "update", {
  creatorId: user.id,
  isLocked: false,
  status: "draft",
})
```

#### Pros

| Benefit                  | Description                                    |
| ------------------------ | ---------------------------------------------- |
| **Unified API**          | One `can()` function for all checks            |
| **Declarative policies** | Conditions describe what's allowed             |
| **No helper explosion**  | No `canUpdateDocument`, `canReadProject`, etc. |
| **Composable**           | Easy to add new conditions                     |

#### Cons

| Limitation              | Description                           |
| ----------------------- | ------------------------------------- |
| **More complex setup**  | Requires building a permission engine |
| **TypeScript overhead** | Condition types can get complex       |

#### When to Use ABAC

- Permissions depend on resource state
- Ownership, status, or context matters
- Need field-level restrictions
- Environment-aware rules (time, location)

#### When to Avoid ABAC

- Only have simple role-based permissions

### Using a Library (CASL)

```typescript
import { AbilityBuilder, Ability } from "@casl/ability"
```

#### What CASL Adds

| Feature                 | Description                                     |
| ----------------------- | ----------------------------------------------- |
| **Battle-tested**       | Production-ready, well-documented               |
| **Familiar API**        | Popular library that your team may already know |
| **Advanced conditions** | MongoDB-style operators (`$ne`, `$in`, etc.)    |

#### CASL Tradeoffs

| Issue                  | Description                                     |
| ---------------------- | ----------------------------------------------- |
| **Class-based design** | Requires `subject()` wrapper for plain objects  |
| **RSC compatibility**  | `subject()` mutates objects (needs workarounds) |

#### When to Use CASL

- You need ABAC level permissions, but don't want to maintain an ABAC system

#### When to Avoid CASL

- You don't need ABAC level permissions
- You don't need the advanced features of CASL
- Working around the `subject()` wrapper is more hassle than writing your own ABAC implementation

---

## Decision Flowchart

### Quick Decision Guide

```
Start
  │
  ▼
┌─────────────────────────────────────┐
│ Are permissions purely role-based?  │
│ (No resource attributes needed)     │
└─────────────────────────────────────┘
  │                    │
  YES                  NO
  │                    │
  ▼                    ▼
┌─────────┐    ┌─────────────────────────────┐
│  RBAC   │    │ Do you need field-level     │
│         │    │ permissions or complex      │
│         │    │ conditions?                 │
└─────────┘    └─────────────────────────────┘
                    │                    │
                    YES                  NO
                    │                    │
                    ▼                    ▼
             ┌─────────────┐    ┌─────────────────┐
             │ Basic ABAC  │    │ Do you want to  │
             │ (custom)    │    │ avoid maintaining│
             │             │    │ custom code?    │
             └─────────────┘    └─────────────────┘
                                     │                    │
                                     YES                  NO
                                     │                    │
                                     ▼                    ▼
                              ┌─────────────┐    ┌─────────────┐
                              │   CASL      │    │ Basic ABAC  │
                              │ (library)   │    │ (custom)    │
                              └─────────────┘    └─────────────┘
```

### General Recommendation

I generally recommend using a **basic ABAC system** as a starting point for most applications. It provides flexibility without the complexity of an advanced ABAC system, and most projects evolve past what RBAC can support rather quickly.

From there, add additional features as you need them, which makes the system easier to maintain over time.

---

## Key Takeaways

1. **Start simple** - Begin with basic authorization, refactor when needed
2. **Centralize authorization** - Use a permissions module and services layer
3. **Match complexity to requirements** - Don't over-engineer; basic ABAC might be enough
4. **Consider libraries** - CASL, Casbin, or managed services to save time
5. **Never trust the client** - All permission checks must be enforced on the server
6. **Fail closed** - When in doubt, deny access
7. **Single source of truth** - Permission logic should live in one place
8. **Automatic enforcement** - Authorization should happen by default
9. **Consistent across frontend and backend** - Same rules, same results
10. **Plan for change** - Permission rules will evolve; make that evolution painless