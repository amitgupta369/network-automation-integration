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

The project uses the maintained `servicenow.itsm` collection for incident
operations. The `networktocode.nautobot` collection and `pynautobot` SDK are
included for the next phase, which will replace the static device inventory
lookup with Nautobot.

## Local development

Ansible control nodes run on Linux.

1. Install the system package used for ICMP checks:

   ```bash
   sudo apt-get update
   sudo apt-get install -y iputils-ping
   ```

2. Create a Python virtual environment and install Python dependencies:

   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   python -m pip install --upgrade pip
   python -m pip install -r requirements.txt
   ```

3. Install the ServiceNow and Nautobot collections:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

4. Configure credentials as described in the next section.

5. In `inventory/hosts.yml`, replace the sample device with a test device and
   its management IP.

6. Copy `examples/incident_update.yml`, change the incident values, and run:

   ```bash
   ansible-playbook playbooks/servicenow_incident_update.yml \
     --extra-vars @examples/incident_update.yml
   ```

The sample management address `192.0.2.10` is a documentation-only address and
is expected to be unreachable until it is replaced.

## Local credentials

The collections read credentials from environment variables. Do not put
passwords or API tokens in playbooks, inventory, example variables, `.env`
files, or shell profiles.

### ServiceNow

Set the instance and username, then enter the password without displaying it or
placing it in shell history:

```bash
export SN_HOST='https://your-instance.service-now.com'
export SN_USERNAME='automation-user'

read -rsp 'ServiceNow password: ' SN_PASSWORD
export SN_PASSWORD
echo
```

Use a dedicated ServiceNow automation account with permission to read and write
incidents, including `incident.parent_incident`.

### Nautobot

Set the Nautobot URL and securely enter its API token:

```bash
export NAUTOBOT_URL='https://nautobot.example.com'

read -rsp 'Nautobot API token: ' NAUTOBOT_TOKEN
export NAUTOBOT_TOKEN
echo
```

Use a read-only Nautobot token for device and interface lookups. A write-enabled
token is needed only when a playbook creates or changes Nautobot objects.

Confirm that variables are set without printing their secret values:

```bash
echo "$SN_HOST"
echo "$SN_USERNAME"
test -n "$SN_PASSWORD" && echo 'SN_PASSWORD is configured'
echo "$NAUTOBOT_URL"
test -n "$NAUTOBOT_TOKEN" && echo 'NAUTOBOT_TOKEN is configured'
```

The variables exist only in the current shell. Remove them when testing is
finished:

```bash
unset SN_HOST SN_USERNAME SN_PASSWORD NAUTOBOT_URL NAUTOBOT_TOKEN
```

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
2. Build or select an execution environment containing the Python dependencies
   from `requirements.txt`, collections from `requirements.yml`, and the
   `iputils-ping` operating-system package.
3. Create an inventory with a `localhost` host.
4. Create the ServiceNow and Nautobot credentials described below.
5. Create a job template that selects
   `playbooks/servicenow_incident_update.yml`, the localhost inventory, the
   execution environment, and both credentials.
6. Add survey fields using the input variable names in the table above, or
   enable Prompt on launch for variables.

### AAP ServiceNow credential

Create a custom credential type with this input configuration:

```yaml
fields:
  - id: host
    type: string
    label: ServiceNow URL
  - id: username
    type: string
    label: ServiceNow username
  - id: password
    type: string
    label: ServiceNow password
    secret: true
required:
  - host
  - username
  - password
```

Use this injector configuration:

```yaml
env:
  SN_HOST: '{{ host }}'
  SN_USERNAME: '{{ username }}'
  SN_PASSWORD: '{{ password }}'
```

Create a credential from this type, enter the ServiceNow values, and attach it
to the job template.

### AAP Nautobot credential

Create another custom credential type with this input configuration:

```yaml
fields:
  - id: url
    type: string
    label: Nautobot URL
  - id: token
    type: string
    label: Nautobot API token
    secret: true
required:
  - url
  - token
```

Use this injector configuration:

```yaml
env:
  NAUTOBOT_URL: '{{ url }}'
  NAUTOBOT_TOKEN: '{{ token }}'
```

Create a credential from this type, enter the Nautobot values, and attach it to
the same job template. AAP stores fields marked `secret: true` encrypted and
injects them only into the job environment.

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
