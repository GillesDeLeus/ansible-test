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
cve-remediation.yaml                  console.redhat.com flow: ServiceNow-triggered remediation (lightspeed-remediation.yaml), then resolve the task
lightspeed-remediation.yaml           console.redhat.com flow: remediation playbook downloaded from console.redhat.com
aap/credential-type-satellite.yml     custom AAP credential type for Satellite (admin password, registry login, manifest)
--- Satellite + local Red Hat Lightspeed (see "Satellite demo" below) ---
provision-satellite.yml               Satellite 6.19 VM with local Lightspeed (IOP), manifest, RHEL 9 content, activation key
register-to-satellite.yml             register VMs to that Satellite instead of console.redhat.com
satellite-demo.yml                    provision-satellite.yml + one client VM + register-to-satellite.yml
provision-satellite-vm.yml            new RHEL 9 VM registered to the Satellite, plus its CMDB CI
patch-from-satellite.yml              apply the errata Satellite lists for a host, verify with Satellite, upload Insights
sync-satellite-cves.yml               Satellite CVEs -> ServiceNow vulnerable items (create, link, resolve); scheduled
cve-remediation-satellite.yaml        optional Satellite path: ServiceNow remediation task -> patch-from-satellite.yml -> resolve the task
aap/credential-type-activation-key.yml  custom AAP credential type for the RHSM activation key
resolve-remediation-task.yml          resolve the remediation task ServiceNow sent (imported by cve-remediation*.yaml)
resolve-vulnerable-items.yml          manual fallback: resolve vulnerable items one by one (imported by nothing)
register-cmdb-ci.yml                  create/update the VM's CI through createCI or the Table API (run at the end of provision-*vm.yml)
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
openshift/ee-build.yml                in-cluster build of the execution environment (adds helm, kubevirt.core,
                                      community.general <12, containers.podman); oc apply does not rebuild: oc start-build
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

## ServiceNow access (all ServiceNow playbooks)

1. In ServiceNow: a REST API key for an integration user that can write
   vulnerable items and remediation tasks (e.g. `sn_vul.remediation_owner`),
   and an API Access Policy with the API Key authentication profile covering
   the Table API (each playbook section lists the resources it needs).
2. Credential type from `aap/credential-type-servicenow.yml`, plus one
   credential of that type (instance URL, API key). The playbooks read it as
   `SN_HOST` / `SN_API_KEY`.

## Remediation paths

Both flows stay available; ServiceNow decides which job it launches.

| Path | ServiceNow launches | Playbooks | Who resolves items and task |
| --- | --- | --- | --- |
| console.redhat.com (Insights) | job template for `cve-remediation.yaml` with `remtask_sys_id`, `vulnerable_item_sys_id` | `lightspeed-remediation.yaml`, then `resolve-remediation-task.yml` | the job resolves the task; ServiceNow resolves its items |
| Satellite (default) | workflow "Patch VM + sync ServiceNow" with `target_hosts` = change CI | `patch-from-satellite.yml`, then `sync-satellite-cves.yml` | the sync resolves the items; Vulnerability Response closes the task |
| Satellite (optional) | job template for `cve-remediation-satellite.yaml` with `remtask_sys_id`, `vulnerable_item_sys_id` | `patch-from-satellite.yml` for those CVEs and hosts, then `resolve-remediation-task.yml` | the job resolves the task |

`resolve-vulnerable-items.yml` is not part of any path: run it by hand when
resolving a task through the API does not resolve its items.

## Resolving a remediation task (`resolve-remediation-task.yml`)

Used by the console.redhat.com path (`cve-remediation.yaml`) and the optional
Satellite path (`cve-remediation-satellite.yaml`); the default Satellite
workflow does not need it.

ServiceNow launches the remediation job template (`cve-remediation.yaml`) when
a remediation task is approved, with the task as `remtask_sys_id` and its
vulnerable items as `vulnerable_item_sys_id` (comma-separated sys_ids). After
the remediation plays, the job checks that the remediation ran on at least one
host and failed on none, then imports `resolve-remediation-task.yml`, which
sets the task to `Resolved` with a work note. ServiceNow then resolves the
task's vulnerable items itself. A failed or empty remediation leaves the task
open. Without `remtask_sys_id` (a manual run) the ServiceNow part is skipped.

The playbook reads the task first (a wrong sys_id or missing access stops the
job), skips the update when the task is already Resolved, and checks the state
ServiceNow stored. Resolving the task resolves its items in the same request,
which can take minutes; if the request still times out, the playbook keeps
reading the task until ServiceNow has finished. It then reads back the vulnerable items sent and waits up to
2 minutes for all of them to be Resolved or Closed, failing with the ones still
open; if that happens, resolving the task through the API does not resolve its
items, and `resolve-vulnerable-items.yml` does it item by item.

Remediation job template:
- Credentials: Machine, plus the "ServiceNow" credential (SN_HOST/SN_API_KEY).
- Extra variables: "Prompt on launch", so the launch API accepts
  `remtask_sys_id` and `vulnerable_item_sys_id`.
- ServiceNow access: the API Access Policy must cover the Table API resource
  `/now/table/{tableName}/{sys_id}` (GET and PATCH) for the remediation task
  table (`sn_vul_vulnerability`), and `/now/table/{tableName}` (GET) for
  `sn_vul_vulnerable_item` for the check. The key's user needs write access to
  the task (e.g. `sn_vul.remediation_owner`).

To run it on its own, use a job template with the ServiceNow credential and the
inventory with `localhost`:

```yaml
remtask_sys_id: 3a65b351c3770b10fff8b14bb001315c
vulnerable_item_sys_id: ae292f5d47bb8b105f8e370cd36d4365,66292f5d47bb8b105f8e370cd36d436a   # optional
```

| Variable | Default | Meaning |
| --- | --- | --- |
| `remtask_sys_id` | required | sys_id of the remediation task |
| `vulnerable_item_sys_id` | empty | Vulnerable items to check afterwards (comma-separated, or a list); `vulnerable_item_sys_ids` also works |
| `snow_remtask_table` | `sn_vul_vulnerability` | Table of the remediation task |
| `snow_remtask_resolved_state` | `Resolved` | State label to set on the task (label, not number) |
| `snow_remtask_timeout` | `600` | Seconds the task update may take (ServiceNow resolves the items in the same request) |
| `snow_remtask_recheck_retries` / `snow_remtask_recheck_delay` | `30` / `20` | After a timeout, how long to keep reading the task until it is Resolved |
| `snow_vi_verify` | `true` | Check the vulnerable items afterwards |
| `snow_vi_done_states` | `[Resolved, Closed]` | Item states that count as resolved |
| `snow_vi_verify_retries` / `snow_vi_verify_delay` | `12` / `10` | How long to wait for ServiceNow to resolve the items |

## Resolving vulnerable items one by one (`resolve-vulnerable-items.yml`)

Manual fallback, imported by no other playbook.

Sets the vulnerable items sent to `Resolved` with a work note, one Table API
PATCH per item, and leaves the remediation task alone. It reads the items in
batches of 100, stops before changing anything when a sys_id is not a readable
vulnerable item, skips items that are already Resolved, and fails when
ServiceNow kept the old state of an item (business rule or ACL). With
`fixed_cves`, only the items whose CVE (`vulnerability.id`) is in that list are
resolved.

ServiceNow access: the API Access Policy must cover the Table API resources
`/now/table/{tableName}` (GET) and `/now/table/{tableName}/{sys_id}` (PATCH)
for `sn_vul_vulnerable_item`, and the key's user needs write access to it.

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
| `snow_vi_resolved_state` | `Resolved` | State label to set on the items (label, not number) |

## CMDB configuration item (`register-cmdb-ci.yml`)

Runs at the end of `provision-vm.yml` and POSTs a `cmdb_ci_server` payload
for the new VM to the createCI Scripted REST API
(`/api/denbv/azure_link_demo/create_ci`), which creates the CI or updates the
one with the same `serial_number`, and builds its relationships.
OpenShift Virtualization keeps the SMBIOS serial on the VirtualMachine, so it
survives restarts; a VM recreated under the same name gets a new serial and a
new CI. Fields come from the VM's facts: name, serial, manufacturer/model
(resolved by name, left empty when ServiceNow has no such record;
`snow_cmdb_references: false` leaves them out), host name, OS and version, pod
IP (the guest only sees 10.0.2.2), RAM, CPU vendor/type/count/cores,
`virtual: true`. Empty values are not sent.

- Needs the ServiceNow credential. The API key's API Access Policies must cover
  the createCI API (POST) and the Table API GET on `cmdb_ci_server` (the
  playbook looks the CI up by serial first, to report Created or Updated).
- createCI can take a while because it builds relations: the call waits up to
  `snow_cmdb_timeout` (default 120 s). On a timeout ServiceNow may still finish,
  so check the CI before rerunning.
- `snow_cmdb_endpoint: ""` uses the Table API instead (POST, or PATCH when the
  CI exists; the key's user needs write access to `cmdb_ci_server`, and the CI
  goes into `cmdb_ci_server` itself: access to a parent table does not cover
  child classes such as `cmdb_ci_linux_server`). Relationships are then not
  built for you: set `snow_cmdb_relations`, e.g. parent "Salary System (Most
  Critical)" (`c69d3c5be8094b50dd26f07907a9fb6c`), type "Depends on::Used by"
  (`1a9cb166f1571100a92eb60da2bce5c5`), and the playbook creates the missing
  ones in `cmdb_rel_ci` (needs read/create there). Default `[]`.
- `cmdb_serial_number` matches/updates an existing CI by that serial instead of
  the VM's own.
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

## Satellite demo (Satellite + local Red Hat Lightspeed + ServiceNow)

Same story as the console.redhat.com flow, but CVE data comes from a Red Hat
Satellite 6.19 with local Red Hat Lightspeed (IOP) running as a VM on
OpenShift Virtualization. A patch is confirmed by Satellite within seconds
instead of waiting for the cloud analysis.

```
Satellite (errata/CVEs per host)
   |  sync-satellite-cves.yml        AAP schedule (15 min) and after every patch
   v
ServiceNow: vulnerable items -> remediation task (VR rule groups by CI + assignment group)
   |  approval flow -> change -> flow action launches the AAP workflow (target_hosts = change CI)
   v
AAP workflow: patch-from-satellite.yml --on success--> sync-satellite-cves.yml
   (patched CVEs disappear from Satellite -> items Resolved -> task closes)
```

### One-time setup

1. OpenShift: `oc apply -f openshift/vm-rbac.yml`. The Satellite VM needs
   about 24-32 GiB RAM; the workers (D4s_v5, 16 GiB) cannot host it, so a
   separate memory-optimised node is used (MachineSet
   `aap-snow-46qm2-virt-westeurope1`, E4s_v5, `vm_cpus: 4`, `vm_memory: 24Gi`
   to fit the Azure core quota). Scale it to 0 after deleting the VMs.
2. Red Hat: a manifest (console.redhat.com > Subscriptions > Manifests) with a
   subscription that includes RHEL, re-exported after adding it; a
   registry.redhat.io service account (terms-based registry).
3. AAP: credential types `aap/credential-type-satellite.yml` (+ the existing
   activation key, ServiceNow and OpenShift ones). The manifest goes base64
   encoded into the Satellite credential.
4. Execution environment: `oc apply -f openshift/ee-build.yml` then
   `oc start-build ee-nginx-patch -n aap --follow` (apply alone does not
   rebuild). Set the EE's pull policy to Always.

### Job templates

| Template | Playbook | Credentials | Extra vars |
| --- | --- | --- | --- |
| Satellite demo - provision | `satellite-demo.yml` | OpenShift token, Machine, Activation Key, Satellite | `satellite_fqdn`, `vm_ssh_public_key`, `vm_cpus`/`vm_memory` |
| Provision Satellite VM | `provision-satellite-vm.yml` | OpenShift token, Machine, Satellite, ServiceNow | `satellite_fqdn`, `snow_cmdb_endpoint: ""`, `snow_cmdb_relations`, `snow_vul_ci_services` (below) |
| Satellite - patch VM | `patch-from-satellite.yml` | Machine, Satellite | `satellite_fqdn`; `target_hosts` prompted |
| Satellite -> ServiceNow CVE sync | `sync-satellite-cves.yml` | Satellite, ServiceNow (no Machine; inventory with localhost only) | `satellite_fqdn`; schedule every 15 min, concurrent jobs off |
| Patch VM + sync ServiceNow (workflow) | patch node, then sync node on success | per node | prompt on launch (`target_hosts`) |
| (optional) Satellite remediation | `cve-remediation-satellite.yaml` | Machine, Satellite, ServiceNow | prompt on launch (`remtask_sys_id`, `vulnerable_item_sys_id`) |

CMDB for Satellite VMs uses the Table API, not the createCI Scripted REST API
(which attaches a fixed list of ~500 CVEs to every new CI):

```yaml
snow_cmdb_endpoint: ""
snow_cmdb_relations:
  - parent: c69d3c5be8094b50dd26f07907a9fb6c   # Salary System
    type: 1a9cb166f1571100a92eb60da2bce5c5     # Depends on::Used by
snow_vul_ci_services: [c69d3c5be8094b50dd26f07907a9fb6c]
```

### What sync-satellite-cves.yml does

Agentless (Satellite and ServiceNow APIs only), for every host registered to
Satellite except the Satellite itself:

- CVEs from the applicable security errata; CI matched by the SMBIOS serial
  Satellite reports (fallback: CI name).
- Creates missing vulnerable items with `assignment_group` (UNIX) and
  `has_impacted_services` (`snow_vi_fields`), like the createCI API did.
- The vulnerability reference is dropped by ServiceNow on insert (field write
  ACL with condition "Vulnerability is empty"), so the playbook creates one
  item, reads it back, and sets the link in a second call when needed. If
  neither works it deletes the test item and stops, instead of creating
  hundreds of unlinked items. The second call also triggers the VR rule that
  groups items into a remediation task.
- Resolves open items whose CVE Satellite no longer reports, and items that
  are linked to no vulnerability at all (`snow_close_fixed_items`,
  `snow_close_orphaned_items`).

### Limitations

- Remote execution from Satellite to the VMs does not work (masquerade
  networking: every VM reports 10.0.2.2). Patching goes through AAP.
- Local Lightspeed only sees hosts registered to this Satellite; those hosts
  are not on console.redhat.com, so `sync-insights-cves.yml` and the
  console.redhat.com remediation playbook do not apply to them.
- Update the Satellite itself only with `satellite-maintain`;
  `patch-from-satellite.yml` always skips it.

