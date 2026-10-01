# ansible-test

Patches the image of a Helm-managed nginx release on OpenShift from Ansible
Automation Platform (AAP), using the service account of the AAP job pod.

## Layout

```
patch_nginx.yml                       playbook, runs on localhost inside the job pod
inventory.yaml                        localhost only
collections/requirements.yml          kubernetes.core (installed by AAP on project sync)
roles/nginx_patch/
  defaults/main.yml                   all variables
  tasks/preflight.yml                 input checks, read the deployed release
  tasks/upgrade.yml                   helm upgrade (atomic, waits for the rollout)
  tasks/verify.yml                    assert the running pods use the patched tag
openshift/rbac.yml                    service account + RoleBinding in the nginx namespace
openshift/container-group-pod-spec.yml  AAP container group that mounts that service account
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
2. Make sure the execution environment has `helm`
   (`podman run --rm <ee-image> helm version`). If not, build and push the one
   in `execution-environment/`.
3. In AAP, create a container group with the pod spec from
   `openshift/container-group-pod-spec.yml` (set the namespace and the image).
4. Create a project from this repository, an inventory from `inventory.yaml`,
   and a job template for `patch_nginx.yml` that uses the container group.

## Variables

Set these on the job template (extra vars or a survey):

| Variable | Required | Meaning |
| --- | --- | --- |
| `nginx_patch_namespace` | yes | Namespace of the release |
| `nginx_patch_release_name` | yes | Helm release name |
| `nginx_patch_chart_ref` | yes | Chart name, or a full `oci://` reference |
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
