---
title: "Common Permission Mistakes"
---

Before we implement a proper permission system, let's identify the problems in our current codebase. These issues are extremely common in real-world applications. You've likely seen (or written!) code with these same mistakes.

## Main Permission Mistakes

Our demo app currently suffers from three major permission problems:

| Problem                       | Description                                                            |
| ----------------------------- | ---------------------------------------------------------------------- |
| **Scattered checks**          | Permission logic duplicated across files                               |
| **Missing permission checks** | Missing permission checks on the server/client                         |
| **Inconsistent logic**        | Same permission are mistakenly checked differently in different places |

## Problem 1: Scattered Permission Checks

Look at our current codebase. Permission checks are sprinkled mainly throughout:

- UI (checking if buttons should render)
- Data access layer (checking if operations should proceed)

When the same logic exists in multiple places, bugs are inevitable. A developer updates the permission logic in one place but forgets the others. Now you have inconsistent behavior that's incredibly hard to debug.

> ⚠️ **Real-world example**: A common bug is allowing a user to access an "Edit" page but blocking them from actually saving changes. Or worse, hiding the UI button but not blocking the API endpoint, so a savvy user can still perform unauthorized actions via the browser console.

## Problem 2: Missing Permission Checks

This is the most dangerous mistake. Our current app has pages that can be accessed by **anyone who knows the URL**, even if the UI doesn't show a link to that page.

For example, if you're logged in as a `viewer` and manually navigate to:

```text
/projects/[projectId]/documents/[documentId]
```

You can access that page (even if you normally wouldn't be able to see that document)

**The golden rule**: Never trust the client. If a permission matters, check it on the server.

We also have some places where we are not checking the permissions on the client side which leads to a confusing/broken user experience.

## Problem 3: Inconsistent Permission Logic

Our permission checks are complex and hard to read. Consider checking if a user can create a document:

```typescript
// This check appears in multiple places
if (user.role !== "admin" && user.role !== "author") {
  // deny access
}

// or

if (user.role === "viewer" || user.role === "editor") {
  // deny access
}
```

We can do this in multiple different ways, and if we don't create a central place for this logic, different developers might implement the same logic in different ways leading to extra confusion when reading the code.

Even worse, some places might accidentally miss a condition or check the wrong roles.

This problems with this are that the permission logic is:

- **Duplicated** across multiple files
- **Error-prone** (easy to miss a condition)
- **Hard to read** (what does this actually check?)
- **Inconsistent** (some places might check slightly differently)

## Why This Matters

These issues compound over time:

1. **Security vulnerabilities** - Missing checks = unauthorized access
2. **Maintenance nightmare** - Changing permission logic requires updating many files
3. **Testing difficulty** - Hard to verify all permission paths work correctly
4. **Onboarding friction** - New developers don't know where all the checks live

## Our Goals for This Section

By the end of this section, we'll fix all the permission mistakes in our app that are labeled with `FIX:` comments.
