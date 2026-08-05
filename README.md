# ServiceNow network incident automation

This project processes a ServiceNow network incident from start to finish. The
same playbook can run from Ansible CLI during development and later serve as an
Ansible Automation Platform (AAP) job-template entry point.

## Design

```text
CLI or AAP job template
        |
        v
playbooks/servicenow_incident_update.yml
        |
        v
Update parent -> parse incident -> lookup device -> ICMP ping
        |
        +-- reachable: close parent incident
        |
        +-- unreachable: create child and put parent on hold
```

The project intentionally uses the maintained `servicenow.itsm` collection
directly instead of wrapping one task in a role or custom Python API client.

## Local development

Ansible control nodes run on Linux. On a Windows workstation, run these
commands in WSL (Ubuntu is sufficient) from this repository directory.

1. Install Ansible in a Python virtual environment:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install --upgrade pip
   python -m pip install ansible-core
   ```

2. Install the ServiceNow collection:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

3. Export credentials. Do not put them in Git or in the example variable file:

   ```bash
   export SN_HOST='https://your-instance.service-now.com'
   export SN_USERNAME='automation-user'
   export SN_PASSWORD='replace-me'
   ```

4. In `inventory/hosts.yml`, replace the sample device with a test device and
   its management IP.

5. Copy `examples/incident_update.yml`, change the incident values, and run:

   ```bash
   ansible-playbook playbooks/servicenow_incident_update.yml \
     --extra-vars @examples/incident_update.yml
   ```

The sample management address `192.0.2.10` is a documentation-only address and
is expected to be unreachable until it is replaced.

## Inputs

The ServiceNow event or local launcher supplies these variables.

| Variable | Purpose |
| --- | --- |
| `servicenow_incident_number` | Incident to update, such as `INC0012345` |
| `servicenow_incident_sys_id` | Parent incident sys_id used to link the child |
| `servicenow_incident_short_description` | Structured device and interface text |
| `servicenow_incident_description` | Full incident description |
| `automation_job_id` | Local simulator job ID; AAP supplies `awx_job_id` |

Required short-description format:

```text
device=router01; interface=GigabitEthernet0/1; issue=down
```

The parsed device name must exist in the Ansible inventory and define
`management_ip`. The controller sends two ICMP ping requests to this address.
The local Ubuntu host and future AAP execution environment must contain the
`ping` command, normally supplied by the `iputils-ping` package.

The playbook defaults can be overridden with extra variables:

| Variable | Default |
| --- | --- |
| `servicenow_automation_assignment_group` | `Network Automation` |
| `servicenow_child_assignment_group` | `Network Support` |
| `servicenow_child_caller` | `admin` |
| `servicenow_success_state` | `closed` |
| `servicenow_success_close_code` | `Solved (Permanently)` |
| `servicenow_hold_reason` | `awaiting_problem` |

## Move to AAP

1. Add this Git repository as an AAP project.
2. Ensure the execution environment contains `servicenow.itsm` in the supported
   2.x range specified by `requirements.yml`.
3. Create an inventory with a `localhost` host.
4. Create a custom credential type that injects these environment variables:

   ```yaml
   env:
     SN_HOST: '{{ host }}'
     SN_USERNAME: '{{ username }}'
     SN_PASSWORD: '{{ password }}'
   ```

5. Create a job template that selects
   `playbooks/servicenow_incident_update.yml`, the localhost inventory, the
   execution environment, and the ServiceNow credential.
6. Add survey fields using the input variable names in the table above, or
   enable Prompt on launch for variables.

No playbook change is needed when moving from CLI to AAP; only the inventory,
execution environment, credentials, and survey move into AAP.

## Future growth

Replace the inventory lookup with ServiceNow CMDB, Nautobot, or another source
of truth when that integration is available. For production, add an idempotency
check using the incident event ID before creating a child incident.

The parent/child relationship is stored in the child's `parent_incident`
reference field. The playbook explicitly updates and verifies this field after
creating a child. In ServiceNow, add **Parent Incident** to the child form and
**Incident -> Parent Incident** to the parent form's related lists to display
both sides of the relationship. The API user also needs write access to
`incident.parent_incident`.
