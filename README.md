# snow-eda-content

Event-Driven Ansible (EDA) content for integrating with ServiceNow ITSM. This repository provides:

- An EDA rulebook that watches ServiceNow records and dispatches automation
- Playbooks to process incidents and change requests
- Roles to create/update/close ServiceNow tickets and to enrich events with RITM data
- An Execution Environment (EE) build context for portable, reproducible runs

## What this does

- Monitors ServiceNow `incident` and `change_request` tables for updates
- Automatically transitions and implements change requests (default or patch-specific)
- Creates change requests based on incidents that meet criteria
- Posts implementation progress and scheduling notes back to ServiceNow
- Optionally enriches events with Service Catalog RITM variables

## Repository layout

- `rulebooks/records.yml` — main EDA rulebook sourcing ServiceNow records and dispatching jobs
- `playbooks/` — automation targets invoked by EDA (implementation, scheduling, alerts, creation)
- `roles/servicenow_ticket/` — thin role to create/update/close/find ServiceNow tickets
- `roles/servicenow_ritm_retrieve_eda/` — enrich Service Catalog request items (RITMs)
- `ee_build/` — Execution Environment build files (Containerfile + decision_environment.yml)

## Requirements

- Red Hat Ansible Automation Platform with EDA (Controller/EDA recommended)
- Collections:
  - `servicenow.itsm`
  - `ansible.eda`
- ServiceNow credentials configured in Controller/EDA (preferred) or via environment variables
  - Typically `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` (and optionally OAuth if used)

## EDA rulebook overview

Rulebook: `rulebooks/records.yml`

Sources:
- ServiceNow incidents: polls `incident`
- ServiceNow change requests in implementation: polls `change_request` with `state=-1`
- ServiceNow new change requests to transition: polls `change_request` with query `state=-5^short_descriptionLIKERelated to Incident`

Rules (high level):
- Transition new change requests to implementation → runs Controller job template `SNOW: Implement - Default`
- Enrich new incident (sys_mod_count == 0) → runs `EDA Enrich ServiceNow REQ`
- Create a software change request for specific incidents → runs `SNOW: Create Software Change Request`
- Dispatch implement-phase change requests:
  - Patch-related → `SNOW: Implement - Patch Deployment`
  - Otherwise → `SNOW: Implement - Default`

Notes:
- Rules use a guard comment marker `EDA-IMPLEMENT-PROCESSED` in work notes to avoid duplicate processing
- Events are posted back to EDA when `post_events: true` is set on the job template call

## Playbooks

Most playbooks assume they are launched by EDA and read fields from `ansible_eda.event`.

- `playbooks/create_software_change_request.yml`
  - Creates a `change_request` from an incoming incident; links back to the incident

- `playbooks/implement_default.yml`
  - Default implement-phase handler; gathers minimal host context (e.g., firewalld status), transitions CR to review then closed, and updates related incident notes

- `playbooks/implement_patch_deployment.yml`
  - Patch-specific handler; keeps CR in implementation, adds work notes, and leaves a placeholder where your patch automation should run

- `playbooks/process_scheduled_change_request.yml`
  - Moves a scheduled CR into implementation and annotates work notes

- `playbooks/notify_change_request_scheduled.yml`
  - Posts scheduling details (planned start/end, assignment) as work notes

- `playbooks/alert_change_request_starting_soon.yml`
  - Calculates time to planned start and posts alert work notes

- `playbooks/enrich_sn_ticket.yml`
  - Runs RITM enrichment role to gather catalog variables and requester email

- `playbooks/create_ticket_snow.yml`
  - Uses `roles/servicenow_ticket` with `servicenow_ticket: create`

Additional examples are provided (e.g., `servicenow_ticket_update_logs.yml`, `track_change_implementation_progress.yml`, `snow_fix_disk_space.yml`, `snow_reinstall_agent.yml`).

## Roles

### `roles/servicenow_ticket`

Dispatch role controlled by `servicenow_ticket` variable:

- `create` → `tasks/servicenow_create.yml`
- `createeda` → `tasks/servicenow_create_eda.yml`
- `createlogs` → `tasks/servicenow_create_logs.yml`
- `update` → `tasks/servicenow_update.yml`
- `updatelogs` → `tasks/servicenow_update_logs.yml`
- `close` → `tasks/servicenow_close.yml`
- `find` → `tasks/servicenow_find.yml`

All tasks leverage `servicenow.itsm` modules; credentials must be provided by Controller/EDA or environment.

### `roles/servicenow_ritm_retrieve_eda`

Performs batched lookups for Service Catalog RITMs related to a given `req_sys_id` and builds an `enriched_event` structure that includes:

- `ritm_details` with nested `variables: [{ name, value }, ...]`
- `ritm_variables` dictionary keyed by RITM sys_id
- `user` (requester email) in aggregated stats via `set_stats`

Outputs are available to downstream tasks via `set_stats` and `ansible_facts`.

## Execution Environment (EE)

EE build context is under `ee_build/` and targets AAP 2.5 minimal DE base image.

- `ee_build/context/Containerfile` — multi-stage build using ansible-builder generated artifacts
- `ee_build/decision_environment.yml` — declares required collections and RPMs

Build options:

Using Podman directly:

```
podman build -t snow-eda-ee -f ee_build/context/Containerfile ee_build/context
```

Or with `ansible-builder` v3 style (when applicable to your environment):

```
ansible-builder build -v3 -t snow-eda-ee -f ee_build/context/Containerfile -c ee_build/context
```

Assign the built image to your Controller/EDA projects and job templates.

## Running with EDA

Controller/EDA is the recommended runtime:

1. Import this repo as a Project in Controller
2. Create Job Templates named to match the rulebook actions:
   - `SNOW: Implement - Default`
   - `SNOW: Implement - Patch Deployment`
   - `EDA Enrich ServiceNow REQ`
   - `SNOW: Create Software Change Request`
3. Point each Job Template at the corresponding playbook in `playbooks/`
4. Attach a ServiceNow credential (or set `SN_HOST`, `SN_USERNAME`, `SN_PASSWORD` as env vars)
5. In EDA, create a Rulebook activation using `rulebooks/records.yml` and the same EE

Local EDA testing is also possible (requires credentials in env):

```
export SN_HOST="https://your-instance.service-now.com"
export SN_USERNAME="api_user"
export SN_PASSWORD="******"

ansible-rulebook -r rulebooks/records.yml -i localhost,
```

## Testing playbooks manually

Some playbooks read from `ansible_eda.event`. You can simulate this by passing `extra_vars`:

```
ansible-playbook playbooks/implement_default.yml \
  -e "ansible_eda={'event': {'sys_id': 'abcd1234abcd1234abcd1234abcd1234', 'number': 'CHG0001234', 'state': 'implementation', 'short_description': 'Test change', 'priority': '3', 'urgency': '3', 'impact': '3'}}"
```

Adjust fields as required by the target playbook.

## Configuration and credentials

The `servicenow.itsm` collection can read credentials from Controller-provided credentials or environment variables. Minimal env variables:

- `SN_HOST` — ServiceNow instance URL
- `SN_USERNAME` — API username
- `SN_PASSWORD` — API password

Alternatively, use Controller’s built-in ServiceNow credential type and attach it to your Job Templates.

## Contributing

- Use clear commit messages and keep playbooks idempotent where feasible
- Prefer small, composable roles for new functionality
- Follow existing code style and formatting

## License

SPDX-License-Identifier: MIT-0
