# Virtualization Migration Factory bootcamp: Day 2 hands-on labs

Hands-on guide for **Ansible for OpenShift Virtualization Migration** (the "Migration Factory",
collection `infra.openshift_virtualization_migration`, release **1.25.0**) on the Red Hat Demo
Platform (RHDP) environment.

On Day 1 you saw *why* large migrations need a factory and *how* it works. Today you build it
and use it:

| Time | Step | Guide |
|---|---|---|
| **The day before** | **Order your RHDP environment** (provisioning can take several hours) | [Before Day 2](#before-day-2) |
| Before Day 2 | Prepare your laptop (downloads are large; do it the day before) | [00 · Prerequisites](00-prerequisites.md) |
| 15:00–15:10 | Recap, check access to your RHDP environment | this page |
| 15:10–16:00 | **Lab 1:** install and configure the factory | [01 · Lab 1](01-lab1-install-configure.md) |
| 16:00–16:10 | Break | |
| 16:10–16:40 | **Lab 2:** prepare a namespace for migration | [02 · Lab 2](02-lab2-prepare-namespace.md) |
| 16:40–17:20 | **Lab 3:** run a migration | [03 · Lab 3](03-lab3-run-migration.md) |
| any time | Something failed? | [04 · Troubleshooting](04-troubleshooting.md) |

Every lab ends with a **✅ Checkpoint**. Don't move on until it's green: Lab 3 needs Lab 2,
and Lab 2 needs Lab 1.

## Before Day 2

1. **Order your RHDP environment the day before** (link from your instructor). Provisioning often
   takes **2 hours or more**, and an environment ordered on the day may still say *provisioning*
   when the labs start. You're ready when the service shows **Running / Ready** and you received
   the details email.
2. **Prepare your laptop** with [00 · Prerequisites](00-prerequisites.md) and complete its
   ✅ pre-flight checklist, especially on **Windows** (WSL2 setup, steps W1–W8).
   **Corporate laptop or network:** have a **phone hotspot** ready. The Lab 1 seed step often fails
   on corporate networks and works from a hotspot.
3. **Get the registry credentials:** the quay.io/redhat-cop robot username and token are posted by
   the instructors in the bootcamp **Slack channel (prerequisites)**.
4. **Open your environment once:** log in to the OpenShift console and the AAP UI with the values
   from the details page. If AAP doesn't load, restart the environment from RHDP (*Stop*, then
   *Start*) before Day 2.

## Where do I run the commands?

| What | Where |
|---|---|
| `oc`, `podman`, `ansible-navigator` (prerequisites, Lab 1 step 1–3) | **Your laptop**: the Ubuntu (WSL2) terminal on Windows, a terminal on macOS/Linux. The RHDP environment has **no** terminal or bastion for you. |
| Everything after the seed (Lab 1 step 4 onwards, Lab 2, Lab 3) | **Web browsers**: the AAP UI and the OpenShift console of your RHDP environment, plus `oc` checks from your laptop |

## Your lab environment

Each participant has **their own RHDP instance**:

| Component | What it is |
|---|---|
| OpenShift cluster | One cluster that is both the factory **hub** and the migration **spoke** (a test layout; production separates them). OpenShift Virtualization and MTV are pre-installed. |
| Ansible Automation Platform | **Already installed** in your environment. In Lab 1 the factory doesn't install AAP; it **seeds** your existing AAP with credentials, job templates and workflows. |
| vCenter | A **demo vCenter** (host on your Lab instructions page, e.g. `vcenter.apps.ocpv07.rhdp.net`). You log in with the user shown there (e.g. `user1`) and see only your own four VMs. Its API is real, but it can't serve disk data (see below). |
| Your laptop | Runs `oc`, `podman` and `ansible-navigator` to seed AAP. After that, everything happens in AAP. |

> **Expected result of Lab 3:** the simulated vCenter answers every API call but can't serve
> disk data. Your migration is created, validated and started by the factory, then **fails at
> the disk conversion step** with `HTTP error response: Service Unavailable`. That's the expected
> result: it proves everything up to the disk copy works. We use it for a troubleshooting
> exercise.

## Where to find your values

Your RHDP service ("OpenShift Virtualization Migration Demo") shows the values on its **details
page / provisioning email**. The vCenter login is on the **Lab instructions** (Showroom) page.

| Value | Where (exact label) | Example | Used in |
|---|---|---|---|
| Cluster API URL | details page: *OpenShift API for command line 'oc' client* | `https://api.cluster-xxxxx.dyn.redhatworkshops.io:6443` | Lab 1 (`oc login`) |
| Console URL | details page: *OpenShift Console* | `https://console-openshift-console.apps.cluster-xxxxx…` | all labs |
| Cluster admin | details page: *User admin with password … is cluster admin* | `admin` | Lab 1 |
| AAP URL (the **AAP login page**) | details page: *Automation Controller URL* | `https://aap-aap.apps.cluster-xxxxx…` | Lab 1 inventory `aap_url` (paste as shown), AAP UI |
| AAP admin user / password | details page: *Automation Controller Admin Login / Admin Password* (often the same password as the cluster admin) | `admin` | Lab 1 inventory `aap_admin_password`, AAP UI |
| vCenter URL | **Lab instructions** page → *Navigate to the vCenter Console* link (differs per environment) | `https://vcenter.apps.ocpv07.rhdp.net` | Lab 1 inventory `vcenter_url` (paste as shown) |
| vCenter user / password | **Lab instructions** page → *Setting Up Your Environment* | `user1` | Lab 1 inventory |
| quay.io/redhat-cop robot username and token | bootcamp **Slack channel (prerequisites)**, posted by the instructors (not in this repo) | | Prerequisites (`podman login`), Lab 1 inventory |

> The RHDP item is also a stand-alone demo: its own AAP job templates (*01 Migrate VMware VMs to
> OpenShift Virtualization*, *02 Provision Linux VMs*, …) and an MTV provider called `vmware`
> already exist. **Ignore them.** In these labs you build and use the Migration Factory's own
> objects (organization *OpenShift Virtualization Migration*, provider `vmware-virt-bootcamp`).

Keep these values in a local note. Never commit them to Git.

## Asking for help

Post in the bootcamp chat or Slack, and include all of this, so we can help in one reply:
```text
Lab / step:            e.g. Lab 1, step 3
OS:                    Windows 11 + WSL Ubuntu 24.04 / macOS / Linux
ansible-navigator:     output of: ansible-navigator --version
Command you ran:       (copy it)
Last error lines:      (copy the "fatal:" / "ERROR!" lines; remove passwords)
```
Check [04 · Troubleshooting](04-troubleshooting.md) first: most errors seen in earlier runs are
listed there with a fix.

## Repository content

```
00-prerequisites.md            laptop setup
01-lab1-install-configure.md   Lab 1
02-lab2-prepare-namespace.md   Lab 2
03-lab3-run-migration.md       Lab 3
04-troubleshooting.md          symptom → cause → fix
inventory/                     Ansible inventories used in Lab 1
images/                        screenshots
```

## Credits
The base inventory (`inventory/inventory-openshift-virt-bootcamp-base.yml`) and the screenshots
come from the APAC OpenShift Virtualization bootcamp runbook
([sabre1041/apac-openshift-virt-bootcamp](https://github.com/sabre1041/apac-openshift-virt-bootcamp)).
Screenshots were taken in that bootcamp's environment, so names such as student folders or
storage classes can differ slightly from what you see. The steps and checks are the same.
