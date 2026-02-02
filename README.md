# LiteMaaS Virtual Keys Collection

Generic Ansible collection for managing LiteLLM virtual API keys. Create, delete, and manage virtual keys for single users or multi-user workshops via the LiteLLM API.

## Features

- **Single-user key provisioning**: Create individual API keys for personal use
- **Multi-user orchestration**: Provision keys for workshops with 100+ participants
- **Idempotent operations**: Safe to run multiple times
- **State-based management**: Use `state: present` or `state: absent`
- **Flexible configuration**: Customize models, duration, budgets, and metadata

## Installation

```bash
ansible-galaxy collection install litemaas.virtual_keys
```

## Quick Start

### Create a Single Key (OpenShift SSO)

If your LiteLLM uses "Login with OpenShift" SSO:

```yaml
---
- name: Create LiteLLM Virtual Key
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create virtual key
      ansible.builtin.include_role:
        name: litemaas.virtual_keys.manage_keys
      vars:
        litellm_vkey_state: present
        litellm_vkey_api_url: "https://litellm.apps.cluster.com"
        litellm_vkey_use_openshift_token: true  # Auto-detect OpenShift token
        litellm_vkey_alias: "my-dev-project"
        litellm_vkey_user_id: "developer@example.com"
        litellm_vkey_models:
          - "openai/granite-3-2-8b-instruct"
        litellm_vkey_duration: "7d"
```

Prerequisites: `oc login https://api.cluster.com:6443` first

### Create a Single Key (Master Key)

If you have a master key:

```yaml
---
- name: Create LiteLLM Virtual Key
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create virtual key
      ansible.builtin.include_role:
        name: litemaas.virtual_keys.manage_keys
      vars:
        litellm_vkey_state: present
        litellm_vkey_api_url: "https://litellm.example.com"
        litellm_vkey_master_key: "{{ lookup('env', 'LITELLM_MASTER_KEY') }}"
        litellm_vkey_alias: "my-dev-project"
        litellm_vkey_user_id: "developer@example.com"
        litellm_vkey_models:
          - "openai/granite-3-2-8b-instruct"
        litellm_vkey_duration: "7d"
```

### Create Workshop Keys (Multi-User)

```yaml
---
- name: Create Workshop Keys
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Create keys for all participants
      ansible.builtin.include_role:
        name: litemaas.virtual_keys.manage_keys
      vars:
        litellm_vkey_state: present
        litellm_vkey_api_url: "https://litellm.example.com"
        litellm_vkey_master_key: "{{ lookup('env', 'LITELLM_MASTER_KEY') }}"
        litellm_vkey_alias: "ai-workshop-2024"
        litellm_vkey_multi_user: true
        litellm_vkey_user_count: 25
        litellm_vkey_user_prefix: "participant"
        litellm_vkey_user_domain: "workshop.example.com"
        litellm_vkey_models:
          - "openai/granite-3-2-8b-instruct"
        litellm_vkey_duration: "3d"
```

### Delete a Key

```yaml
---
- name: Delete Virtual Key
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Delete virtual key
      ansible.builtin.include_role:
        name: litemaas.virtual_keys.manage_keys
      vars:
        litellm_vkey_state: absent
        litellm_vkey_api_url: "https://litellm.example.com"
        litellm_vkey_master_key: "{{ lookup('env', 'LITELLM_MASTER_KEY') }}"
        litellm_vkey_alias: "my-dev-project"
```

## Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `litellm_vkey_api_url` | LiteLLM API URL | `https://litellm.example.com` |
| `litellm_vkey_alias` | Unique key identifier/alias | `my-project` |
| `litellm_vkey_models` | List of models to enable | `["openai/gpt-4"]` |

### Authentication (Choose ONE Method)

**Method 1: Master Key (If available)**

| Variable | Description | Example |
|----------|-------------|---------|
| `litellm_vkey_master_key` | LiteLLM master API key | `sk-xxxxx` |

Best for: Non-SSO deployments or when you have admin-provided master key.

**Method 2: OpenShift Token (Recommended for SSO deployments)**

| Variable | Description | Example |
|----------|-------------|---------|
| `litellm_vkey_use_openshift_token` | Auto-detect OpenShift token | `true` |
| `litellm_vkey_openshift_token` | Manual OpenShift token (optional) | `sha256~xxxxx` |

Best for: LiteLLM deployed on OpenShift with "Login with OpenShift" SSO.

**How it works:**
1. If `litellm_vkey_openshift_token` is provided, uses that token
2. Otherwise tries `oc whoami -t` (if you're logged in with `oc login`)
3. Otherwise reads from `/var/run/secrets/kubernetes.io/serviceaccount/token` (if running in a pod)

**Method 3: Username/Password (Legacy)**

| Variable | Description | Example |
|----------|-------------|---------|
| `litellm_vkey_username` | LiteLLM username | `admin` |
| `litellm_vkey_password` | LiteLLM password | `password123` |

**Important Notes:**
- Username/password authentication requires the LiteLLM `/login` endpoint to be available
- **Does not work with SSO** (OAuth/OIDC) - those require browser-based authentication flows
- For OpenShift SSO deployments, use Method 2 (OpenShift Token)

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `litellm_vkey_state` | `present` | Key state: `present` or `absent` |
| `litellm_vkey_duration` | `30d` | Key validity period (3d, 7d, 30d, 90d) |
| `litellm_vkey_user_id` | `{{ litellm_vkey_alias }}` | User identifier (email/username) |
| `litellm_vkey_max_budget` | `null` | Budget limit (null = unlimited) |
| `litellm_vkey_metadata` | `{}` | Custom metadata dictionary |
| `litellm_vkey_multi_user` | `false` | Enable multi-user mode |
| `litellm_vkey_user_count` | `1` | Number of users (multi-user mode) |
| `litellm_vkey_user_prefix` | `user` | Username prefix (user1, user2...) |
| `litellm_vkey_user_domain` | `example.com` | Email domain for users |

See [`roles/manage_keys/defaults/main.yml`](roles/manage_keys/defaults/main.yml) for complete variable reference.

## Return Values

The role sets a fact `litellm_vkey_result` containing:

### Single-User Result

```yaml
litellm_vkey_result:
  key: "sk-abc123..."
  alias: "my-project"
  key_id: "token_hash"
  api_endpoint: "https://litellm.example.com"
  api_base_url: "https://litellm.example.com/v1"
  models: ["openai/gpt-4"]
  duration: "7d"
  user_id: "developer@example.com"
  state: "present"
  changed: true
```

### Multi-User Result

```yaml
litellm_vkey_result:
  api_endpoint: "https://litellm.example.com"
  api_base_url: "https://litellm.example.com/v1"
  models: ["openai/gpt-4"]
  duration: "3d"
  user_count: 3
  state: "present"
  changed: true
  keys:
    - user: "participant1"
      key: "sk-..."
      alias: "workshop-participant1"
      key_id: "..."
      user_id: "participant1@workshop.example.com"
```

## Examples

See [`playbooks/examples/`](playbooks/examples/) for complete example playbooks:

- [`create_key_openshift_sso.yml`](playbooks/examples/create_key_openshift_sso.yml) - **OpenShift SSO authentication** (recommended for OpenShift deployments)
- [`create_single_key.yml`](playbooks/examples/create_single_key.yml) - Single-user key creation (master key)
- [`delete_single_key.yml`](playbooks/examples/delete_single_key.yml) - Delete a key
- [`create_workshop_keys.yml`](playbooks/examples/create_workshop_keys.yml) - Multi-user workshop provisioning

## License

MIT

## Author

Prakhar Srivastava (psrivast@redhat.com)

## Repository

https://github.com/rhpds/litemaas.virtual_keys

## Issues

https://github.com/rhpds/litemaas.virtual_keys/issues
