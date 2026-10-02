# ansible-test

Patches the image of a Helm-managed nginx release on OpenShift from Ansible
Automation Platform (AAP), using a service account token stored in an AAP
credential.

## Layout

```
patch_nginx.yml                       playbook, runs on localhost inside the job pod
inventory.yaml                        localhost and the variables of the vulnerable-nginx release
charts/vulnerable-nginx/              chart of that release, extracted from the cluster
collections/requirements.yml          kubernetes.core (installed by AAP on project sync)
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
5. Create a project from this repository, an inventory from `inventory.yaml`,
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
