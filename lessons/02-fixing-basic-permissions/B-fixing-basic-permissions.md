---
title: "Fixing Basic Permissions"
---

Let's focus on fixing the errors with our basic permission system.

> 💡 **Important:** Each issue is labeled with the comment `FIX:` to make them easier to find.

## Vulnerable Pages

Our app has several pages that are missing permission checks:

| Page                                 | Who Should Access                        |
| ------------------------------------ | ---------------------------------------- |
| `/projects/[id]`                     | Users in same department, or Admins      |
| `/projects/[id]/edit`                | Admins only                              |
| `/projects/new`                      | Admins only                              |
| `/projects/[id]/documents/[id]`      | Users in same department, or Admins      |
| `/projects/[id]/documents/[id]/edit` | Editors/Authors in department, or Admins |
| `/projects/[id]/documents/new`       | Authors in department, or Admins         |

## Incorrect Permissions

There are multiple places where our permission checks are wrong.

| File                                            | Issue                                                 |
| ----------------------------------------------- | ----------------------------------------------------- |
| `app/(dashboard)/projects/[projectId]/page.tsx` | Admins should be able to view the new document button |
| `dal/documents/mutations.ts`                    | Viewers should not be able to create documents        |

## Example Fix

Here's an example of adding a missing permission check:

```tsx
// Before: No permission check!
export default async function DocumentDetailPage({ params }) {
  const document = await getDocumentById(params.documentId)
  return <DocumentDetails document={document} />
}

// After: Check user role before rendering
export default async function DocumentDetailPage({ params }) {
  const user = await getCurrentUser()
  const project = await getProjectById(params.projectId)

  // Permission check
  if (
    user == null ||
    (user.role !== "admin" &&
      project.department != null &&
      user.department !== project.department)
  ) {
    return redirect(`/`)
  }

  const document = await getDocumentById(params.documentId)
  return <DocumentDetails document={document} />
}
```

## The Problem With This System

As we add these checks, notice how we're:

- Copying and pasting the same permission checks everywhere
- Hoping we don't make typos or forget a condition

This is exactly why we need to **centralize** this logic. If our permission rule changes, we now have to update it in **every single file**.

## Branch Checkpoint

After completing these fixes, your code should match:

```text
Branch: 2-fix-basic-permission-errors
```

Run the following to sync up:

```bash
git checkout 2-fix-basic-permission-errors
```
