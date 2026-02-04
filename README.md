# LiteLLM Virtual Keys Collection

Generic Ansible collection for managing LiteLLM virtual API keys. Create and delete virtual keys for single users or multi-user workshops via the LiteLLM API.

## Features

- **Single-user key provisioning**: Create individual API keys for personal use
- **Multi-user orchestration**: Provision keys for workshops with 100+ participants
- **Quota management**: Set budget limits and expiration times
- **Idempotent operations**: Safe to run multiple times
- **State-based management**: Use `state: present` or `state: absent`

## Installation

```bash
ansible-galaxy collection install litemaas.virtual_keys
```

## Quick Start

### Create a Single Key

**Create a simple playbook (create_key.yml):**
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
          - "gpt-4"
          - "gpt-3.5-turbo"
        litellm_vkey_duration: "7d"
        litellm_vkey_max_budget: 100
```

**Run with command-line parameters:**
```bash
export LITELLM_MASTER_KEY="sk-xxxxx"

ansible-playbook create_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="my-project-name" \
  -e litellm_vkey_user_id="developer@example.com" \
  -e litellm_vkey_models='["gpt-4"]' \
  -e litellm_vkey_duration="7d" \
  -e litellm_vkey_max_budget=100
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
          - "gpt-4"
        litellm_vkey_duration: "3d"
        litellm_vkey_max_budget: 50
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
| `litellm_vkey_master_key` | LiteLLM master API key | `sk-xxxxx` |
| `litellm_vkey_alias` | Unique key identifier/alias | `my-project` |
| `litellm_vkey_models` | List of models to enable | `["gpt-4", "gpt-3.5-turbo"]` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `litellm_vkey_state` | `present` | Key state: `present` or `absent` |
| `litellm_vkey_duration` | `30d` | Key validity period (3d, 7d, 30d, 90d) |
| `litellm_vkey_user_id` | `{{ litellm_vkey_alias }}` | User identifier (email/username) |
| `litellm_vkey_max_budget` | `null` | Budget limit in dollars (null = unlimited) |
| `litellm_vkey_metadata` | `{}` | Custom metadata dictionary |
| `litellm_vkey_multi_user` | `false` | Enable multi-user mode |
| `litellm_vkey_user_count` | `1` | Number of users (multi-user mode) |
| `litellm_vkey_user_prefix` | `user` | Username prefix (user1, user2...) |
| `litellm_vkey_user_domain` | `example.com` | Email domain for users |

See [`roles/manage_keys/defaults/main.yml`](roles/manage_keys/defaults/main.yml) for complete variable reference.

## Getting the Master Key

### From OpenShift/Kubernetes Secret

```bash
# Get the master key from your LiteLLM deployment
kubectl get secret litellm-secret -n litellm -o jsonpath='{.data.LITELLM_MASTER_KEY}' | base64 -d

# Or with oc (OpenShift)
oc get secret litellm-secret -n litellm -o jsonpath='{.data.LITELLM_MASTER_KEY}' | base64 -d

# Set as environment variable
export LITELLM_MASTER_KEY="sk-xxxxx"
```

### Use in Playbook

```yaml
vars:
  litellm_vkey_master_key: "{{ lookup('env', 'LITELLM_MASTER_KEY') }}"
```

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
  models: ["gpt-4"]
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
  models: ["gpt-4"]
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

- [`create_single_key.yml`](playbooks/examples/create_single_key.yml) - Single-user key creation
- [`delete_single_key.yml`](playbooks/examples/delete_single_key.yml) - Delete a key
- [`create_workshop_keys.yml`](playbooks/examples/create_workshop_keys.yml) - Multi-user workshop provisioning

## Common Use Cases

### Create Multiple Keys with Different Names

```bash
# Development key
ansible-playbook create_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="dev-environment" \
  -e litellm_vkey_models='["gpt-3.5-turbo"]'

# Production key
ansible-playbook create_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="prod-environment" \
  -e litellm_vkey_models='["gpt-4"]' \
  -e litellm_vkey_max_budget=1000

# Personal key
ansible-playbook create_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="john-personal" \
  -e litellm_vkey_user_id="john@example.com" \
  -e litellm_vkey_models='["gpt-4"]'
```

### Replace an Existing Key

If you get "key already exists" error, create a delete playbook (delete_key.yml):

```yaml
---
- name: Delete LiteLLM Virtual Key
  hosts: localhost
  gather_facts: false

  tasks:
    - name: Delete virtual key
      ansible.builtin.include_role:
        name: litemaas.virtual_keys.manage_keys
      vars:
        litellm_vkey_state: absent
        litellm_vkey_api_url: "{{ litellm_vkey_api_url }}"
        litellm_vkey_master_key: "{{ lookup('env', 'LITELLM_MASTER_KEY') }}"
        litellm_vkey_alias: "{{ litellm_vkey_alias }}"
```

Then:
```bash
# 1. Delete the old key
ansible-playbook delete_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="my-project"

# 2. Create new key with same name
ansible-playbook create_key.yml \
  -e litellm_vkey_api_url="https://litellm.example.com" \
  -e litellm_vkey_alias="my-project" \
  -e litellm_vkey_models='["gpt-4"]'
```

## Quota Management

Control spending and access with quotas:

```yaml
vars:
  litellm_vkey_max_budget: 100      # $100 spending limit
  litellm_vkey_duration: "30d"      # Expires in 30 days
  litellm_vkey_models:              # Only these models
    - "gpt-4"
    - "gpt-3.5-turbo"
```

**How it works:**
- LiteLLM tracks token usage per key
- Automatically blocks requests when budget exceeded
- Key auto-expires after duration period
- Users only have access to specified models

## License

MIT

## Author

Prakhar Srivastava (psrivast@redhat.com)

## Repository

https://github.com/rhpds/litemaas.virtual_keys

## Issues

https://github.com/rhpds/litemaas.virtual_keys/issues
