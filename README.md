# ansible-test

Patches the image of a Helm-managed nginx release on OpenShift from Ansible
Automation Platform (AAP), using a service account token stored in an AAP
credential.

## Layout

```
patch_nginx.yml                       playbook, runs on localhost inside the job pod
inventory.yaml                        localhost only
group_vars/all.yml                    variables of the vulnerable-nginx release (loaded with any inventory)
charts/vulnerable-nginx/              chart of that release, extracted from the cluster
collections/requirements.yml          collections installed by AAP on project sync
provision-vm.yml                      create a RHEL VM on OpenShift Virtualization, register it with console.redhat.com
aap/credential-type-activation-key.yml  custom AAP credential type for the RHSM activation key
openshift/vm-rbac.yml                 namespace + service account for provision-vm.yml
roles/nginx_patch/
  defaults/main.yml                   all variables
  tasks/preflight.yml                 input checks, read the deployed release
  tasks/upgrade.yml                   helm upgrade (atomic, waits for the rollout)
  tasks/verify.yml                    assert the running pods use the patched tag
openshift/rbac.yml                    service account + RoleBinding in the nginx namespace
openshift/ee-build.yml                in-cluster build of the execution environment (adds helm)
execution-environment/                ansible-builder definition (kubernetes.core + helm)
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

Creates a VM from the `rhel8` golden image, waits for SSH on its pod network
IP, registers it with an activation key and enables rhc + Insights remediation.
The job pod connects to the VM directly, so jobs must run in a container group
on the same cluster.

One-time setup:

1. `oc apply -f openshift/vm-rbac.yml` (change `aap` / `rhel-vms` if needed).
2. Organization -> Galaxy credentials: add an Automation Hub token credential
   (console.redhat.com) above Ansible Galaxy, then sync the project.
3. Credential type from `aap/credential-type-activation-key.yml`, plus one
   credential of that type (org ID + activation key from console.redhat.com).
4. Credential "OpenShift or Kubernetes API Bearer Token": endpoint
   `https://kubernetes.default.svc`, token from the `aap-vm-provisioner-token`
   secret, CA from `kube-root-ca.crt`.
5. Machine credential: user `cloud-user`, the private key, privilege escalation `sudo`.
6. Job template for `provision-vm.yml` with the three credentials, the
   inventory with `localhost`, and a survey for `vm_name` and `vm_ssh_public_key`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `vm_name` | required | VM name, lowercase DNS-1123 |
| `vm_ssh_public_key` | required | Public key matching the Machine credential |
| `vm_namespace` | `rhel-vms` | Namespace of the VM |
| `vm_instancetype` / `vm_preference` | `u1.medium` / `rhel.8` | Cluster instance type and preference |
| `vm_os_datasource` | `rhel8` | DataSource in `openshift-virtualization-os-images` |
| `vm_disk_size` / `vm_storage_class` | `30Gi` / cluster default | Root disk |
| `aap_inventory_name`, `aap_inventory_source_ids` | unset | Inventory sources to sync afterwards (needs an AAP credential) |
