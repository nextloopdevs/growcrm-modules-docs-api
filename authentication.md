---
title: Authentication
description: How to authenticate requests to the CRM API.
---

# Authentication

Every CRM API request is authenticated with an **API key**, sent as a Bearer token in the
`Authorization` header:

```
Authorization: Bearer YOUR_API_KEY
```

## Obtaining a key

API keys are created inside the CRM under **App → Settings → API → API Keys**. Give the key a name and the
system generates its value. Treat the key like a password — anyone with it has full API access.

## Privileges

All API keys are **administrator-level**: a valid key runs with full privileges as the CRM's
**main administrator**. There are no user-scoped keys.

Where an action supports per-record attribution, you can set it explicitly in the payload — for
example `project_creatorid` when creating a project (it defaults to the main administrator if
omitted).

## Multitenant (SaaS)

On the SaaS subscription edition, call the API on **your own account domain**:

```
https://abcd.domain.com/api/...
```

The domain identifies your account, and the API key is validated against your account's data.
No account identifier is needed in the key or the URL.

## Errors

| Status | Meaning |
|---|---|
| `401 Unauthorized` | The API key is missing or invalid. |

```json
{
  "message": "Unauthorized. A valid API key is required."
}
```
