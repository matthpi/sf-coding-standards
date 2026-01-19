# Sharing Model

## Overview

**Always use explicit sharing declarations** on all classes (never omit).

## Sharing Keywords Explained

### `inherited sharing`

The class adopts the sharing context of its caller:
- Runs in `with sharing` mode if called from a `with sharing` context
- Runs in `without sharing` mode if called from a `without sharing` context
- Defaults to `with sharing` if called from an Apex REST service or trigger

**Use for:** Selectors, Services, Utils, Invocables, Application/Core classes

### `with sharing`

Enforces record-level security and sharing rules based on the current user's permissions.

**Use for:** Controllers, any user-facing code that should enforce sharing

### `without sharing`

Runs in system context, bypassing sharing rules and accessing all records.

**Use for:** Batch/Queueable/Schedulable jobs, system-level maintenance operations

**⚠️ Never use on user-facing classes without explicit security review**

### No declaration

Test classes should have no sharing declaration (they run in system context).

## Security Best Practices

1. **Always be explicit** - Never omit sharing declarations
2. **Default to `inherited sharing`** for core layers (Selector, Service, Domain, Utils)
3. **Use `with sharing`** for anything users interact with directly
4. **Justify `without sharing`** - Document why system context is needed
5. **Review regularly** - Audit sharing declarations during code reviews
