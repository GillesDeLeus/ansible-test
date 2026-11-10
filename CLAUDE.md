# CLAUDE.md

Demo repo: Ansible Automation Platform (AAP) on OpenShift + ServiceNow
Vulnerability Response + Red Hat CVE data, either from console.redhat.com
(Insights) or from a Red Hat Satellite 6.19 with local Red Hat Lightspeed.
README.md describes every playbook, job template and variable; read the
relevant section before changing a playbook.

**This GitHub repo is public.** Never commit secrets, manifests, tokens, API
keys, personal names or customer data.

## Two flows, both kept on purpose

- console.redhat.com (Insights): `provision-vm.yml`, `sync-insights-cves.yml`,
  `cve-remediation.yaml` (+ `lightspeed-remediation.yaml`,
  `resolve-remediation-task.yml`).
- Satellite + local Lightspeed: `provision-satellite*.yml`,
  `sync-satellite-cves.yml`, AAP workflow `patch-from-satellite.yml` ->
  `sync-satellite-cves.yml`; `cve-remediation-satellite.yaml` is an optional
  alternative. `resolve-vulnerable-items.yml` is a manual fallback.
Don't remove a playbook because nothing imports it; see "Remediation paths"
in README.md.

## How changes reach the demo

- AAP runs playbooks from GitHub `main`. A change only counts after commit,
  push and a project sync in AAP. The user usually runs the git commands
  themselves (`! git commit ... && git push`); offer the exact command.
- Jobs run in a container group on the cluster with the in-cluster EE
  (`openshift/ee-build.yml`, image `ee-nginx-patch`): ansible-core 2.16,
  redhat.satellite 3.15 (no `registration_command` module), community.general
  pinned <12. `oc apply -f openshift/ee-build.yml` does not rebuild; also run
  `oc start-build ee-nginx-patch -n aap --follow`.
- Cluster-changing `oc` commands (apply, patch, delete, scale) are blocked by
  the permission classifier in Claude Code: hand them to the user as
  `! oc ...`. Read-only `oc get/describe` and throwaway inspection pods are fine:
  `oc run ee-check-$RANDOM -n aap --rm -i --restart=Never --quiet --image=image-registry.openshift-image-registry.svc:5000/aap/ee-nginx-patch:latest --command -- <cmd>`
- AAP, ServiceNow and the Satellite UI are not reachable from Claude Code; the
  user pastes job output. Ask for the failing task name and error.

## Verifying before pushing

Nothing here can run end to end locally, so test what can be tested:

- YAML: `PY=$(head -1 $(which ansible-playbook) | sed 's/^#!//'); $PY -c "import yaml;yaml.safe_load(open('FILE'))"`
- Jinja logic: small throwaway playbooks in the scratchpad with mock API
  responses, ideally loading the expression straight from the playbook (parse
  the YAML, reuse the task's `vars`/`when`) instead of retyping it.
- Module/collection questions: inspect the EE image (command above), e.g.
  `ansible-doc <module>`; the local Ansible (2.15) differs from the EE (2.16).
- Say clearly what was tested and what was not; label guesses.

## Playbook conventions

- API calls carrying keys/passwords: `no_log: true`, `failed_when: false`, then
  an `assert` with a specific `fail_msg` (HTTP code -> likely cause).
- ServiceNow dot-walked fields (`vulnerability.id`) are single keys: use
  `item['vulnerability.id']` or a Jinja loop, never `map/selectattr(attribute='a.b')`.
- Bulk writes to ServiceNow: write one record, read back what was stored,
  stop (and clean up) if a field was dropped. ServiceNow silently ignores
  fields the user may not write.
- Idempotent and re-runnable: skip finished steps (marker files, lookups first).
- Comments explain why, not what.

## Known facts and gotchas

Satellite / OpenShift
- Satellite VM: `rhel-vms/satellite`, FQDN `satellite.apps.<cluster-domain>`,
  Route (passthrough) for the UI/API, Service ClusterIP pinned in clients'
  /etc/hosts. Inside the VM the FQDN maps to its own 10.0.2.2.
- Satellite needs >= 24 GiB: runs on a separate E4s_v5 node (Azure core quota
  is the limit). Scale MachineSet `aap-snow-46qm2-virt-westeurope1` to 0 after
  the demo (delete the VMs first).
- IOP reads the registry login only from `/etc/foreman/registry-auth.json`.
- redhat.satellite modules need a writable `XDG_CACHE_HOME`; global modules
  (`setting`) reject `organization`; RHEL 9 repo sets match on `releasever` only.
- Remote execution from Satellite does not work (all VMs report 10.0.2.2):
  patch through AAP (`patch-from-satellite.yml`).
- Never patch the Satellite with dnf; only `satellite-maintain`.

ServiceNow
- Vulnerable items: write ACL on `vulnerability` with condition "Vulnerability
  is empty" drops the link on insert; the sync sets it in a second PATCH.
- Remediation tasks (`sn_vul_vulnerability`) are created by the VR rule
  "group by asset" (CI + assignment group) via the after insert/update business
  rule; it needs CI and vulnerability filled in. Links: `sn_vul_m2m_vul_group_item`.
- The createCI Scripted REST API (`/api/denbv/azure_link_demo/create_ci`)
  attaches a hardcoded list of ~500 CVEs to every new CI: Satellite VMs use the
  Table API (`snow_cmdb_endpoint: ""`) plus `snow_cmdb_relations` and
  `snow_vul_ci_services`.
- The patch is launched by the flow action "VR - Ansible- trigger PATCH JOBS"
  (AAP workflow: patch, then sync, `target_hosts` = change CI name). AAP
  ignores extra_vars unless the template/workflow has Prompt on launch.

## Letting Claude Desktop check things in the browser

Claude Desktop (Claude in Chrome) has the user's browser sessions for
ServiceNow, AAP, the Satellite UI and the Red Hat portals; Claude Code does
not. When an answer depends on what is configured or stored there (ACLs,
business rules, flows, scripts, records, AAP template settings), don't guess:
write a prompt the user can paste into Claude Desktop. Make it:

- read-only by default: "Change nothing; only read and report.";
- specific: table/menu path, record or sys_id, the exact fields wanted, and
  "quote the relevant script excerpts / field values verbatim";
- falsifiable: ask for the evidence (counts, created-by, timestamps,
  conditions), not just a conclusion.

Treat what comes back as a report, not as fact: check it against job output
and the code (it has been wrong before, e.g. blaming display-value lookups
for links the sync sets by sys_id). Changes in ServiceNow/AAP go through the
user, never through a Claude Desktop prompt the user did not ask for.
