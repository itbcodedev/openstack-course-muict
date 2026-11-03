# Lecture 3: Keystone — Identity & Access Management

## Learning Objectives

- Understand authentication vs. authorization in OpenStack
- Manage users, projects, roles, and domains
- Work with service catalogs and endpoints

---

## What is Keystone?

Keystone is the **identity service** in OpenStack. It provides:

- **Authentication** — verifying who you are
- **Authorization** — determining what you can do
- **Service Catalog** — listing available services and their endpoints

---

## Core Concepts

```
┌─────────────────────────────────────────┐
│               Domain                     │
│  ┌─────────────────────────────────┐    │
│  │          Project (Tenant)        │   │
│  │  ┌───────────────────────────┐   │   │
│  │  │         User               │   │   │
│  │  │  Role: admin / member / reader │ │
│  │  └───────────────────────────┘   │   │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

### Domain
- Top-level namespace for users, groups, and projects
- Enables multi-tenancy at the organizational level
- Default domain: `default`

### Project (formerly Tenant)
- A container for resources (VMs, networks, volumes)
- Users are granted access via roles within a project

### User
- An entity representing a person or service account
- Authenticated via password, token, or external IdP

### Role
- Defines a set of permissions within a project
- Common roles: `admin`, `member`, `reader`

### Group
- Collection of users
- Roles can be assigned to groups for easier management

---

## Token-Based Authentication

```
User credentials → Keystone
    ↓
Keystone verifies credentials
    ↓
Returns Token (UUID or JWT)
    ↓
User presents Token to other services
    ↓
Services ask Keystone to validate Token
```

### Token Types

| Type | Description |
|------|-------------|
| **UUID** | Random string, validated by Keystone on every request |
| **PKI/PKIz** | Self-validating, cryptographically signed |
| **Fernet** | Default in modern OpenStack, encrypted payload |

---

## Service Catalog & Endpoints

Keystone maintains a catalog of all OpenStack services:

| Endpoint Type | Purpose |
|---------------|---------|
| **public** | External-facing API |
| **internal** | Inter-service communication |
| **admin** | Administrative operations |

Example:
```yaml
Service: compute (nova)
  public:   https://cloud.muict.ac.th:8774/v2.1
  internal: http://10.0.0.11:8774/v2.1
  admin:    http://10.0.0.11:8774/v2.1
```

---

## Policy Files

Each service has a `policy.json` or `policy.yaml` that maps roles to API operations:

```yaml
# Example: Only admins can create flavors
"os_compute_api:os-flavor-manage:create": "rule:admin_api"
```

---

## Federated Identity

Keystone supports external identity providers:

- **LDAP** — Active Directory, OpenLDAP
- **SAML 2.0** — Shibboleth, ADFS
- **OpenID Connect** — Google, Keycloak

This enables single sign-on (SSO) for OpenStack.

---

## Hands-on Commands

```bash
# List users
openstack user list

# Create a project
openstack project create --description "Student Project" student-project

# Assign role to user in project
openstack role add --user alice --project student-project member

# Get token
openstack token issue
```

---

## Slides

📥 [Download Lecture 3 Slides (PDF)](../assets/slides/lecture03.pdf){ .md-button }

## Further Reading

- [Keystone Administrator Guide](https://docs.openstack.org/keystone/latest/admin/)
- [RBAC in OpenStack](https://docs.openstack.org/keystone/latest/admin/auth-tokens.html)
