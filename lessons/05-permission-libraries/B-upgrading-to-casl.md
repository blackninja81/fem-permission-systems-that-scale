---
title: "Upgrading to CASL"
---

Let's upgrade our custom ABAC implementation to use CASL.

## Installing CASL

First, install the CASL library:

```bash
npm install @casl/ability
```

We will also be using the types from the `@ucast/core` library:

```bash
npm install @ucast/core
```

## Updating the Permission Module

### Key Changes

Here's what changes when migrating to CASL:

```typescript
// BEFORE (custom): resource first, action second
builder.allow("document", "read")
builder.allow("document", "read", { status: "published" }, ["title", "content"])

// AFTER (CASL): action first, resource second, fields before conditions
allow("read", "document")
allow("read", "document", ["title", "content"], { status: "published" })
```

### Type Setup

CASL needs types to understand your resources which are broken down into subjects and actions:

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

## Using CASL in Components

The usage pattern is almost identical, with one key difference:

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

Note the differences:

1. Resource and action order is swapped: `can("update", "document")` vs `permissions.can("document", "update")`
2. Use `subject()` helper to wrap data for condition checking

> Why the `subject()` wrapper? CASL needs to know what _type_ of object you're checking. Our custom implementation knew because we passed the resource name first. CASL uses class names by default, but since we're using plain objects, we need to tell it explicitly.

## Implementation

Let's convert our current permission checks to use CASL

## CASL Benefits

CASL provides several benefits over a custom implementation:

### 1. Advanced Conditions

CASL supports MongoDB-style query operators:

```typescript
can("read", "document", { status: { $ne: "archived" } }) // not equal
can("read", "document", { status: { $in: ["published", "draft"] } }) // in array
```

### 2. No Complicated TypeScript

CASL handles all the complex TypeScript for you, so you don't have to manually define types for each resource and action combination.

### 3. No Maintenance Headaches

CASL is actively maintained and widely used, so you don't have to worry about maintaining a custom permission system.

## CASL Drawbacks

CASL is great at many things, but since it was created as a backend Node focused project it struggles when used with frontend frameworks like React or Next.js due to its reliance on classes.

### 1. Reliance on Classes

CASL uses class names to determine which permissions to check on an object. Since we're using plain objects (not class instances), permissions don't work by default which is why we need the `subject()` wrapper everywhere.

### 2. React Server Component Compatibility

The `subject()` function unfortunately, mutates our objects in a way that is incompatible with React Server components. There are two workarounds:

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

## Branch Checkpoint

After completing this lesson, your code should match:

```text
Branch: 8-casl
```

Run the following to sync up:

```bash
git checkout 8-casl
```

## Summary

Migrating to CASL gives us:

- ✅ Battle-tested permission library
- ✅ Built-in field-level permissions
- ✅ Community support and documentation

The concepts are identical to what we built, but using a library like CASL can give you extra functionality and make it easier to work with on a larger team.
