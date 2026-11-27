# 02 · Lab 2: prepare a namespace for migration (30 min)

**Goal:** prepare a new application namespace, `vmexamples-automation`, as a migration
destination. Each application team usually gets its own namespace, and every namespace needs
its **own** VDDK secret, network map and storage map.

```text
1. create the namespace            (Platform team; the factory doesn't create namespaces)
2. Configure Networks  [optional]  NADs for the port groups the VMs use
3. Configure Target                VDDK secret + NetworkMap + StorageMap in the namespace
4. Override storage    [optional]  a different storage class for this namespace
```

## Step 1 · Create the namespace

Console: **Home → Projects → Create Project**, name `vmexamples-automation`. Or from the command line:
```bash
oc new-project vmexamples-automation
```
![Projects](images/l2-openshift-console-projects.png)

> **Why first?** No factory template creates namespaces. If the namespace doesn't exist,
> Configure Networks and Configure Target fail on their first object.

## Step 2 · (Optional) Configure Networks

**Skip this step in the RHDP environment.** It has no NMState operator (`oc get nncp` →
"the server doesn't have a resource type"), so there's no localnet or bridge for a NAD to use.
Your VMs map to the pod network, which is fine for this lab. Read the step anyway: it's how you'd
do it on a real cluster. Run it only if your instructor gives you a localnet name.

The job template **Configure Networks - vmware-virt-bootcamp - openshift-virt.bootcamp.redhat.com**
creates NetworkAttachmentDefinitions (NADs) and labels each one with the vSphere port group it
represents. Launch it and enter these **Variables**:
```yaml
network_mgmt_manual_localnet_name: <localnet name from the instructor>
network_mgmt_openshift_network_bridge_mode: ovs-bridge
network_mgmt_manual_nad_list:
  - name: <nad name, e.g. vm-network>
    portgroup: <exact vSphere port group name>     # case-sensitive
    namespace: default                             # shared NAD, usable from every namespace
    trunk: false
    vlan:
      vlan_id: 0                                   # 0 = untagged
```
![Configure Networks template](images/l2-aap-job-templates-nmstate.png)
![Variables](images/l2-aap-nmstate-variables.png)

The NAD appears under **Networking → NetworkAttachmentDefinitions**:

![NAD created](images/l2-openshift-console-nad.png)

> **Two rules:**
> 1. `portgroup` must match the vSphere port group name **exactly**, including upper and lower case.
>    A mismatch silently maps the port group to the pod network.
> 2. Each port group may have **only one** labelled NAD per cluster. A second one in another
>    namespace makes the mapping job fail with "Could not locate a single
>    NetworkAttachmentDefinition".

## Step 3 · Run Configure Target for the namespace

In **Templates**, launch the workflow
**Configure Target - vmware-virt-bootcamp - openshift-virt.bootcamp.redhat.com** with this **Variable**:
```yaml
mtv_management_migration_namespace: vmexamples-automation
```
![Configure Target launch](images/l2-aap-configure-target.png)

This runs the same chain as in Lab 1 (Provider → VDDK → Mapping), but creates the VDDK secret
and the maps **in `vmexamples-automation`**. The provider stays in `openshift-mtv`.

## ✅ Checkpoint

| Check | Where (project `vmexamples-automation`) | Expected |
|---|---|---|
| VDDK secret | Workloads → Secrets | `vmware-virt-bootcamp-vddk` |
| Network map | Migration for Virtualization → Network maps | `vmware-virt-bootcamp-host`, Ready |
| Storage map | Migration for Virtualization → Storage maps | `vmware-virt-bootcamp-host`, Ready |

![Secret in the namespace](images/l2-openshift-console-namespace-secrets.png)

```bash
oc get secret vmware-virt-bootcamp-vddk -n vmexamples-automation
oc get networkmap,storagemap -n vmexamples-automation
oc get networkmap vmware-virt-bootcamp-host -n vmexamples-automation \
  -o jsonpath='{range .spec.map[*]}{.source.name} -> {.destination.type} {.destination.namespace}/{.destination.name}{"\n"}{end}'
```

Expected in the RHDP environment (the workflow takes about 2 minutes):
```text
segment-migrating-to-ocpvirt -> pod /
```
and the storage map sends datastore `workloadshare` to `ocs-external-storagecluster-ceph-rbd`
(the cluster's default storage class):
```bash
oc get storagemap vmware-virt-bootcamp-host -n vmexamples-automation \
  -o jsonpath='{range .spec.map[*]}{.source.name} -> {.destination.storageClass}{"\n"}{end}'
```

**Read your network map.** Each vSphere port group points either to a NAD (`multus`) or to the
**pod network** (`pod`):
- **Without step 2, all port groups map to `pod`.** No NAD carries a matching label, so the
  factory falls back to the pod network. That's correct for this lab.
- **With step 2, your port group maps to `multus default/<nad name>`.** The screenshot below shows
  an example with a NAD (from the bootcamp environment):

![NetworkMap YAML (example with a NAD)](images/l2-openshift-console-networkmap.png)

Not green? See [04 · Troubleshooting](04-troubleshooting.md#lab-2).

## Step 4 · (Optional) Override the default storage class

By default the storage map sends every vSphere datastore to the cluster's **default** storage
class. To use a different class for **this namespace only**, launch the same workflow again,
**Configure Target - vmware-virt-bootcamp - openshift-virt.bootcamp.redhat.com**, with one
extra **Variable**:

```yaml
mtv_management_migration_namespace: vmexamples-automation
mtv_management_default_storage_class: ocs-external-storagecluster-ceph-rbd-immediate
```

First list the classes on your cluster and pick one:
```bash
oc get storageclass
```
In the RHDP environment you'll see:

| Storage class | Notes |
|---|---|
| `ocs-external-storagecluster-ceph-rbd` (default) | Ceph RBD, the volume is created when the VM first starts |
| `ocs-external-storagecluster-ceph-rbd-immediate` | Ceph RBD, the volume is created immediately (**good choice for this step**) |
| `ocs-external-storagecluster-cephfs` | CephFS (file storage) |
| `openshift-storage.noobaa.io` | object storage buckets: **not** for VM disks |

**Check** that the storage map now uses your class:
```bash
oc get storagemap vmware-virt-bootcamp-host -n vmexamples-automation \
  -o jsonpath='{range .spec.map[*]}{.source.name} -> {.destination.storageClass}{"\n"}{end}'
# workloadshare -> ocs-external-storagecluster-ceph-rbd-immediate
```
In Lab 3, the migrated VM's disk (PVC) is then created with this class.

**How the factory picks the class**, per vSphere datastore:
1. an entry in `mtv_management_storage_map_overrides` for that datastore (see below);
2. else `mtv_management_default_storage_class` (what you just set);
3. else the cluster's default storage class.

**Different classes per datastore:** map each datastore by its vSphere **id** (not its name):
```yaml
mtv_management_migration_namespace: vmexamples-automation
mtv_management_storage_map_overrides:
  - id: <datastore id>                          # e.g. datastore-1234
    storageClass: ocs-external-storagecluster-cephfs
```
Find the ids in the existing map:
```bash
oc get storagemap vmware-virt-bootcamp-host -n openshift-mtv \
  -o jsonpath='{range .spec.map[*]}{.source.name} id={.source.id}{"\n"}{end}'
```

> **Keep in mind:**
> - Launch variables apply only to that run. If you re-run Configure Target (or Configure MTV
>   Mapping) for this namespace **without** the variable, the map goes back to the cluster default.
> - The class must exist on the cluster (`oc get storageclass`). A typo doesn't fail the mapping
>   job; the migration fails later, when the disk can't be created.
> - To make a class the default for **every** namespace of a target, an administrator puts
>   `mtv_management_default_storage_class` under the target's `mapping:` block in the inventory and
>   re-seeds.

## Think about it (2 minutes)
- Why does the namespace need its own maps, while the provider lives only in `openshift-mtv`?
- A port group should be usable by *all* namespaces. Where do you put its NAD? (`default`,
  labelled once.)
- Your Gold-tier datastore must land on fast storage, everything else on the default. Which
  variable do you set? (`mtv_management_storage_map_overrides` for that datastore id.)

---
Previous: [01 · Lab 1](01-lab1-install-configure.md) · Next: [03 · Lab 3: run a migration](03-lab3-run-migration.md)
