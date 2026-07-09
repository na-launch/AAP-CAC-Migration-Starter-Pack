# AAP-CaC (AAP 2.4/AWX -> AAP 2.5/2.6) - Migration Starter Pack

> **Important support notice**  
> This repository is a **community starting point** for moving Automation Controller objects that are exportable via **Config-as-Code (CaC)** from **AAP 2.4 or AWX** into **AAP 2.5+**.  
> It is **not** an official or supported migration path from Red Hat. Validate everything in non-production first.

> **Credential secrets cannot be migrated**  
> CaC does **not** export secret values. During import, credential objects are created without their passwords/tokens/private keys. You must re-enter secrets manually in AAP 2.5 (or seed them via your secret manager/API) after import.

## What this repo contains

- `export_old.yml` - exports CaC data from **AAP <= 2.4 / AWX**
- `export_new.yml` - exports CaC data from **AAP 2.5+**
- `import_new.yml` - imports exported CaC bundle into **AAP 2.5+** in dependency-safe order
- `vars.yml` - base variables file to customize
- `collections/` - pinned collections for repeatable runs

## Prerequisites

- Ansible on your runner host
- API/network access to source and destination controllers
- Admin-level credentials/tokens
- Non-production environment for first runs (recommended)

Install required collections:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Configure variables

Use a layered vars approach:

- vars.yml for non-sensitive defaults
- vars.vault.yml (encrypted) for secrets

1) Create/edit vars.yml (non-secret values)

```
# Source (AAP <= 2.4 / AWX)
controller_username: "admin"
controller_hostname: "controller.example.com" # no http(s) prefix
controller_api_plugin: awx.awx.controller_api

# Target (AAP 2.5+)
aap_username: "admin"
aap_hostname: "aap.example.com" # no http(s) prefix

# General
export_organization: "{{ default(None) }}"
output_path: "./exports/{{ export_organization }}"
flatten_output: true
aap_validate_certs: "false"
controller_validate_certs: "false"

# Logging/debug

controller_configuration_credentials_secure_logging: "false"
cas_secure_logging: "false"
```

2) Create encrypted vars.vault.yml (secret values)


`ansible-vault create vars.vault.yml`

Example Contents:

```
controller_password: "<source-admin-password>"
aap_password: "<target-admin-password>"

# Optional token-based auth if your playbooks support it
controller_oauthtoken: "<optional-source-token>"
aap_oauthtoken: "<optional-target-token>"
```

3) Run playbooks with both var files

```
# Export from <=2.4 / AWX
ansible-playbook export_old.yml -e @vars.yml -e @vars.vault.yml --ask-vault-pass -e export_organization="<Org Name>"

# Export from 2.5+
ansible-playbook export_new.yml -e @vars.yml -e @vars.vault.yml --ask-vault-pass

# Import into 2.5+
ansible-playbook import_new.yml -e @vars.yml -e @vars.vault.yml --ask-vault-pass
```

## Typical object coverage

- Organizations, teams, users
- Credential types, credentials (metadata only)
- Projects, inventories/groups/hosts, inventory sources
- Execution environments, notification templates
- Job templates, workflow job templates, schedules, RBAC mappings

### Post-import required actions

- Re-enter credential secrets
- Re-validate SCM credential links and project sync behavior
- Run test jobs/workflows to confirm parity

## Troubleshooting

- 401/403: verify credentials/tokens and *_validate_certs
- Project sync failures: rebind SCM credentials/secrets
- Credential test failures: re-enter credential secret fields
- Idempotency/re-runs: repeated imports should converge, but validate object-by-object

### Known module compatibility issue

If import fails with:
- couldn't resolve module/action 'ansible.controller.application'

then your runtime likely lacks that module while the dispatch flow is attempting application roles. Workarounds:

- exclude application roles in dispatcher config, or
- gate application import behind an opt-in variable/tag

## FAQ

**Is this an official Red Hat migration path?**
No.

**Why are passwords/tokens not exported in CaC?**
Secret values are protected at rest and intentionally not emitted in exports.
