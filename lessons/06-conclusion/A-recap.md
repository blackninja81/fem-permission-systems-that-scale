---
title: "Choosing the Right Permission Model"
---

Let's recap what we've learned, how our codebase evolved, and when to use each authorization approach.

## Remembering Our Goals

At the start of this workshop, we established what a well-designed permission system should achieve:

- **Prevent unauthorized access**
- **Single source of truth**
- **Automatic enforcement**
- **Consistent across frontend and backend**
- **Never trust the client**
- **Fail closed**

All of our systems by default fail closed so we don't have to worry about that, but we will analyze how each evolution of our codebase addressed (or failed to address) these goals.

## The Evolution of Our Permission System

### Stage 1: Basic Permissions (Scattered Checks)

**Branch:** `1-basic-permissions`

We started with permission checks scattered throughout the codebase and no central policy system:

```typescript
// In a page component
if (user.role !== "admin" && user.role !== "author") {
  return redirect("/")
}

// In another file - slightly different logic
if (user.role === "viewer" || user.role === "editor") {
  throw new Error("Unauthorized")
}
```

#### Goals Assessment

| Goal                        | Status | Notes                                  |
| --------------------------- | ------ | -------------------------------------- |
| Prevent unauthorized access | ❌     | Missing checks on several pages        |
| Single source of truth      | ❌     | Logic scattered everywhere             |
| Automatic enforcement       | ❌     | Developers must remember to add checks |
| Consistent front/back end   | ❌     | Different logic in different places    |
| Never trust the client      | ❌     | Some pages relied only on UI hiding    |

### Stage 2: Fixing Permission Bugs

**Branch:** `2-fix-basic-permission-errors`

We added missing permission checks to all vulnerable pages and fixed incorrect permission logic.

```typescript
// Added proper server-side checks
export default async function DocumentDetailPage({ params }) {
  const user = await getCurrentUser()
  const project = await getProjectById(params.projectId)

  if (
    user == null ||
    (user.role !== "admin" &&
      project.department != null &&
      user.department !== project.department)
  ) {
    return redirect("/")
  }
  // ...
}
```

#### Goals Assessment

| Goal                        | Status | Notes                       |
| --------------------------- | ------ | --------------------------- |
| Prevent unauthorized access | ✅     | All pages now have checks   |
| Single source of truth      | ❌     | Still duplicated everywhere |
| Automatic enforcement       | ❌     | Still manual                |
| Consistent front/back end   | ❌     | Copy-paste prone to drift   |
| Never trust the client      | ✅     | Server checks in place      |

**The core problem:** We fixed the symptoms but not the root cause. Adding a new permission rule meant updating every file.

### Stage 3: Services Layer

**Branch:** `3-add-service-layer`

We introduced a **services layer** that centralizes business logic and authorization:

```typescript
// Services handle auth + business logic
export async function createDocumentService(
  projectId: string,
  data: DocumentFormValues,
) {
  const user = await getCurrentUser()
  if (user == null) throw new Error("Unauthenticated")

  // Step 1: Authorization - can they perform this action?
  const permissions = await getUserPermissions()
  if (!permissions.can("document", "create")) throw new AuthorizationError()

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

// Actions become thin wrappers
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

#### Goals Assessment

| Goal                        | Status | Notes                                                 |
| --------------------------- | ------ | ----------------------------------------------------- |
| Prevent unauthorized access | ✅     | Services enforce all access                           |
| Single source of truth      | ⚠️     | Better, but still lots of duplication                 |
| Automatic enforcement       | ✅     | Services automatically enforce permissions            |
| Consistent front/back end   | ❌     | Checks on the client are still copy-paste error prone |
| Never trust the client      | ✅     | Services run server-side                              |

**Key insight:** The services layer gave us **security by default** as long as all data access goes through services.

### Stage 4: Basic RBAC

**Branch:** `4-basic-rbac`

We created a centralized permission system where roles map to permissions:

```typescript
type Permission =
  | "document:create"
  | "document:read"
  | "document:update"
  | "document:delete"

const permissionsByRole: Record<UserRole, Permission[]> = {
  admin: [
    "document:create",
    "document:read",
    "document:update",
    "document:delete",
  ],
  editor: ["document:read", "document:update"],
  author: ["document:create", "document:read"],
  viewer: ["document:read"],
}

function can(user: { role: UserRole }, permission: Permission) {
  return permissionsByRole[user.role].includes(permission)
}
```

#### Goals Assessment

| Goal                        | Status | Notes                                                           |
| --------------------------- | ------ | --------------------------------------------------------------- |
| Prevent unauthorized access | ✅     | Services enforce all access                                     |
| Single source of truth      | ⚠️     | Permissions are centralized, but still require helper functions |
| Automatic enforcement       | ✅     | Services automatically enforce permissions                      |
| Consistent front/back end   | ✅     | Same `can()` function everywhere                                |
| Never trust the client      | ✅     | Server-enforced                                                 |

### Stage 5: RBAC Limitations Exposed

**Branch:** `5-rbac-limits`

When we added more realistic requirements, RBAC struggled:

> "Authors can edit documents **they created**, that are **not locked**, in **draft status**"

This forced us to create exploding permission names:

```typescript
type Permission =
  | "document:update:all"
  | "document:update:unlocked"
  | "document:update:own-unlocked-draft"
  | "document:read:all"
  | "document:read:own"
  | "document:read:non-draft"
// ... keeps growing
```

And helper functions that duplicate logic:

```typescript
export function canUpdateDocument(user, document) {
  return (
    can(user, "document:update:all") ||
    (can(user, "document:update:unlocked") && !document.isLocked) ||
    (can(user, "document:update:own-unlocked-draft") &&
      document.creatorId === user.id &&
      !document.isLocked &&
      document.status === "draft")
  )
}
```

#### Goals Assessment

| Goal                        | Status | Notes                                                           |
| --------------------------- | ------ | --------------------------------------------------------------- |
| Prevent unauthorized access | ✅     | Services enforce all access                                     |
| Single source of truth      | ⚠️     | Permissions are centralized, but still require helper functions |
| Automatic enforcement       | ✅     | Services automatically enforce permissions                      |
| Consistent front/back end   | ✅     | Same `can()` function everywhere                                |
| Never trust the client      | ✅     | Server-enforced                                                 |

### Stage 6: Basic ABAC

**Branch:** `6-abac-basic`

ABAC evaluates permissions using attributes of users, resources, actions, and environment:

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

// Usage - one unified API
permissions.can("document", "update", document)
```

#### Goals Assessment

| Goal                        | Status | Notes                                                                    |
| --------------------------- | ------ | ------------------------------------------------------------------------ |
| Prevent unauthorized access | ✅     | Services enforce all access                                              |
| Single source of truth      | ⚠️     | All policies in permission builder, but DB queries duplicate permissions |
| Automatic enforcement       | ✅     | Services automatically enforce permissions                               |
| Consistent front/back end   | ✅     | Same builder everywhere                                                  |
| Never trust the client      | ✅     | Server-enforced                                                          |

### Stage 7: Advanced ABAC

**Branch:** `7-abac-advanced`

We added powerful features that RBAC simply cannot support:

#### Field-Level Permissions

```typescript
// Different roles see different fields
builder.allow("document", "update", { isLocked: false }, [
  "content",
  "title",
  "status",
])

// Check field access
permissions.can("document", "update", document, "status")
```

#### Environment-Based Rules

```typescript
const isWeekend = new Date().getDay() === 0 || new Date().getDay() === 6

if (!isWeekend) {
  builder.allow("document", "update", { isLocked: false })
}
```

#### Automatic Database Filtering

Convert ABAC conditions directly to SQL WHERE clauses:

```typescript
// ABAC condition → SQL query
{ status: "published", department: user.department }
// becomes
WHERE status = 'published' AND department = 'Engineering'
```

#### Goals Assessment

| Goal                        | Status | Notes                                              |
| --------------------------- | ------ | -------------------------------------------------- |
| Prevent unauthorized access | ✅     | Services enforce all access and DB-level filtering |
| Single source of truth      | ✅     | All policies in permission builder                 |
| Automatic enforcement       | ✅     | Services automatically enforce permissions         |
| Consistent front/back end   | ✅     | Same builder everywhere                            |
| Never trust the client      | ✅     | Server-enforced                                    |

### Stage 8: CASL

**Branch:** `8-casl`

We migrated to CASL to avoid reinventing the wheel and to leverage a battle-tested library for complex permission logic.

```typescript
import { AbilityBuilder, createMongoAbility } from "@casl/ability"

const { can: allow, build } = new AbilityBuilder(createMongoAbility)

allow("read", "document", ["title", "content"], { status: "published" })
allow("update", "document", { creatorId: user.id, isLocked: false })

const ability = build()

// Usage
ability.can("update", subject("document", document))
```

#### Goals Assessment

| Goal                        | Status | Notes                                              |
| --------------------------- | ------ | -------------------------------------------------- |
| Prevent unauthorized access | ✅     | Services enforce all access and DB-level filtering |
| Single source of truth      | ✅     | All policies in permission builder                 |
| Automatic enforcement       | ✅     | Services automatically enforce permissions         |
| Consistent front/back end   | ✅     | Same builder everywhere                            |
| Never trust the client      | ✅     | Server-enforced                                    |

## The Permission Models Compared

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

#### When to Use

- Simple app with admin/user roles
- Manageable number of roles
- No/minimal attribute-based rules needed

#### When to Avoid

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

#### When to Use

- Permissions depend on resource state
- Ownership, status, or context matters
- Need field-level restrictions
- Environment-aware rules (time, location)

#### When to Avoid

- Only have simple role-based permissions

### Using a Library

Libraries like CASL or Casbin can simplify permission management by providing a structured API, type safety, and query generation.

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

#### When to Use

- You need ABAC level permissions, but don't want to maintain an ABAC system

#### When to Avoid

- You don't need ABAC level permissions
- You don't need the advanced features of CASL
- Working around the `subject()` wrapper is more hassle than writing your own ABAC implementation

### Quick Decision Guide

I generally recommend using a basic ABAC system as a starting point for most applications. It provides flexibility without the complexity of an advanced ABAC system and most projects evolve past what RBAC can support rather quickly.

From there add additional features as you need them which makes the system easier to maintain over time.

## Key Takeaways

1. **Start simple** - Begin with basic authorization, refactor when needed
2. **Centralize authorization** - Use a permissions module and services layer
3. **Match complexity to requirements** - Don't over-engineer; basic ABAC might be enough
4. **Consider libraries** - CASL, Casbin, or managed services to save time
