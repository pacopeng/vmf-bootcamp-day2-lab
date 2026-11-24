# Virtualization Migration Factory bootcamp: Day 2 hands-on labs

Hands-on guide for **Ansible for OpenShift Virtualization Migration** (the "Migration Factory",
collection `infra.openshift_virtualization_migration`, release **1.25.0**) on the Red Hat Demo
Platform (RHDP) environment.

On Day 1 you saw *why* large migrations need a factory and *how* it works. Today you build it
and use it:

| Time | Step | Guide |
|---|---|---|
| Before Day 2 | Prepare your laptop (downloads are large; do it the day before) | [00 · Prerequisites](00-prerequisites.md) |
| 15:00–15:10 | Recap, check access to your RHDP environment | this page |
| 15:10–16:00 | **Lab 1:** install and configure the factory | [01 · Lab 1](01-lab1-install-configure.md) |
| 16:00–16:10 | Break | |
| 16:10–16:40 | **Lab 2:** prepare a namespace for migration | [02 · Lab 2](02-lab2-prepare-namespace.md) |
| 16:40–17:20 | **Lab 3:** run a migration | [03 · Lab 3](03-lab3-run-migration.md) |
| any time | Something failed? | [04 · Troubleshooting](04-troubleshooting.md) |

Every lab ends with a **✅ Checkpoint**. Don't move on until it's green: Lab 3 needs Lab 2,
and Lab 2 needs Lab 1.

## Your lab environment

Each participant has **their own RHDP instance**:

| Component | What it is |
|---|---|
| OpenShift cluster | One cluster that is both the factory **hub** and the migration **spoke** (a test layout; production separates them). OpenShift Virtualization and MTV are pre-installed. |
| Ansible Automation Platform | **Already installed** in your environment. In Lab 1 the factory doesn't install AAP; it **seeds** your existing AAP with credentials, job templates and workflows. |
| vCenter | A **simulated vCenter, shared** by all participants. You log in with your own `userN` and see only your own VMs. |
| Your laptop | Runs `oc`, `podman` and `ansible-navigator` to seed AAP. After that, everything happens in AAP. |

> **Expected result of Lab 3:** the simulated vCenter answers every API call but can't serve
> disk data. Your migration is created, validated and started by the factory, then **fails at
> the disk conversion step** with `HTTP error response: Service Unavailable`. That's the expected
> result: it proves everything up to the disk copy works. We use it for a troubleshooting
> exercise.

## Where to find your values

| Value | Where | Used in |
|---|---|---|
| Cluster API URL, console URL | RHDP environment details page (your service in the RHDP catalog) | Lab 1 (`oc login`) |
| Cluster admin user and password | RHDP environment details page | Lab 1 |
| AAP URL, admin user and password | RHDP environment details page (if missing, see [Lab 1, step 1](01-lab1-install-configure.md#step-1--collect-your-values-and-log-in)) | Lab 1 inventory, AAP UI |
| vCenter host, `userN`, password | RHDP environment details page | Lab 1 inventory |
| quay.io/redhat-cop robot username and token | **Your instructor** (not in this repo) | Prerequisites, Lab 1 inventory |

Keep these values in a local note. Never commit them to Git.

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
