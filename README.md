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

### Configure Automation Hub access

`infra.aap_configuration` and `infra.aap_configuration_extended` both declare
hard dependencies on certified Red Hat collections (`ansible.controller`,
`ansible.hub`, `ansible.platform`) that are **only published on Automation
Hub**, not on the public Ansible Galaxy. `ansible.cfg` in this repository
configures Automation Hub as a named `galaxy` server (`automation_hub`), but
you must still:

1. Edit the `url` in the `[galaxy_server.automation_hub]` section of
   `ansible.cfg` to point at your Automation Hub instance (on-prem/Private
   Automation Hub or Red Hat Cloud/Hosted Hub).
2. Export your token before installing collections:

   ```bash
   export ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN="<your-token>"
   ```

   Ansible only picks this env var up because `automation_hub` is declared as
   a named server in `ansible.cfg`'s `server_list` — the env var alone, with
   no matching server definition, is silently ignored and collection installs
   will fall back to the public Galaxy, which does **not** have
   `ansible.controller` (this is the cause of `NOTINSTALLED` /
   `Check ... ansible.controller is installed` failures at playbook runtime).
3. If you're using the Red Hat Cloud/Hosted Hub (console.redhat.com), also
   uncomment `auth_url` in `ansible.cfg` so the offline token can be exchanged
   for an access token via SSO.

### Install required collections

Install collections from `collections/requirements.yml`:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

`collections/requirements.yml` includes:

- `infra.aap_configuration`
- `infra.aap_configuration_extended`
- `ansible.controller`, `ansible.hub`, `ansible.platform` (certified
  dependencies, sourced from `automation_hub`)

Verify `ansible.controller` actually installed before running playbooks:

```bash
ansible-galaxy collection list ansible.controller
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
