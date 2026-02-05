---
title: "The Case for Better Architecture"
---

We have now successfully centralized our authorization logic into a services layer, but we still have many issues to address.

## What's Still Wrong

Despite our fixes, the codebase has fundamental problems that will only get worse as the application grows.

### 1. Duplication Everywhere

Count how many times this exact pattern appears:

```typescript
if (user == null || user.role !== "admin") {
  // Redirect/Error
}
```

If we add a new rule (e.g., "managers can create/edit projects"), we need to find and update **every occurrence**. Miss one? Security vulnerability.

### 2. No Single Source of Truth

Where do you go to understand "who can create a document"? There's no clear answer:

- The page component has a check
- The service layer has a check
- The UI conditionally renders buttons

Each place has its own version of the "truth."

### 3. Hard to Read Permission Logic

What does this check actually mean?

```typescript
if (user.role !== "author" && user.role !== "admin") {
  return redirect(`/projects/${projectId}`)
}
```

Compare that to:

```typescript
if (!canCreateDocument(user)) {
  return redirect(`/projects/${projectId}`)
}
```

The second version is self-documenting. The first requires mental parsing.

### 4. Adding New Permissions is Painful

Imagine we need to add a new rule: "Documents can be marked as 'confidential' and only the creator can edit them."

With the current architecture, we'd need to:

1. Update every edit page
2. Update every edit service layer function
3. Update all the UI conditionals
4. Hope we didn't miss anything

## Key Takeaways

Before moving on, remember these principles:

1. **Always check on the server** - Client-side checks are for UX, not security
2. **Centralize permission logic** - One place to define, one place to update
3. **Use descriptive functions** - `canEditDocument(user, doc)` beats inline conditionals
4. **Plan for change** - Permission rules will evolve; make that evolution painless

## What's Next

In the next section, we'll implement **Role-Based Access Control (RBAC)**. We'll create a permissions layer that:

- Defines all permissions in one place
- Provides helper functions for readable code
- Makes adding new permissions straightforward
