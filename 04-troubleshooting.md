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
| EE pull takes very long | the image is about 550 MB | pull it before Day 2 (prerequisites, step 5) |

## Lab 1

| Symptom | Cause | Fix |
|---|---|---|
| `Unauthorized` / `the server has asked for the client to provide credentials` | `oc` token expired or wrong cluster | `oc login …` again; check `oc whoami --show-server`; re-run step 3 |
| `No OpenShift auth method provided` | the `-e openshift_host=… -e openshift_temporary_api_key=…` options are missing, or `oc whoami -t` returned nothing | log in again and re-run the full command |
| `Failed to get token: The read operation timed out` | AAP was slow while many objects were created in parallel | `aap_request_timeout: 60` must be in your inventory; re-run |
| `Invalid credentials` / 401 from AAP | wrong `controller_*` / `aap_*` values | host name **without** `https://`, admin password from your RHDP page |
| A playbook task tries to install or subscribe AAP | `install_aap: false` is missing under `migration_hub` | add it (see the inventory template) and re-run |
| Workflow: **Configure MTV Provider** fails | wrong vCenter values | fix `vmware_*` in the inventory and re-run step 3 (or edit the `vmware-virt-bootcamp` credential in AAP) and relaunch the workflow |
| Provider not Ready, message about the certificate | TLS check against the simulated vCenter | `insecureSkipTlsVerify: true` on the target, re-seed, relaunch |

## Lab 2

| Symptom | Cause | Fix |
|---|---|---|
| `namespaces "vmexamples-automation" not found` | step 1 skipped | create the namespace, relaunch |
| `Could not locate a single NetworkAttachmentDefinition for network portgroup '<name>'` | **two** NADs are labelled with the same port group (in any namespace) | keep one; remove the other or its `infra.openshift-virtualization-migration/source-portgroup` annotation; relaunch Configure Target |
| A port group maps to `pod` although you created a NAD | the `portgroup` value differs from the vSphere name (case!), or the NAD was created after the maps | fix the name, re-run Configure Networks, then Configure Target |
| Maps created in `openshift-mtv` instead of your namespace | `mtv_management_migration_namespace` not entered in the launch Variables | relaunch with the variable |

## Lab 3

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to locate VM with name/path …` | the name doesn't exist in **your** user's inventory, or it isn't unique | copy the exact name from Providers → vmware-virt-bootcamp → Virtual Machines; use `path` if names repeat |
| `Unable to locate StorageMap/NetworkMap …` | Lab 2 not finished, or a wrong namespace in the request | check `oc get networkmap,storagemap -n vmexamples-automation` |
| `Unable to locate Source Provider …` | `source_namespace` missing (it defaults to `mtv_namespace`) | set `source_namespace: openshift-mtv` and `destination_namespace: openshift-mtv` |
| Plan stays **Validating** for minutes | the VDDK validator job is pulling images for the first time | wait; check `oc get pods -n vmexamples-automation` and act only on `ErrImagePull` |
| Plan warnings `NetMapPreservingIPsOnPodNetwork`, `VMMissingGuestIPs` | port groups on the pod network / no guest IPs from vCenter | warnings only; the Plan still runs |
| `dry_run: true` and nothing appears in the console | expected: a dry run only prints the Plan in the job output | run again with `start_migration: false` |
| **ImageConversion** fails with `HTTP error response: Service Unavailable` | the simulated vCenter has no disk service | **expected** in this environment; see the Lab 3 checkpoint |

## Still stuck?
Collect the job output (AAP → Jobs → the failed job → Output) and the exact error line, then ask
your instructor.

---
Back to [README](README.md)
