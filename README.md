# ServiceNow incident update with Ansible

This project contains one small playbook that updates an existing ServiceNow
incident. The same playbook can run from Ansible CLI during development and
later serve as an Ansible Automation Platform (AAP) job-template entry point.

## Design

```text
CLI or AAP job template
        |
        v
playbooks/servicenow_incident_update.yml
        |
        v
servicenow.itsm.incident
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

4. Copy `examples/incident_update.yml`, change the values, and run:

   ```bash
   ansible-playbook playbooks/servicenow_incident_update.yml \
     --extra-vars @examples/incident_update.yml
   ```

Use `--check --diff` first if the target module and ServiceNow instance support
the desired update in check mode.

## Inputs

Provide `servicenow_incident_number` and whichever fields you want to update.

| Variable | Purpose |
| --- | --- |
| `servicenow_incident_number` | Incident to update, such as `INC0012345` |
| `servicenow_incident_state` | `new`, `in_progress`, `on_hold`, `resolved`, `closed`, or `canceled` |
| `servicenow_incident_work_notes` | Journal entry appended to work notes |
| `servicenow_incident_close_code` | Instance-specific close code |
| `servicenow_incident_close_notes` | Resolution or closing notes |
| `servicenow_incident_other` | Dictionary of custom incident fields |

Example custom field input:

```yaml
servicenow_incident_other:
  u_automation_job_id: "12345"
```

The dedicated work-notes variable overrides a `work_notes` value placed inside
`servicenow_incident_other`.

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
6. Add survey fields using the input variable names in the table above.

No playbook change is needed when moving from CLI to AAP; only the inventory,
execution environment, credentials, and survey move into AAP.

## Future growth

Keep direct collection calls in small playbooks while each integration remains
only one or two tasks. Introduce a role later only when several playbooks need
to share a larger sequence of tasks, defaults, templates, or handlers.
