# 00 · Prerequisites: prepare your laptop

Do this **before Day 2**. The execution environment image is about 550 MB, and downloading it
during the lab wastes your lab time.

You'll install and check:

| Tool | Why |
|---|---|
| Podman | `ansible-navigator` runs the automation inside a container (the execution environment, "EE") |
| Python 3.9+ and a virtual environment | to install `ansible-navigator` without touching your system Python |
| ansible-navigator | runs the factory playbook in the EE |
| `oc` | logs in to your OpenShift cluster and gives the playbook a token |
| This repository (git **or** a ZIP download) | the inventory files used in Lab 1 |
| Registry login to `quay.io/redhat-cop` | the EE image is in a protected repository |

**Supported laptops:** macOS (Apple Silicon or Intel) and Linux. On Windows use **WSL2**
(Ubuntu or Fedora) and follow the Linux steps inside WSL; that path is best effort. You need
about **5 GB of free disk space**.

---

## 1. Podman

**macOS** (Homebrew):
```bash
brew install podman
podman machine init --cpus 4 --memory 8192
podman machine start
```
**Fedora / RHEL:**
```bash
sudo dnf install -y podman
```
**Ubuntu / Debian (also WSL2):**
```bash
sudo apt-get update && sudo apt-get install -y podman
```
Check:
```bash
podman info --format '{{.Host.Arch}} {{.Version.Version}}'
```

> **macOS:** the Podman VM only shares your **home directory** (`/Users/...`). Keep this
> repository and your work folder under your home directory, not under `/tmp`.

## 2. Python virtual environment and ansible-navigator

```bash
python3 --version                      # 3.9 or newer
mkdir -p ~/.virtualenvs
python3 -m venv ~/.virtualenvs/openshift_virt_bootcamp
source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate
pip install --upgrade pip
pip install ansible-navigator
ansible-navigator --version
```
Activate the virtual environment again (`source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate`)
in every new terminal you use for the labs.

## 3. The `oc` command line

Download the client that matches your OS:
- from the **OpenShift web console** of your lab cluster: **?** (top right) → **Command line tools**, or
- from <https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/>.

Put `oc` on your `PATH`, then check:
```bash
oc version --client
```

## 4. Get this repository

You only need the two files in `inventory/`. Either way works, as long as the folder ends up as
`~/vmf-bootcamp-day2-lab` (in your home directory):

**Option A: git**
```bash
cd ~
git clone <URL of this repository> vmf-bootcamp-day2-lab
```

**Option B: no git, download the ZIP**
On the repository page click **Code → Download ZIP**, then:
```bash
cd ~
unzip ~/Downloads/<repository name>-main.zip
mv <repository name>-main vmf-bootcamp-day2-lab
```

> **Why not clone the collection source?** The factory playbook is called by its full name
> (`infra.openshift_virtualization_migration.migration_factory_aap`) and runs **inside the EE**,
> which already contains the collection. You only need the inventory files.

## 5. Registry login and the execution environment

Your instructor gives you a **robot account** for `quay.io/redhat-cop`.

```bash
podman login quay.io/redhat-cop
# Username: <robot username from the instructor>
# Password: <robot token from the instructor>

podman pull quay.io/redhat-cop/openshift-virtualization-migration-ee:2.6
podman images | grep openshift-virtualization-migration-ee
```

> **Use the command line, not Podman Desktop**, to log in. Podman Desktop treats
> `quay.io/redhat-cop` as a host name and fails with 403 even with correct credentials.

The tag `2.6` is the **AAP** version the EE is built for, not the collection version.

## 6. Shell quirks (macOS)

- **Podman not found in some terminals:** zsh loads Homebrew from `~/.zprofile`, but bash
  doesn't. If `which podman` prints nothing, run:
  ```bash
  eval "$(/opt/homebrew/bin/brew shellenv)"
  ```
- **SSH agent socket:** `ansible-navigator` tries to mount your SSH agent socket into the container,
  and on macOS that fails. Every `ansible-navigator` command in this guide starts with
  `SSH_AUTH_SOCK=""` for this reason.

## ✅ Pre-flight checklist

Run these in a **new** terminal, with the virtual environment activated:

| Check | Command | Expected |
|---|---|---|
| Podman works | `podman info >/dev/null && echo ok` | `ok` |
| ansible-navigator | `ansible-navigator --version` | a version number |
| oc | `oc version --client` | a version number |
| EE image present | `podman images \| grep migration-ee` | one line with tag `2.6` |
| Repository (git clone or ZIP) under your home directory | `ls ~/vmf-bootcamp-day2-lab/inventory` | two `.yml` files |

Problems? See [04 · Troubleshooting](04-troubleshooting.md).

---
Next: [01 · Lab 1: install and configure the factory](01-lab1-install-configure.md)
