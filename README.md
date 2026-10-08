# ansible-test

Patches the image of a Helm-managed nginx release on OpenShift from Ansible
Automation Platform (AAP), using a service account token stored in an AAP
credential.

## Layout

```
patch_nginx.yml                       playbook, runs on localhost inside the job pod
inventory.yaml                        localhost only
group_vars/all.yml                    variables of the vulnerable-nginx release (loaded with any inventory)
host_vars/localhost.yml               Python interpreter of the execution environment, for localhost only
charts/vulnerable-nginx/              chart of that release, extracted from the cluster
provision-vm.yml                      create a RHEL VM on OpenShift Virtualization, register it with console.redhat.com
aap/credential-type-activation-key.yml  custom AAP credential type for the RHSM activation key
resolve-vulnerable-item.yml           set a ServiceNow vulnerable item to Resolved
resolve-remediation-task.yml          resolve the vulnerable items ServiceNow sent (imported by cve-remediation.yaml)
register-cmdb-ci.yml                  create/update the VM's cmdb_ci_server CI (run at the end of provision-vm.yml)
sync-insights-cves.yml                ServiceNow vulnerable item per Insights CVE of the VM (run after register-cmdb-ci.yml)
tasks/redhat-api-token.yml            console.redhat.com token for the service account
aap/credential-type-redhat-service-account.yml  custom AAP credential type for the Red Hat service account
aap/credential-type-servicenow.yml    custom AAP credential type for ServiceNow (SN_HOST/SN_API_KEY)
openshift/vm-rbac.yml                 namespace + service account for provision-vm.yml
roles/nginx_patch/
  defaults/main.yml                   all variables
  tasks/preflight.yml                 input checks, read the deployed release
  tasks/upgrade.yml                   helm upgrade (atomic, waits for the rollout)
  tasks/verify.yml                    assert the running pods use the patched tag
openshift/rbac.yml                    service account + RoleBinding in the nginx namespace
openshift/ee-build.yml                in-cluster build of the execution environment (adds helm, kubevirt.core)
execution-environment/                ansible-builder definition (helm + all collections, requirements.yml)
```

## What the playbook does

1. Fails if the release does not exist or the requested tag is the vulnerable one.
2. Reads the chart version and values of the deployed release.
3. Runs `helm upgrade` with the same chart version and the same values, with
   only `image.tag` overridden. `atomic` rolls back if the pods do not become ready.
4. Lists the pods of the release and fails unless every nginx container runs
   the patched tag.

## One-time setup

1. Replace the `aap` and `nginx` namespaces in `openshift/rbac.yml`, then
   `oc apply -f openshift/rbac.yml`.
2. Build an execution environment that has `helm`. Without an external
   registry: `oc apply -f openshift/ee-build.yml` builds it in the cluster and
   stores it in the internal registry (needs the registry to be `Managed`).
   With a registry: build and push the one in `execution-environment/`.
3. In AAP, add that image as an execution environment
   (`image-registry.openshift-image-registry.svc:5000/aap/ee-nginx-patch:latest`
   for the in-cluster build).
4. In AAP, create a credential of type "OpenShift or Kubernetes API Bearer
   Token". AAP refuses `automountServiceAccountToken` in container group pod
   specs, so the job pod cannot use its own service account.
   - Endpoint: `https://kubernetes.default.svc`
   - Token: `oc create token aap-nginx-patcher -n aap --duration=24h`
     (expires; create a new one and update the credential for later runs)
   - Verify SSL on, CA data:
     `oc get cm kube-root-ca.crt -n aap -o jsonpath='{.data.ca\.crt}'`
5. Create a project from this repository, an inventory with a `localhost` host,
   and a job template for `patch_nginx.yml` with that execution environment
   and that credential. The default container group is fine.

## Variables

Set these on the job template (extra vars or a survey):

| Variable | Required | Meaning |
| --- | --- | --- |
| `nginx_patch_namespace` | yes | Namespace of the release |
| `nginx_patch_release_name` | yes | Helm release name |
| `nginx_patch_chart_ref` | yes | Chart name, a full `oci://` reference, or a path to a chart directory |
| `nginx_patch_chart_repo_url` | for non-OCI charts | Helm repository URL |
| `nginx_patch_image_tag` | yes | Patched image tag |
| `nginx_patch_vulnerable_tag` | no | Defaults to `1.2.0` |
| `nginx_patch_chart_version` | no | Defaults to the deployed chart version |
| `nginx_patch_value_overrides` | no | Defaults to `image.tag`; change if the chart uses another key |

Example:

```yaml
nginx_patch_namespace: nginx
nginx_patch_release_name: nginx
nginx_patch_chart_ref: oci://registry.example.com/charts/nginx
nginx_patch_image_tag: "1.2.1"
```

Run the job template in check mode first: it shows the planned change without
upgrading (install the `helm-diff` plugin in the execution environment for a
full diff).

## Provisioning RHEL VMs (`provision-vm.yml`)

Creates a VM from the `rhel8`, `rhel9` or `rhel10` golden image (`vm_os`), waits for SSH on its pod network
IP, registers it with an activation key and enables rhc + Insights remediation.
The job pod connects to the VM directly, so jobs must run in a container group
on the same cluster.

One-time setup:

1. `oc apply -f openshift/vm-rbac.yml` (change `aap` / `rhel-vms` if needed).
2. Rebuild the execution environment (`oc apply -f openshift/ee-build.yml`):
   it carries every collection, so the project has no collections/requirements.yml.
3. Credential type from `aap/credential-type-activation-key.yml`, plus one
   credential of that type (org ID + activation key from console.redhat.com).
4. Credential "OpenShift or Kubernetes API Bearer Token": endpoint
   `https://kubernetes.default.svc`, token from the `aap-vm-provisioner-token`
   secret, CA from `kube-root-ca.crt`.
5. Machine credential: user `cloud-user`, the private key, privilege escalation `sudo`.
6. Job template for `provision-vm.yml` with the three credentials plus the
   ServiceNow one (for the CMDB CI, see below), the
   inventory with `localhost`, and a survey for `vm_name`, `vm_ssh_public_key` and
   `vm_os` (multiple choice `rhel8` / `rhel9` / `rhel10`).

| Variable | Default | Meaning |
| --- | --- | --- |
| `vm_name` | required | VM name, lowercase DNS-1123 |
| `vm_ssh_public_key` | required | Public key matching the Machine credential |
| `vm_namespace` | `rhel-vms` | Namespace of the VM |
| `vm_os` | `rhel9` | RHEL version (`rhel8`, `rhel9`, `rhel10`); sets the DataSource and the preference |
| `vm_instancetype` | `u1.medium` | Cluster instance type |
| `vm_os_datasource` / `vm_preference` | from `vm_os` | Override only via extra vars, they must match |
| `vm_disk_size` / `vm_storage_class` | `30Gi` / cluster default | Root disk |
| `aap_inventory_name`, `aap_inventory_source_ids` | unset | Inventory sources to sync afterwards (needs an AAP credential) |

## Resolving a ServiceNow vulnerable item (`resolve-vulnerable-item.yml`)

Sets the `sn_vul_vulnerable_item` record with sys_id `vulnerability_sys_id` to
`Resolved` through the Table API, adds a work note, then reads the record back
and fails if ServiceNow kept the old state (business rule or ACL). An item that
is already Resolved is left alone.

1. In ServiceNow: a REST API key for an integration user that can write
   vulnerable items (e.g. `sn_vul.remediation_owner`), and an API Access
   Policy with the API Key authentication profile covering the Table API.
2. Credential type from `aap/credential-type-servicenow.yml`, plus one
   credential of that type (instance URL, API key).
3. Job template for `resolve-vulnerable-item.yml` with that credential and the
   inventory with `localhost`, and a required survey question
   `vulnerability_sys_id` (text, 32 characters). Callers that launch through
   the API pass it as `extra_vars`; the survey is what lets them.

| Variable | Default | Meaning |
| --- | --- | --- |
| `vulnerability_sys_id` | required | sys_id of the vulnerable item (32 hex characters) |
| `snow_vi_resolved_state` | `Resolved` | State label to set (label, not number) |
| `snow_vi_work_note` | AAP job reference | Work note added with the change |

## Resolving the vulnerable items of a remediation task (`resolve-remediation-task.yml`)

ServiceNow launches the remediation job template (`cve-remediation.yaml`) when
a remediation task is approved, with the task's vulnerable items as
`vulnerable_item_sys_id` (comma-separated sys_ids). After the remediation plays,
the job checks that the remediation ran on at least one host and failed on none,
then imports `resolve-remediation-task.yml`, which sets those vulnerable items
to `Resolved` with a work note. A failed or empty remediation leaves them open.
Without `vulnerable_item_sys_id` (a manual run) the ServiceNow part is skipped.

The playbook reads the items in batches of 100, stops before changing anything
when a sys_id is not a readable vulnerable item, skips items that are already
Resolved, and fails when ServiceNow kept the old state of an item (business
rule or ACL). With `fixed_cves`, only the items whose CVE (`vulnerability.id`)
is in that list are resolved.

Remediation job template:
- Credentials: Machine, plus the "ServiceNow" credential (SN_HOST/SN_API_KEY).
- Extra variables: "Prompt on launch", so the launch API accepts
  `vulnerable_item_sys_id`.
- ServiceNow access: the API Access Policy must cover the Table API resources
  `/now/table/{tableName}` (GET) and `/now/table/{tableName}/{sys_id}` (PATCH)
  for `sn_vul_vulnerable_item`, and the key's user needs write access to it
  (e.g. `sn_vul.remediation_owner`).

To run it on its own, use a job template with the ServiceNow credential and the
inventory with `localhost`:

```yaml
vulnerable_item_sys_id: ae292f5d47bb8b105f8e370cd36d4365,66292f5d47bb8b105f8e370cd36d436a
cmdb_ci_sys_id: b529e75d47bb8b105f8e370cd36d4331   # optional
fixed_cves: [CVE-2025-71116, CVE-2025-71147]        # optional
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `vulnerable_item_sys_id` | required | sys_ids of the vulnerable items (comma-separated, or a list); `vulnerable_item_sys_ids` also works |
| `fixed_cves` | empty | Only resolve items with these CVEs (list, or comma-separated) |
| `cmdb_ci_sys_id` | empty | Only resolve items of this CI |
| `snow_vi_resolved_state` | `Resolved` | State label to set (label, not number) |

## CMDB configuration item (`register-cmdb-ci.yml`)

Runs at the end of `provision-vm.yml` and creates a `cmdb_ci_server` CI
for the new VM through the Table API, or updates it when one with the same
`serial_number` exists. OpenShift Virtualization keeps the SMBIOS serial on
the VirtualMachine, so it survives restarts; a VM recreated under the same name
gets a new serial and a new CI. Fields come from the VM's facts: name, serial,
manufacturer/model (resolved by name, left empty when ServiceNow has no such
record; `snow_cmdb_references: false` leaves them out), host name, OS and version, pod IP (the guest only sees 10.0.2.2), RAM,
CPU vendor/type/count/cores, `virtual: true`. Empty values are not sent.

- Needs the ServiceNow credential (API key; the key's user needs write access
  to `cmdb_ci_server`). The CI goes into `cmdb_ci_server` itself: access to a
  parent table does not cover child classes such as `cmdb_ci_linux_server`.
- Afterwards it links the CI to each entry of `snow_cmdb_relations` in
  `cmdb_rel_ci`, with the CI as child. Default: parent "Salary System (Most
  Critical)" (`c69d3c5be8094b50dd26f07907a9fb6c`), type "Depends on::Used by"
  (`1a9cb166f1571100a92eb60da2bce5c5`); the business application and offering
  behind that service instance then show up as L2 relations. Existing
  relationships are left alone; needs read/create on `cmdb_rel_ci`;
  `snow_cmdb_relations: []` skips it.
- `cmdb_serial_number` matches/updates an existing CI by that serial instead of
  the VM's own; `snow_cmdb_endpoint` posts the payload to a Scripted REST API
  instead of the Table API (`snow_cmdb_timeout`, default 120 s).
- `snow_cmdb_register: false` skips it.
- Standalone, for VMs already in an AAP inventory: a job template for
  `register-cmdb-ci.yml` with the Machine and ServiceNow credentials and
  `cmdb_target: <host or group>`.

## Insights CVEs as ServiceNow vulnerable items (`sync-insights-cves.yml`)

Runs at the end of `provision-vm.yml`, after the CMDB CI exists. Per VM:

1. Reads the Insights ID (`/etc/insights-client/machine-id`) and serial from the VM.
2. Waits until the host is in Insights inventory (up to 5 min) and until the
   vulnerability service has evaluated it (up to 10 min; it answers 404 before).
3. Lists the host's CVEs (paged), optionally filtered with `insights_cve_filter`.
4. Finds the CMDB CI by serial, and the `sn_vul_nvd_entry` record of each CVE.
5. Creates an `sn_vul_vulnerable_item` (CI + vulnerability, work note with
   impact, CVSS and fix availability) for each CVE that has none for this CI.

CVEs without an `sn_vul_nvd_entry` record (not imported from NVD yet) get a
minimal record (CVE id + description from Insights), so every CVE gets a
vulnerable item; `snow_create_missing_cves: false` skips and lists them instead. Running it again only adds what is new.

Setup:

1. console.redhat.com -> Settings -> Service Accounts: create one, then in User
   Access add it to a group with the "Vulnerability viewer" and "Inventory
   Hosts viewer" roles.
2. Credential type from `aap/credential-type-redhat-service-account.yml` and a
   credential with the client ID and secret.
3. Add it and the ServiceNow credential to the provisioning job template. The
   API key needs read/create on `sn_vul_vulnerable_item` and
   `sn_vul_nvd_entry`, and read on `cmdb_ci_server`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `insights_cves_sync` | `true` | `false` skips the playbook |
| `insights_cve_filter` | empty (all CVEs) | Extra vulnerability API filter, e.g. `advisory_available=true` or `impact=5,7` |
| `snow_create_missing_cves` | `true` | Create minimal `sn_vul_nvd_entry` records for CVEs ServiceNow does not know (needs create access) |
| `insights_inventory_wait_retries` / `insights_evaluation_wait_retries` | `15` / `30` | Waits, 20 s per retry |
| `cve_sync_target` | `new_vms` | Host or group when run on its own |
