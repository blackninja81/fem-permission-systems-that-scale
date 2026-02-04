---
title: "Advanced ABAC Features"
---

Our basic ABAC system works, but many real world applications need more sophisticated features. Let's add **field-level permissions** and environment-based rules.

## New Requirements

In many applications certain users are restricted on which fields they can read or modify. This is something that is impossible to do in RBAC, but ABAC makes these policies straightforward to express using field-level permissions.

### 1. Field-Level Read Permissions

Different roles should see different fields on documents:

| Field            | Admin | Editor | Author | Viewer |
| ---------------- | ----- | ------ | ------ | ------ |
| `title`          | ✅    | ✅     | ✅     | ✅     |
| `content`        | ✅    | ✅     | ✅     | ✅     |
| `status`         | ✅    | ✅     | ✅     | ✅     |
| `isLocked`       | ✅    | ✅     | ✅     | ❌     |
| `creatorId`      | ✅    | ✅     | ✅     | ❌     |
| `lastEditedById` | ✅    | ✅     | ✅     | ❌     |
| `createdAt`      | ✅    | ✅     | ✅     | ❌     |
| `updatedAt`      | ✅    | ✅     | ✅     | ❌     |

### 2. Field-Level Write Permissions

When creating or editing documents, only certain fields should be writable:

| Field      | Admin | Editor | Author |
| ---------- | ----- | ------ | ------ |
| `title`    | ✅    | ✅     | ✅     |
| `content`  | ✅    | ✅     | ✅     |
| `status`   | ✅    | ✅     | ❌     |
| `isLocked` | ✅    | ❌     | ❌     |

Authors can write content but can't change document status or lock documents.

### 3. Environment-Based Rules

Since this company values work-life balance, editors and authors are not allowed to make changes on weekends which is an example of an **environment-based** rule that uses attributes from neither the user nor the resource.

## Implementing Environment Based Rules

Adding environment-based rules is straightforward. We can just check the current day in our permission builder and use that to filter which permissions we allow:

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

## Implementing Field-Level Permissions

Field level permissions are much more complicated to implement. This is mostly because consuming field level permissions requires lots of code. Implementing field level checks in our policy engine is actually quite easy.

### Extending the Permission API

We need to enhance our `can()` function to support field checks:

```typescript
// Check if user can read a specific field
permissions.can("document", "read", document, "isLocked")

// Check if user can update a specific field
permissions.can("document", "update", document, "status")
```

### Enhanced Policy Definitions

The `allow` function needs to accept an optional fields array as a fourth parameter:

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

### Implementing the Permission Builder Updates

We only need to change three things in our existing builder:

1. Add support for field-level permissions in the `allow()` method.
2. Update the `can()` method to check field-level permissions.
3. Add a way to filter permitted fields

Let's do that now.

### Using Field Permissions

Adding support for field permissions takes a little bit of TypeScript wizardry and extra work, but for the most part isn't too bad. Where field permissions become difficult to work with is when consuming which fields a user has access to and ensuring the UI works for all possible permutations.

We need to update all the following locations:

1. Forms
2. Actions
3. Schemas
4. Display pages

Let's do that now.

## The Power of ABAC

Notice what we can now express:

```typescript
// "Authors can read the createdAt field only on documents they created"
permissions.can("document", "read", { creatorId: user.id }, "createdAt")

// "Editors can update status on unlocked documents"
permissions.can("document", "update", { isLocked: false }, "status")
```

These complex, conditional, field-level permissions are natural in ABAC but would require an explosion of permission strings in RBAC which is impossible to manage.

## What's Next

Our ABAC system is powerful, but we still have permission checks scattered throughout:

- Page components
- Server actions
- Mutations
- Query functions

In the next lesson, we'll create a **services layer** that centralizes all authorization logic and even converts permissions to database queries automatically.
