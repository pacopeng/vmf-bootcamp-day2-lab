# 04 · Troubleshooting

All of these were seen while testing this lab. Find your symptom, apply the fix, and re-run.
Every factory job is **idempotent**, so re-running is the normal way to recover.

## Laptop / prerequisites

| Symptom | Cause | Fix |
|---|---|---|
| `Error: No container engine could be found: 'auto'` | `podman` isn't on the `PATH` of this shell (typical for bash on macOS) | `eval "$(/opt/homebrew/bin/brew shellenv)"`, check `which podman`; on macOS also `podman machine start` |
| `statfs …/com.apple.launchd…: no such file or directory` | ansible-navigator mounts the macOS SSH agent socket, which the Podman VM doesn't have | prefix the command with `SSH_AUTH_SOCK=""` |
| `Error: statfs /tmp/…` or the inventory isn't found inside the container | work folder outside your home directory; the macOS Podman VM only shares `/Users` | keep the repository under `~` |
| Podman Desktop shows 403 for `quay.io/redhat-cop` | Podman Desktop uses the whole path as a host name | log in from the CLI: `podman login quay.io/redhat-cop` |
| `invalid username/password` when pulling the EE | an old stored entry for `quay.io/redhat-cop` takes precedence | `podman logout quay.io/redhat-cop`, then log in again with the robot token |
| EE pull takes very long | the image is about 550 MB | pull it before Day 2 (prerequisites, step W8 / 5) |
| `pip install` fails with `SSL: CERTIFICATE_VERIFY_FAILED` | corporate network inspects HTTPS | `pip config set global.cert <company CA file>`, or one-time: `pip install --trusted-host pypi.org --trusted-host pypi.python.org --trusted-host files.pythonhosted.org ansible-navigator` |
| RHDP service still says *provisioning* when the lab starts | RHDP provisioning can take 2 hours or more | order the environment the day before; meanwhile finish the prerequisites and read Lab 1 |
| `podman login`: where are the username and password? | the quay robot account isn't in this repo | bootcamp **Slack channel (prerequisites)** |
| `ERROR! the playbook: …/inventory/infra.openshift_virtualization_migration.migration_factory_aap could not be found` | an **old ansible-navigator** (e.g. 2.2.0) runs instead of the venv's: installed earlier by `apt` / `pip --user`, or a venv built on Python < 3.10. It treats the collection playbook name as a local file | `ansible-navigator --version` and `which -a ansible-navigator`; in the venv `pip install --upgrade ansible-navigator`, `hash -r`, remove the older copy (or call `~/.virtualenvs/openshift_virt_bootcamp/bin/ansible-navigator`); Python < 3.10 → recreate the venv (prerequisites, step 2) |
| WSL2: `cd ~/vmf-bootcamp-day2-lab/...: No such file or directory`, or odd mount errors | the repository is on the Windows drive (`/mnt/c/Users/.../My Documents/...`) | clone it again inside WSL under `~` and copy your filled-in inventory over |

## Lab 1

| Symptom | Cause | Fix |
|---|---|---|
| `Unauthorized` / `the server has asked for the client to provide credentials` | `oc` token expired or wrong cluster | `oc login …` again; check `oc whoami --show-server`; re-run step 3 |
| `No OpenShift auth method provided` | the `-e openshift_host=… -e openshift_temporary_api_key=…` options are missing, or `oc whoami -t` returned nothing | log in again and re-run the full command |
| `Failed to get token: The read operation timed out` | AAP was slow while many objects were created in parallel | `aap_request_timeout: 60` must be in your inventory; re-run |
| `TimeoutError: The read operation timed out` (`…/api/controller/v2/organizations/…`, many attempts) | AAP not ready yet / overloaded (fresh environment), or a slow network path | check `curl -sk https://<AAP host>/api/gateway/v1/ping/` and the AAP UI login; wait and re-run; if it persists restart the environment in RHDP (*Stop* / *Start*); `aap_request_timeout: 60` in the inventory; corporate network: run the seed from a phone hotspot |
| `Invalid credentials` / 401 from AAP | wrong `aap_url` / `aap_admin_password` | `aap_url` = the **Automation Controller URL** (AAP login page), password = *Automation Controller Admin Password*; check `curl -sk https://<AAP host>/api/gateway/v1/ping/` |
| `SSL: UNEXPECTED_EOF_WHILE_READING … EOF occurred in violation of protocol` while seeding AAP (a different random request fails on each run) | the **corporate network** (VPN / security client such as Zscaler, Netskope, Palo Alto) cuts HTTPS connections. Container network, proxy variables and AAP settings are **not** the cause | **Confirmed fix: switch the laptop to a phone hotspot** (or a home network with VPN and security client off) **for the seed step only**, then re-run (idempotent). After `failed=0` you can go back to the corporate network: Labs 1 (step 4 on) to 3 run in the browser. If a hotspot isn't possible: disconnect the VPN / pause the security client for the seed, or try a lower WSL MTU (`sudo ip link set dev eth0 mtu 1300`) |
| URLs like `https://https://…` or "Name or service not known" for vCenter/AAP | an old inventory where the host name lines had `https://` pasted in | use the current inventory template: paste the URLs into `vcenter_url` / `aap_url`; the host names are derived |
| A playbook task tries to install or subscribe AAP | `install_aap: false` is missing under `migration_hub` | add it (see the inventory template) and re-run |
| Workflow: **Configure MTV Provider** fails | wrong vCenter values | fix `vcenter_url` / `vmware_username` / `vmware_password` in the inventory and re-run step 3 (or edit the `vmware-virt-bootcamp` credential in AAP) and relaunch the workflow |
| Workflow fails at mapping: `Source Provider with name 'vmware-virt-bootcamp' not found in namespace 'openshift-mtv'` | the provider exists but MTV couldn't connect to vCenter (wrong URL/user/password), or MTV's inventory wasn't loaded yet | check the provider conditions (Lab 1, after the checkpoint); fix the vCenter values and re-run step 3, or just relaunch the workflow if the provider is Ready |
| Provider not Ready, message about the certificate | TLS check against the simulated vCenter | `insecureSkipTlsVerify: true` on the target, re-seed, relaunch |

## Lab 2

| Symptom | Cause | Fix |
|---|---|---|
| `namespaces "vmexamples-automation" not found` | step 1 skipped | create the namespace, relaunch |
| `Could not locate a single NetworkAttachmentDefinition for network portgroup '<name>'` | **two** NADs are labelled with the same port group (in any namespace) | keep one; remove the other or its `infra.openshift-virtualization-migration/source-portgroup` annotation; relaunch Configure Target |
| A port group maps to `pod` although you created a NAD | the `portgroup` value differs from the vSphere name (case!), or the NAD was created after the maps | fix the name, re-run Configure Networks, then Configure Target |
| Maps created in `openshift-mtv` instead of your namespace | `mtv_management_migration_namespace` not entered in the launch Variables | relaunch with the variable |
| Storage map still shows the default class after the optional override | `mtv_management_default_storage_class` missing in that launch, or a later re-run without it reset the map | relaunch Configure Target with both variables; check with the `oc get storagemap … -o jsonpath` command in Lab 2, step 4 |
| Lab 3: disk PVC stays `Pending` / DiskAllocation fails after an override | the storage class name doesn't exist (typo) or can't host VM disks (e.g. `openshift-storage.noobaa.io`) | `oc get storageclass`; relaunch Lab 2 step 4 with a valid RBD/CephFS class, then re-create the Plan |

## Lab 3

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to locate VM with name/path …` | the name doesn't exist in **your** user's inventory, or it isn't unique | copy the exact name from Providers → vmware-virt-bootcamp → Virtual Machines; use `path` if names repeat |
| `Unable to locate StorageMap/NetworkMap …` | Lab 2 not finished, or a wrong namespace in the request | check `oc get networkmap,storagemap -n vmexamples-automation` |
| `Unable to locate Source Provider …` | `source_namespace` missing (it defaults to `mtv_namespace`) | set `source_namespace: openshift-mtv` and `destination_namespace: openshift-mtv` |
| Plan stays **Validating** for minutes | the VDDK validator job is pulling images for the first time | wait; check `oc get pods -n vmexamples-automation` and act only on `ErrImagePull` |
| Plan warnings `NetMapPreservingIPsOnPodNetwork`, `VMMissingGuestIPs` | port groups on the pod network / no guest IPs from vCenter | warnings only; the Plan still runs |
| `dry_run: true` and nothing appears in the console | expected: a dry run only prints the Plan in the job output | run again with `start_migration: false` |
| **ImageConversion** fails with `HTTP error response: Service Unavailable` | the demo vCenter has no disk service | **expected** in this environment; see the Lab 3 checkpoint |
| Conversion pod log ends with only `Error executing v2v command: exit status 1` | the real error is in the middle of a ~1,000-line log | `oc logs -n vmexamples-automation <pod> \| grep -E 'OpenEx\|Service Unavailable' \| head -5` |
| You used the RHDP demo's own template *01 Migrate VMware VMs to OpenShift Virtualization* | that's the demo's template, not the factory | use **OpenShift Virtualization Migration - Migrate - openshift-virt.bootcamp.redhat.com** |

## Still stuck?
Collect the job output (AAP → Jobs → the failed job → Output) and the exact error line, then ask
your instructor.

---
Back to [README](README.md)
