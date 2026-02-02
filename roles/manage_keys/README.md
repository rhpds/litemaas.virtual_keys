# manage_keys Role

Ansible role for managing LiteLLM virtual API keys.

## Description

This role provides a generic interface for creating and deleting LiteLLM virtual keys. It supports both single-user and multi-user provisioning scenarios.

## Requirements

- Ansible 2.14 or higher
- Access to a LiteLLM API endpoint
- LiteLLM master key with permission to create/delete virtual keys

## Role Variables

### Required

- `litellm_vkey_api_url` - LiteLLM API URL
- `litellm_vkey_master_key` - LiteLLM master key
- `litellm_vkey_alias` - Unique key identifier
- `litellm_vkey_models` - List of models (required for creation)

### Optional

See [`defaults/main.yml`](defaults/main.yml) for complete variable reference.

## Dependencies

None

## Example Playbook

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
        litellm_vkey_master_key: "sk-xxxxx"
        litellm_vkey_alias: "my-project"
        litellm_vkey_models:
          - "openai/gpt-4"
        litellm_vkey_duration: "7d"
```

## Return Values

The role sets a fact `litellm_vkey_result` with key details.

## License

MIT

## Author

Prakhar Srivastava
