# snow-eda-content

Event-Driven Ansible (EDA) content for integrating with ServiceNow ITSM. This repository provides a complete automation framework for monitoring and responding to ServiceNow incidents and change requests.

## Overview

This repository provides:

- **EDA Rulebook** — Monitors ServiceNow tables and dispatches automation based on record state and content
- **Playbooks** — Comprehensive playbook collection for processing incidents, change requests, and operational tasks
- **Roles** — Reusable roles for ServiceNow ticket lifecycle management and RITM enrichment
- **Execution Environment** — Complete containerized runtime with all dependencies for AAP 2.5

## Key Capabilities

- **Real-time Monitoring** — Continuously polls ServiceNow `incident` and `change_request` tables
- **Automated Change Management** — Automatically transitions and implements change requests with deduplication
- **Incident-to-Change Workflow** — Creates change requests from incidents matching specific criteria
- **Patch Management Integration** — Special handling for patch deployment workflows
- **Service Catalog Integration** — Enriches events with RITM (Request Item) variables and requester details
- **Progress Tracking** — Updates ServiceNow with implementation progress, alerts, and scheduling information
- **Operational Remediation** — Includes example playbooks for disk space cleanup and agent reinstallation

## Repository Layout

```
snow-eda-content/
├── rulebooks/
│   └── records.yml                              # Main EDA rulebook with 5 event-driven rules
├── playbooks/
│   ├── create_software_change_request.yml       # Create CR from incident
│   ├── implement_default.yml                    # Default CR implementation handler
│   ├── implement_patch_deployment.yml           # Patch-specific CR handler
│   ├── implement_firewalld_fix.yml              # Firewalld remediation automation
│   ├── process_scheduled_change_request.yml     # Transition scheduled CRs
│   ├── notify_change_request_scheduled.yml      # Post scheduling notifications
│   ├── alert_change_request_starting_soon.yml   # Send deadline alerts
│   ├── track_change_implementation_progress.yml # Monitor CR progress with metrics
│   ├── close_cr_incident.yml                    # Close CR and linked incident
│   ├── enrich_sn_ticket.yml                     # RITM enrichment workflow
│   ├── create_ticket_snow.yml                   # Generic ticket creation
│   ├── servicenow_ticket_update_logs.yml        # Add job logs to tickets
│   ├── snow_fix_disk_space.yml                  # Disk cleanup remediation
│   └── snow_reinstall_agent.yml                 # Agent reinstallation workflow
├── roles/
│   ├── servicenow_ticket/                       # Multi-mode ticket management role
│   │   ├── tasks/
│   │   │   ├── main.yml                         # Dispatcher based on mode
│   │   │   ├── servicenow_create.yml            # Create tickets
│   │   │   ├── servicenow_create_eda.yml        # Create from EDA events
│   │   │   ├── servicenow_create_logs.yml       # Create with logs
│   │   │   ├── servicenow_update.yml            # Update tickets
│   │   │   ├── servicenow_update_logs.yml       # Update with logs
│   │   │   ├── servicenow_close.yml             # Close tickets
│   │   │   └── servicenow_find.yml              # Find/search tickets
│   │   └── defaults/main.yml
│   └── servicenow_ritm_retrieve_eda/            # Service Catalog RITM enrichment
│       └── tasks/main.yml
├── files/
│   └── firewalld/
│       └── custom.xml                           # Firewall configuration template
└── ee_build/
    ├── decision_environment.yml                 # Collection & system dependencies
    └── context/
        ├── Containerfile                        # Multi-stage EE build
        └── _build/                              # ansible-builder artifacts
```

## Requirements

### Platform
- Red Hat Ansible Automation Platform 2.5+ with Event-Driven Ansible
- AAP Controller + EDA Controller (recommended) or standalone `ansible-rulebook` CLI

### Collections
- `servicenow.itsm` — ServiceNow ITSM integration
- `ansible.eda` — Event-Driven Ansible source plugins

### Credentials
ServiceNow API credentials via Controller credential type or environment variables:
- `SN_HOST` — ServiceNow instance URL (e.g., `https://dev12345.service-now.com`)
- `SN_USERNAME` — API user with appropriate table permissions
- `SN_PASSWORD` — API password or token

### ServiceNow Permissions
The API user requires read/write access to:
- `incident` table
- `change_request` table
- `sc_request` table (for RITM enrichment)
- `sc_req_item` table (for RITM enrichment)
- `sys_user` table (for user lookups)

## EDA Rulebook Overview

### File: `rulebooks/records.yml`

The rulebook defines three ServiceNow event sources and five conditional rules for automated responses.

### Event Sources

| Source Name | Table | Interval | Query Filter | Description |
|-------------|-------|----------|--------------|-------------|
| Watch for any updated incidents | `incident` | 5s | None | Monitors all incident updates |
| Monitor ServiceNow change requests in implementation | `change_request` | 10s | `state=-1` | Tracks CRs in implementation phase |
| Monitor new change requests to transition to implementation | `change_request` | 10s | `state=-5^short_descriptionLIKERelated to Incident` | Finds new CRs ready for transition |

### Automation Rules

#### 1. Transition new change requests to implementation
- **Condition**: CR state is "new" (-5) AND short description contains "Related to Incident" AND no "EDA-IMPLEMENT-PROCESSED" marker
- **Action**: Run job template `SNOW: Implement - Default`
- **Purpose**: Automatically begin implementation for incident-related change requests

#### 2. Enrich new incident received
- **Condition**: Incident with `sys_mod_count == "0"` (brand new incident)
- **Action**: Run job template `SNOW: EDA Enrich ServiceNow REQ`
- **Purpose**: Gather Service Catalog RITM variables and requester information for new incidents

#### 3. Create a change request if needed
- **Condition**: Active incident with `category == "Software"` AND `short_description == "test service failure"`
- **Action**: Run job template `SNOW: Create Software Change Request`
- **Purpose**: Automatically create CRs for specific incident patterns (customizable criteria)

#### 4. Dispatch patch-related incidents in progress
- **Condition**: Active incident containing "patch" in short description AND no processing marker
- **Action**: Run workflow template `SNOW: Implement - Patch Deployment`
- **Purpose**: Special handling for patch deployment workflows

#### 5. Dispatch default handler for change requests in implementation
- **Condition**: CR in "implementation" state (-1) AND no processing marker
- **Action**: Run workflow template `SNOW: Implement - CR`
- **Purpose**: Execute standard CR implementation workflow

### Deduplication Strategy

All rules use the marker `EDA-IMPLEMENT-PROCESSED` in `comments_and_work_notes` fields to prevent duplicate processing. Playbooks add this marker after handling records.

## Playbooks

All playbooks read event data from `ansible_eda.event` structure when invoked by EDA Controller.

### Change Request Implementation

#### `implement_default.yml`
**Purpose**: Standard change request implementation workflow

**Workflow**:
1. Extracts CR context from event data
2. Retrieves full CR details from ServiceNow
3. Parses linked incident identifiers from CR description
4. Performs implementation tasks (customizable)
5. Transitions CR to "review" then "closed" state
6. Updates linked incident with closure notes
7. Adds `EDA-IMPLEMENT-PROCESSED` marker

**Variables**: Accepts all standard CR fields via `ansible_eda.event`

#### `implement_patch_deployment.yml`
**Purpose**: Specialized handler for patch deployment change requests

**Workflow**:
1. Supports both CR and incident as source events
2. Keeps CR in "implementation" state (not auto-closed)
3. Adds work notes with patch deployment status
4. Includes placeholder for patch automation integration
5. Updates both CR and incident with progress
6. Adds processing marker to prevent re-runs

**Variables**: Accepts CR and incident fields, auto-detects source type

#### `implement_firewalld_fix.yml`
**Purpose**: Automated firewall rule remediation

**Features**:
- Deploys custom firewall rules from `files/firewalld/custom.xml`
- Reloads firewalld service
- Updates ServiceNow with results

### Change Request Creation

#### `create_software_change_request.yml`
**Purpose**: Automatically create CR from ServiceNow incident

**Workflow**:
1. Receives incident data from EDA event
2. Queries ServiceNow for full incident details
3. Checks for existing linked CRs to avoid duplicates
4. Creates new CR with incident context embedded
5. Links CR back to original incident
6. Sets appropriate priority, urgency, impact from incident

**State**: Creates CR in "implementation" state (-1)
**Linking**: Embeds incident number and sys_id in CR description

### Change Request Lifecycle

#### `close_cr_incident.yml`
**Purpose**: Close change request and linked incident together

**Workflow**:
1. Determines if working with CR or incident (flexible input)
2. Transitions CR to "review" then "closed" with "successful" close code
3. Extracts linked incident identifiers from CR description
4. Closes linked incident with matching closure notes
5. Supports lookup by sys_id or number

**Variables**: `change_request_sys_id`, `incident_sys_id`, or `incident_number`

#### `process_scheduled_change_request.yml`
**Purpose**: Transition scheduled CRs into implementation

**Use Case**: Batch processing of scheduled maintenance windows

#### `track_change_implementation_progress.yml`
**Purpose**: Monitor and report CR implementation progress

**Features**:
- Calculates progress percentage based on start/end times
- Updates work notes with progress metrics
- Detects approaching deadlines (< 1 hour remaining)
- Sends alert work notes for deadline warnings
- Provides real-time progress visibility

**Variables**: Requires `start_date` and `planned_end_date` in CR event

### Notifications & Alerts

#### `notify_change_request_scheduled.yml`
**Purpose**: Post scheduling information to CR work notes

**Content**: Planned start/end times, assignment group, assigned to

#### `alert_change_request_starting_soon.yml`
**Purpose**: Send deadline alerts for upcoming CRs

**Logic**: Calculates time until planned start and posts alerts

### Service Catalog Integration

#### `enrich_sn_ticket.yml`
**Purpose**: Enrich incident with Service Catalog RITM data

**Workflow**:
1. Invokes `servicenow_ritm_retrieve_eda` role
2. Retrieves all RITMs linked to request sys_id
3. Fetches catalog variables for each RITM
4. Resolves requester user information
5. Publishes enriched data via `set_stats` for downstream tasks

**Output Structure**:
```yaml
enriched_event:
  req_sys_id: "<request_sys_id>"
  ritm_details:
    - sys_id: "<ritm_sys_id>"
      number: "RITM0001234"
      description: "..."
      variables:
        - name: "var_name"
          value: "var_value"
  ritm_variables:
    "<ritm_sys_id>":
      - name: "..."
        value: "..."
  user: "requester@example.com"
```

### Generic Ticket Operations

#### `create_ticket_snow.yml`
**Purpose**: Generic ticket creation using `servicenow_ticket` role

**Mode**: Uses role with `servicenow_ticket: create`

#### `servicenow_ticket_update_logs.yml`
**Purpose**: Append job execution logs to ticket work notes

**Mode**: Uses role with `servicenow_ticket: updatelogs`

### Operational Remediation

#### `snow_fix_disk_space.yml`
**Purpose**: Automated disk space cleanup remediation

**Features**:
- Cleans `/tmp`, `/var/tmp`, `/var/log`, `/var/cache`
- Removes files older than configurable retention periods
- Cleans package manager caches (yum/dnf/apt)
- Removes old kernel versions
- Vacuums systemd journal logs
- Reports before/after disk usage and space freed

**Variables**:
```yaml
cleanup_directories: [/tmp, /var/tmp, /var/log, /var/cache]
min_disk_space_gb: 2
log_retention_days: 30
cache_retention_days: 7
```

**Execution**: `hosts: "{{ hostname | default('all') }}"` with `become: true`

#### `snow_reinstall_agent.yml`
**Purpose**: Reinstall monitoring or management agents

**Use Case**: Remediate failed or corrupted agent installations

## Roles

### `roles/servicenow_ticket`

**Purpose**: Multi-mode dispatcher role for ServiceNow ticket lifecycle management

**Architecture**: Main dispatcher (`tasks/main.yml`) routes to specific task files based on `servicenow_ticket` variable.

#### Available Modes

| Mode | Task File | Description |
|------|-----------|-------------|
| `create` | `servicenow_create.yml` | Create new tickets (incident/change_request) |
| `createeda` | `servicenow_create_eda.yml` | Create tickets from EDA event structure |
| `createlogs` | `servicenow_create_logs.yml` | Create tickets and attach job logs |
| `update` | `servicenow_update.yml` | Update existing ticket fields |
| `updatelogs` | `servicenow_update_logs.yml` | Append job logs to ticket work notes |
| `close` | `servicenow_close.yml` | Close tickets with resolution details |
| `find` | `servicenow_find.yml` | Search/query tickets by criteria |

#### Usage Example

```yaml
- name: Update incident with work notes
  include_role:
    name: servicenow_ticket
  vars:
    servicenow_ticket: update
    ticket_sys_id: "abc123..."
    work_notes: "Automation in progress"
```

#### Integration

All task files use `servicenow.itsm` collection modules. Credentials are consumed from:
- AAP Controller credential injection (preferred)
- Environment variables (`SN_HOST`, `SN_USERNAME`, `SN_PASSWORD`)

### `roles/servicenow_ritm_retrieve_eda`

**Purpose**: Enrich ServiceNow requests with Service Catalog RITM (Request Item) variables and requester details

**Architecture**: Single comprehensive task file performing batched API queries with retry logic

#### Workflow

1. **Retrieve RITMs**: Query `sc_req_item` table for all items linked to `req_sys_id`
2. **Resolve User**: Lookup requester email from `sys_user` table
3. **Map Variables**: Query `sc_item_option_mtom` for variable-to-RITM mappings
4. **Fetch Values**: Retrieve variable details from `sc_item_option` table
5. **Resolve Names**: Get human-readable names from `item_option_new` table
6. **Build Structure**: Combine into nested enriched event structure
7. **Publish**: Export via `set_stats` for downstream consumption

#### Output Structure

```yaml
enriched_event:
  req_sys_id: "abc123..."
  ritm_details:
    - sys_id:
        value: "def456..."
      number: "RITM0001234"
      description: "Request for server access"
      variables:
        - name: "server_name"
          value: "prod-web-01"
        - name: "access_level"
          value: "read-only"
  ritm_variables:
    "def456...":
      - name: "server_name"
        value: "prod-web-01"
  user: "john.doe@example.com"
```

#### Retry Logic

- All API queries include `retries: 3` and `delay: 5`
- Handles eventual consistency in ServiceNow catalog subsystem
- Variable mappings use `ignore_errors: true` for graceful degradation

#### Consumption

Published data is available to subsequent tasks via:
- `ansible_facts.enriched_event`
- `hostvars[inventory_hostname]['enriched_event']` (from `set_stats`)

#### Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `req_sys_id` | Yes | ServiceNow request sys_id to enrich |
| `req_number` | No | Request number for logging (optional) |

## Execution Environment (EE)

### Overview

The repository includes a complete Decision Environment (DE) build context for AAP 2.5 Event-Driven Ansible.

### Files

| File | Purpose |
|------|---------|
| `ee_build/decision_environment.yml` | Defines collection and system dependencies |
| `ee_build/context/Containerfile` | Multi-stage container build definition |
| `ee_build/context/_build/` | ansible-builder generated artifacts (scripts, requirements) |

### Dependencies

#### Collections (from `decision_environment.yml`)
- `servicenow.itsm` — ServiceNow ITSM modules
- `ansible.eda` — Event-Driven Ansible source plugins

#### System Packages (RHEL 8)
- `pkgconf-pkg-config` — Build dependency resolution
- `systemd-devel` — Systemd integration headers
- `gcc` — C compiler for Python extensions
- `python3.11-devel` — Python development headers

### Build Architecture

The Containerfile uses a **multi-stage build**:

1. **Base Stage**: Install pip and core utilities
2. **Galaxy Stage**: Install Ansible collections and roles
3. **Builder Stage**: Introspect dependencies via `bindep` and prepare wheels
4. **Final Stage**: Assemble minimal runtime image with all dependencies

**Base Image**: `registry.redhat.io/ansible-automation-platform-25/de-minimal-rhel8:latest`

### Building the EE

#### Option 1: Direct Podman Build (Recommended)

```bash
cd /path/to/snow-eda-content
podman build -t snow-eda-ee:latest -f ee_build/context/Containerfile ee_build/context
```

#### Option 2: Using ansible-builder

```bash
cd ee_build/context
ansible-builder build -t snow-eda-ee:latest --container-runtime podman
```

### Pushing to Registry

```bash
# Tag for your private registry
podman tag snow-eda-ee:latest registry.example.com/snow-eda-ee:1.0

# Push to registry
podman push registry.example.com/snow-eda-ee:1.0
```

### Using in AAP

1. Navigate to **Execution Environments** in AAP Controller
2. Click **Add**
3. Provide image URL: `registry.example.com/snow-eda-ee:1.0`
4. Assign to Projects and Rulebook Activations in EDA Controller

## Deployment Guide

### AAP Controller/EDA Setup (Recommended)

#### 1. Create ServiceNow Credential

**Controller → Credentials → Add**

- **Name**: ServiceNow Production
- **Type**: ServiceNow (or Basic Auth if custom)
- **Host**: `https://yourinstance.service-now.com`
- **Username**: API service account
- **Password**: API password/token

#### 2. Import Project

**Controller → Projects → Add**

- **Name**: ServiceNow EDA Integration
- **SCM Type**: Git
- **SCM URL**: `<this_repo_url>`
- **Branch**: `main`
- **Update Revision on Launch**: ✓

#### 3. Create Job Templates

Create job templates matching rulebook action names:

| Template Name | Playbook | Credentials | Execution Environment |
|---------------|----------|-------------|----------------------|
| `SNOW: Implement - Default` | `playbooks/implement_default.yml` | ServiceNow Production | snow-eda-ee |
| `SNOW: Implement - Patch Deployment` | `playbooks/implement_patch_deployment.yml` | ServiceNow Production | snow-eda-ee |
| `SNOW: EDA Enrich ServiceNow REQ` | `playbooks/enrich_sn_ticket.yml` | ServiceNow Production | snow-eda-ee |
| `SNOW: Create Software Change Request` | `playbooks/create_software_change_request.yml` | ServiceNow Production | snow-eda-ee |

**Important**: Set `Prompt on Launch` for **Extra Variables** on all templates to accept EDA event data.

#### 4. Create Workflow Templates (Optional)

For `SNOW: Implement - CR` and `SNOW: Implement - Patch Deployment`, you can chain multiple job templates:

1. Create Workflow Template
2. Add nodes for sequential/parallel execution
3. Include job templates, approval nodes, or inventory updates

#### 5. Configure EDA Rulebook Activation

**EDA Controller → Rulebook Activations → Add**

- **Name**: ServiceNow Records Monitor
- **Project**: ServiceNow EDA Integration
- **Rulebook**: `rulebooks/records.yml`
- **Decision Environment**: snow-eda-ee
- **Credentials**: ServiceNow Production
- **Restart Policy**: Always
- **Extra Variables** (optional):
  ```yaml
  organization: SNOW
  ```

#### 6. Enable Activation

Click **Enable** to start monitoring ServiceNow tables.

### Local Testing (Development)

For local development and testing without AAP:

#### Prerequisites

```bash
pip install ansible ansible-rulebook ansible-runner
ansible-galaxy collection install servicenow.itsm ansible.eda
```

#### Set Credentials

```bash
export SN_HOST="https://dev12345.service-now.com"
export SN_USERNAME="api_user"
export SN_PASSWORD="your_password"
```

#### Run Rulebook

```bash
ansible-rulebook \
  --rulebook rulebooks/records.yml \
  --inventory localhost, \
  --verbose
```

#### Simulate Events

Mock ServiceNow events by creating test records in your ServiceNow instance or using the ServiceNow REST API.

## Testing Playbooks

### Manual Playbook Execution

Most playbooks expect `ansible_eda.event` structure. Simulate with `--extra-vars`:

```bash
# Test implement_default.yml
ansible-playbook playbooks/implement_default.yml \
  -e '{
    "ansible_eda": {
      "event": {
        "sys_id": "abcd1234abcd1234abcd1234abcd1234",
        "number": "CHG0001234",
        "state": "implementation",
        "short_description": "Test change for deployment",
        "description": "Incident Number: INC0012345\nIncident Sys ID: 1234567890abcdef1234567890abcdef",
        "priority": "3",
        "urgency": "3",
        "impact": "3"
      }
    }
  }'

# Test create_software_change_request.yml
ansible-playbook playbooks/create_software_change_request.yml \
  -e '{
    "ansible_eda": {
      "event": {
        "number": "INC0012345",
        "sys_id": "1234567890abcdef1234567890abcdef",
        "category": "Software",
        "short_description": "test service failure",
        "description": "Application service failed to start",
        "priority": "2",
        "urgency": "2",
        "impact": "2",
        "assignment_group": "IT Operations",
        "assigned_to": "admin"
      }
    }
  }'
```

### Controller Job Template Testing

In AAP Controller, launch job templates with **Extra Variables**:

```yaml
ansible_eda:
  event:
    sys_id: "test_sys_id_123"
    number: "CHG0001234"
    state: "implementation"
    short_description: "Test change request"
```

## Configuration

### Environment Variables

All playbooks support environment-based ServiceNow configuration:

```bash
export SN_HOST="https://yourinstance.service-now.com"
export SN_USERNAME="api_service_account"
export SN_PASSWORD="secure_password_or_token"
```

### Controller Credential Injection (Preferred)

When using AAP Controller credentials, variables are auto-injected:
- Credentials are encrypted at rest
- No plaintext passwords in playbooks
- Centralized credential rotation

### Customizing Rulebook Conditions

Edit `rulebooks/records.yml` to adjust trigger conditions:

```yaml
# Example: Change CR creation criteria
- name: Create a change request if needed
  condition: |
    event.sys_class_name == "incident" 
    and event.state not in ["6", "7"]
    and event.category == "Hardware"
    and event.priority in ["1", "2"]
  action:
    run_job_template:
      name: "SNOW: Create Hardware Change Request"
```

### Adjusting Poll Intervals

Modify `interval` values in source definitions:

```yaml
sources:
  - name: Watch for any updated incidents
    servicenow.itsm.records:
      table: incident
      interval: 30  # Increase from 5 to 30 seconds
```

## Troubleshooting

### Rulebook Not Triggering

1. **Check EDA Activation Logs** in EDA Controller
2. **Verify Credentials** are attached to activation
3. **Test ServiceNow Connectivity**:
   ```bash
   curl -u "$SN_USERNAME:$SN_PASSWORD" \
     "$SN_HOST/api/now/table/incident?sysparm_limit=1"
   ```
4. **Review Condition Logic** — Add `ansible.builtin.debug` tasks to playbooks

### Duplicate Processing

- Ensure playbooks add `EDA-IMPLEMENT-PROCESSED` marker
- Check `comments_and_work_notes` field in ServiceNow records
- Review rulebook conditions for marker exclusion

### Job Template Not Found

- Verify exact template names match rulebook actions
- Check organization name matches (`organization: SNOW`)
- Ensure templates have **Prompt on Launch** for Extra Variables

### Permission Denied in ServiceNow

- ServiceNow user needs `rest_api_explorer` role
- Grant read/write to `incident`, `change_request`, `sc_request`, `sc_req_item` tables
- Check IP ACLs in ServiceNow instance

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    ServiceNow Instance                  │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────┐ │
│  │   Incidents  │  │ Change Requests │  │  SC RITMs  │ │
│  └──────┬───────┘  └────────┬────────┘  └─────┬──────┘ │
└─────────┼────────────────────┼──────────────────┼────────┘
          │                    │                  │
          │ Polls every 5s     │ Polls every 10s  │
          ▼                    ▼                  │
┌─────────────────────────────────────────────────┼────────┐
│              Event-Driven Ansible                │        │
│  ┌───────────────────────────────────────────┐  │        │
│  │     Rulebook: records.yml                 │  │        │
│  │  - 3 Sources (incident, CR, CR-new)       │  │        │
│  │  - 5 Rules with conditions                │  │        │
│  └───────────────┬───────────────────────────┘  │        │
└──────────────────┼──────────────────────────────┼────────┘
                   │                              │
                   │ Dispatch Job Templates       │
                   ▼                              │
┌──────────────────────────────────────────────────────────┐
│              Ansible Automation Platform                 │
│  ┌─────────────────────┐  ┌────────────────────────┐    │
│  │   Job Templates     │  │   Workflow Templates   │    │
│  │  - Implement CR     │  │  - Patch Deployment    │    │
│  │  - Enrich RITM      │◄─┤  - Multi-stage flows   │    │
│  │  - Create CR        │  │                        │    │
│  └──────────┬──────────┘  └────────────────────────┘    │
└─────────────┼──────────────────────────────────────────┘
              │
              │ Execute Playbooks
              ▼
┌──────────────────────────────────────────────────────────┐
│                    Playbooks + Roles                     │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────┐  │
│  │ Implement      │  │ servicenow_  │  │ Remediation │  │
│  │ - Default      │  │ ticket       │  │ - Disk      │  │
│  │ - Patch        │  │ - RITM       │  │ - Agent     │  │
│  └────────────────┘  └──────────────┘  └─────────────┘  │
└─────────────┬────────────────────────────────────────────┘
              │
              │ Update/Close Records
              ▼
┌──────────────────────────────────────────────────────────┐
│                ServiceNow (Feedback Loop)                │
│          Work Notes • State Changes • Closures           │
└──────────────────────────────────────────────────────────┘
```

## Best Practices

### Idempotency
- Use `EDA-IMPLEMENT-PROCESSED` marker consistently
- Check for existing records before creating
- Use `changed_when` and `failed_when` in playbooks

### Credential Security
- Never commit credentials to version control
- Use AAP Controller credential injection
- Rotate ServiceNow API passwords regularly
- Use dedicated service accounts with minimal permissions

### Monitoring
- Enable EDA activation logs in AAP
- Set up ServiceNow alerts for marker additions
- Monitor job template execution history
- Create dashboards for automation metrics

### Customization
- Keep original playbooks intact, create copies for modifications
- Use `group_vars` or `host_vars` for environment-specific config
- Version control your customizations separately
- Document changes in playbook headers

### Testing
- Test in ServiceNow sandbox/dev instance first
- Use `--check` mode for dry runs
- Create test incidents/CRs for validation
- Implement approval workflows for production CRs

## Contributing

Contributions are welcome! Please follow these guidelines:

- **Code Style**: Follow existing patterns, use YAML best practices
- **Idempotency**: Ensure playbooks can run multiple times safely
- **Documentation**: Update README for new playbooks or roles
- **Testing**: Test changes in isolated ServiceNow environment
- **Commit Messages**: Use clear, descriptive commit messages
- **Pull Requests**: Include description of changes and testing performed

## License

SPDX-License-Identifier: MIT-0

This repository is provided as-is without warranty. You are free to use, modify, and distribute this code.
