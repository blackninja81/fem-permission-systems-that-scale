---
title: "What is ABAC?"
---

**Attribute-Based Access Control (ABAC)** is a permission system that makes decisions based on **attributes** of users, resources, and the environment. Instead of asking "Does this role have permission?", ABAC asks "Do the attributes of this request satisfy the policy?"

## The Core Concept

ABAC evaluates permissions using four categories of attributes:

| Category        | Description                 | Examples                                                     |
| --------------- | --------------------------- | ------------------------------------------------------------ |
| **Subject**     | Who is making the request   | `user.role`, `user.department`, `user.id`                    |
| **Resource**    | What is being accessed      | `document.status`, `document.creatorId`, `document.isLocked` |
| **Action**      | What operation is requested | `read`, `create`, `update`, `delete`                         |
| **Environment** | Contextual factors          | Time of day, IP address, device type                         |

## How ABAC Differs from RBAC

```text
RBAC:  User → Role → Permission
ABAC:  (Subject + Resource + Action + Environment) → Policy → Decision
```

In RBAC, you ask: "Does the admin role have the document:update permission?"

In ABAC, you ask: "Can a user with `role=editor` and `department=Engineering` update a document with `status=draft` and `isLocked=false`?"

## A Simple ABAC Policy

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

## Why ABAC Solves Our Problems

Remember the permissions that broke our RBAC system?

| Requirement          | RBAC Approach                                               | ABAC Approach                             |
| -------------------- | ----------------------------------------------------------- | ----------------------------------------- |
| "Edit own documents" | Create `document:update:own` permission + manual check      | `condition: { creatorId: user.id }`       |
| "Edit unlocked only" | Create `document:update:unlocked` permission + manual check | `condition: { isLocked: false }`          |
| "View non-drafts"    | Create `document:read:non-draft` permission + manual check  | `condition: { status: { $ne: "draft" } }` |

ABAC expresses these naturally as **conditions**, not encoded permission strings.

## ABAC in Code

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

### Implementation

Let's implement a simple ABAC system into our project. This system will follow the exact same policies of our current RBAC system.

The only additional changes we will make is adding a few database optimizations and finally ensuring that users can only access documents within projects they have access to (including creation).

#### Checkpoint

We just finished implementing most of the ABAC permissions file, so now it is your turn to define the remaining permissions and use them in the code.

You can checkout this branch to follow along from where we are:

```bash
git checkout 5.5-basic-abac-checkpoint
```

## What We Gain

| Benefit                    | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| **Unified API**            | One `can()` function for all checks                          |
| **Declarative policies**   | Conditions describe what's allowed                           |
| **No helper functions**    | No more `canUpdateDocument`, `canReadProject`, etc.          |
| **Type safety**            | TypeScript enforces valid resources, actions, and conditions |
| **Composable**             | Easy to add new conditions without new permission strings    |
| **Database Optimizations** | Caching saves repeated queries and improves performance      |
| **Document Policies**      | Documents are correctly limited to the user's access         |

## Branch Checkpoint

After completing this conversion, your code should match:

```text
Branch: 6-abac-basic
```

Run the following to sync up:

```bash
git checkout 6-abac-basic
```

## What's Next

This basic ABAC system has the same functionality as our RBAC + helpers approach, just with cleaner architecture. In the next lessons, we'll add **advanced features** that ABAC makes possible like field-level permissions and automatic query filtering.
