---
title: "ABAC Limitations"
---

Let's summarize when ABAC works well and when you should consider alternatives or simplifications.

## Where ABAC Excels

ABAC is the right choice when your permissions are:

| Characteristic               | Example                                                             |
| ---------------------------- | ------------------------------------------------------------------- |
| **Attribute-driven**         | "Users can edit their own unlocked draft documents"                 |
| **Fine-grained**             | Permissions depend on resource state, ownership, or context         |
| **Environment-aware**        | Rules change based on time, location, or device                     |
| **Field-level restrictions** | Different roles see or modify different fields on the same resource |

## Where ABAC Struggles

Despite its power, ABAC introduces challenges that RBAC doesn't have:

### 1. TypeScript Complexity

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
type Condition<T> = { $gte: T } | { $lte: T } | { $between: [T, T] } |  { $ne: T } | { $in: T[] }

// The permission builder now needs to handle all these extra condition operators
type Permission<Res extends keyof Resources> = {
  action: Resources[Res]["action"]
  condition?: // ??? This gets massive
  fields?: (keyof Resources[Res]["data"])[]
}

// The can function also needs to handle all these conditions now
can() {
  const validData = // Way more complex
}
```

Every new condition operator or resource type requires updating multiple type definitions, and the cognitive overhead grows with each addition.

### 2. Over-Engineering for Simple Use Cases

Many applications don't actually need advanced ABAC features:

- **Field-level permissions**
- **Complex conditions**
- **Automatic query generation**

If your application only needs simple attributes checks then a **basic ABAC implementation** (without field filtering and complex operators) dramatically reduces complexity while still being more flexible than RBAC.

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

## What's Next

The next logical step (if you need these complex ABAC features) is to reach for a library that manages all the complexities behind the scenes for you. We will be looking at converting our entire application to CASL which will reduce the amount of complex TypeScript code in our application and give us many new features we were missing before.
