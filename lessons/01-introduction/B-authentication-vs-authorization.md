---
title: "Authentication vs Authorization"
description: "Understanding the critical difference between authentication (who are you?) and authorization (what can you do?), and why authorization is often the harder problem."
---

# Authentication vs Authorization

Before diving into permission systems, we need to clearly understand two concepts that are often confused: **authentication** and **authorization**.

## The Key Distinction

| Concept            | Question It Answers | Example                             |
| ------------------ | ------------------- | ----------------------------------- |
| **Authentication** | "Who are you?"      | Logging in with email/password      |
| **Authorization**  | "What can you do?"  | Can this user delete this document? |

Think of it like entering a secure building:

- **Authentication** is showing your ID badge at the entrance, proving you are who you claim to be
- **Authorization** is the access levels on your badge, determining which floors and rooms you can enter

## Authorization Is Where Most Complexity Lives

Authorization is generally more complex because:

### 1. It's Deeply Tied to Business Logic

Every application has unique rules:

- "Editors can modify documents, but only in their department"
- "Authors can publish their own drafts, but need approval for others"
- "Admins can do everything, except delete locked documents"

These rules can't be outsourced to a generic provider, they're specific to _your_ application.

### 2. It Touches Every Part of Your Codebase

Permission checks appear everywhere:

- **API routes**: Can this user call this endpoint?
- **Database queries**: Which records should be returned?
- **UI components**: Should this button be visible?
- **Business logic**: Is this state transition allowed?

### 3. It Evolves Constantly

Business requirements change:

- "We need to add a new role for external contractors"
- "Documents should be locked after 30 days"
- "Department heads can now approve cross-department access"

A rigid permission system becomes a bottleneck for feature development.

## The Cost of Getting It Wrong

Poor authorization leads to real problems:

| Issue                | Consequence                                                  |
| -------------------- | ------------------------------------------------------------ |
| **Over-permissive**  | Data breaches, unauthorized actions, compliance violations   |
| **Under-permissive** | Frustrated users, support tickets, lost productivity         |
| **Inconsistent**     | Security holes where checks are missing, confusing UX        |
| **Unmaintainable**   | New features require touching dozens of files, bugs multiply |

## This Workshop's Focus

This workshop is **100% about authorization**. We assume you already have authentication handled. Users can log in and you know who they are.

Our challenge is: given a known user, how do we efficiently and maintainably determine what they're allowed to do?

## Goals of a Permission System

Before diving into the code, it's important to understand what a well-designed permission system should look like:

- **Impossible to access unauthorized data**: This means all data access should go through auth checks before any data is returned or modified.

- **Single source of truth**: Permission logic should be centralized in one place, not scattered throughout your codebase. This makes it easier to audit, update, and reason about.

- **Automatic enforcement**: Authorization should be automatically applied whenever data is accessed or mutated. Developers shouldn't need to remember to add permission checks manually. They should happen by default.

- **Consistent across frontend and backend**: The same permission rules should produce the same results whether evaluated on the client or server. This prevents confusing UX where buttons appear enabled but actions fail.

- **Never trust the client**: All permission checks must be enforced on the server. Client-side checks are only for UX (hiding buttons, disabling fields) and never for actual security.

- **Fail closed**: When in doubt, deny access. If something goes wrong or a check is missing, the default should be to block access rather than allow it.
