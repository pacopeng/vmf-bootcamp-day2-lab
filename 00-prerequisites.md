# 00 · Prerequisites: prepare your laptop

Do this **before Day 2**. The execution environment image is about 550 MB, and downloading it
during the lab wastes your lab time. You need about **5 GB of free disk space**.

## Version requirements

**Older versions than the minimum below cause real failures.** For example, ansible-navigator
2.2.0 fails Lab 1 with *"the playbook: … could not be found"*. Check every row before Day 2.

**On your laptop:**

| Component | Why you need it | Minimum | Tested with | Check with | Known not to work |
|---|---|---|---|---|---|
| **ansible-navigator** | runs the factory playbook inside the EE | **25.x** | 26.11.0 | `ansible-navigator --version` and `which -a ansible-navigator` (first path inside the venv) | **2.2.0** and other old releases: treat the collection playbook name as a local file |
| **Python** (for the venv) | ansible-navigator 25.x and newer require it | **3.10** | 3.12 (Ubuntu 24.04), 3.13 (macOS) | `python3 --version` | 3.8 / 3.9: `pip` silently installs an old ansible-navigator |
| **Podman** | runs the execution environment container | **4.9** | 4.9 (Ubuntu 24.04 package), 5.7 (macOS) | `podman --version` | Podman Desktop for Windows images (not visible inside WSL) |
| **oc** | logs in to the cluster, gives the playbook a token | **4.18** (matches the cluster) | 4.18.23 | `oc version --client` | older clients mostly work but aren't supported here |
| **git** (optional) | clones this repository; a ZIP download also works | any 2.x | 2.x | `git --version` | |
| **curl**, **unzip** | downloads and checks | any | | `curl --version` | |
| **EE image** | the automation itself (includes the collection) | `quay.io/redhat-cop/openshift-virtualization-migration-ee:2.6` | `:2.6` | `podman images \| grep migration-ee` | other tags (`2.5`, `2.7`) are built for other AAP versions |

**Windows laptops only:**

| Component | Minimum | Recommended / tested | Check with | Notes |
|---|---|---|---|---|
| **Windows** | Windows 10 | Windows 11 | `winver` | |
| **WSL** | WSL **2** | WSL 2.0 or newer (`wsl --update`) | `wsl --version`, `wsl -l -v` (VERSION 2) | WSL 1 is not supported |
| **Linux distribution in WSL** | **Ubuntu 22.04** (Python 3.10) | **Ubuntu 24.04** (Python 3.12) | `cat /etc/os-release` | Ubuntu 20.04 (Python 3.8) leads to an old ansible-navigator |

**Provided by your RHDP environment** (nothing to install; for reference):

| Component | Version in the RHDP lab |
|---|---|
| OpenShift | 4.18 (4.18.23) |
| Migration Toolkit for Virtualization | 2.10 |
| Ansible Automation Platform | 2.5 (controller 4.6) |
| Migration Factory collection (AAP Project) | tag `1.25.0` (set by `aap_project_branch` in the inventory) |

**Pick your path:**
- **[Windows laptop → steps W1–W8](#windows-laptop-wsl2)** (most participants)
- **[macOS or Linux laptop → steps 1–6](#macos-or-linux-laptop)**

Both paths end with the same [✅ pre-flight checklist](#-pre-flight-checklist).

---

## Windows laptop (WSL2)

On Windows everything runs inside **WSL2** (Windows Subsystem for Linux) with **Ubuntu 24.04**.

> **Corporate laptop or network? Have a phone hotspot ready on Day 2.** Corporate networks (VPN /
> security clients) often cut the HTTPS connections of the Lab 1 seed step
> (`SSL: UNEXPECTED_EOF_WHILE_READING`). Switching the laptop to a phone hotspot for that one step
> fixed it for every affected participant.
Follow every step in order. Most problems we saw on Day 2 came from working under `/mnt/c` (step W3) or an old ansible-navigator (step W5).

### W1 · Check your Windows version
Press **Win + R**, type `winver`, press Enter.
- **Windows 11** or **Windows 10** with WSL2: both work.

### W2 · Install WSL2 and Ubuntu 24.04
Open **PowerShell as Administrator** (Start → type *PowerShell* → *Run as administrator*):
```powershell
wsl --install -d Ubuntu-24.04
```
Restart Windows if asked. Ubuntu then opens and asks you to create a **Linux user name and
password** (they don't need to match your Windows login; remember the password, `sudo` asks for it).

Already have WSL? Update it and check the version:
```powershell
wsl --update
wsl -l -v          # Ubuntu-24.04 must show VERSION 2
```
If you have an older Ubuntu (20.04 or 22.04), install 24.04 next to it and make it the default:
```powershell
wsl --install -d Ubuntu-24.04
wsl --set-default Ubuntu-24.04
```

### W3 · Always work in the Ubuntu terminal, in your Linux home
Open **Ubuntu 24.04** from the Start menu or Windows Terminal. All commands from here on, in all
labs, run **in this terminal**:
```bash
cd ~
pwd                # /home/<your linux user>
```
> **Never work under `/mnt/c/...`** (your Windows drive, e.g. `/mnt/c/Users/<you>/Documents`).
> Those paths are slow, often contain spaces (`My Documents`) and break container mounts.

### W4 · Install the Linux packages
```bash
sudo apt-get update
sudo apt-get install -y podman python3-venv git curl unzip
podman info --format '{{.Host.Arch}} {{.Version.Version}}'
```
> **Don't install `ansible` or `ansible-navigator` with `apt`.** Ubuntu's packages are too old.
> If you installed them earlier, remove them: `sudo apt-get remove -y ansible-navigator`.

### W5 · Python virtual environment and ansible-navigator
```bash
python3 --version                                  # Ubuntu 24.04: 3.12
mkdir -p ~/.virtualenvs
python3 -m venv ~/.virtualenvs/openshift_virt_bootcamp
source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate
pip install --upgrade pip ansible-navigator
hash -r
ansible-navigator --version                        # 25.x or newer (tested: 26.11)
which -a ansible-navigator                         # FIRST line must be ~/.virtualenvs/openshift_virt_bootcamp/bin/ansible-navigator
```
Run `source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate` again in **every new terminal**.

> **`pip install` fails with `SSL: CERTIFICATE_VERIFY_FAILED` (corporate network / proxy)?** Your
> company inspects HTTPS traffic. Best: point pip to the company CA bundle,
> `pip config set global.cert /path/to/company-ca.pem`. Quick one-time workaround for the lab:
> ```bash
> pip install --trusted-host pypi.org --trusted-host pypi.python.org --trusted-host files.pythonhosted.org ansible-navigator
> ```

> If `ansible-navigator --version` shows an old version (e.g. 2.2.0) or `which -a` lists another
> path first, an old copy is in the way and Lab 1 will fail with *"the playbook: …
> could not be found"*. Remove the other copy and run `hash -r`, or call
> `~/.virtualenvs/openshift_virt_bootcamp/bin/ansible-navigator` explicitly.

### W6 · Install `oc`
```bash
curl -L -o /tmp/oc.tar.gz https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.18/openshift-client-linux.tar.gz
sudo tar -xzf /tmp/oc.tar.gz -C /usr/local/bin oc kubectl
oc version --client                                # 4.18.x
```

### W7 · Get this repository (into your Linux home)
With git:
```bash
cd ~
git clone https://github.com/pacopeng/vmf-bootcamp-day2-lab.git
```
Or without git: download the ZIP in Windows from <https://github.com/pacopeng/vmf-bootcamp-day2-lab>
(**Code → Download ZIP**), then copy it into Linux:
```bash
cd ~
cp /mnt/c/Users/<your Windows user>/Downloads/vmf-bootcamp-day2-lab-main.zip ~/
unzip vmf-bootcamp-day2-lab-main.zip
mv vmf-bootcamp-day2-lab-main vmf-bootcamp-day2-lab
```

### W8 · Registry login and the execution environment
The robot username and token are posted in the bootcamp **Slack channel (prerequisites)**.
```bash
podman login quay.io/redhat-cop
# Username: <robot username from the Slack channel>
# Password: <robot token from the Slack channel>
podman pull quay.io/redhat-cop/openshift-virtualization-migration-ee:2.6
podman images | grep openshift-virtualization-migration-ee
```
Log in here in Ubuntu, not in Podman Desktop for Windows. Podman Desktop images are not
visible inside WSL.

**Done.** Continue with the [✅ pre-flight checklist](#-pre-flight-checklist).

---

## macOS or Linux laptop

### 1. Podman

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
**Ubuntu / Debian:**
```bash
sudo apt-get update && sudo apt-get install -y podman
```
Check:
```bash
podman info --format '{{.Host.Arch}} {{.Version.Version}}'
```

> **macOS:** the Podman VM only shares your **home directory** (`/Users/...`). Keep this
> repository and your work folder under your home directory, not under `/tmp`.

### 2. Python virtual environment and ansible-navigator

```bash
python3 --version                      # 3.10 or newer (required by current ansible-navigator)
mkdir -p ~/.virtualenvs
python3 -m venv ~/.virtualenvs/openshift_virt_bootcamp
source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate
pip install --upgrade pip ansible-navigator
hash -r
ansible-navigator --version            # 25.x or newer (tested: 26.11)
which -a ansible-navigator             # first line inside ~/.virtualenvs/openshift_virt_bootcamp/
```
> **Old ansible-navigator in the way:** an older copy (from `apt`, `pip --user`, or a venv built on
> Python < 3.10) doesn't understand collection playbook names. Lab 1 then fails with *"the
> playbook: …/inventory/infra.openshift_virtualization_migration.migration_factory_aap could not be
> found"*. Upgrade inside the venv, run `hash -r`, and remove or bypass the older copy. If your
> Python is older than 3.10, install a newer one (e.g. `python3.11`) and recreate the venv with it.

Activate the virtual environment again (`source ~/.virtualenvs/openshift_virt_bootcamp/bin/activate`)
in every new terminal you use for the labs.

> **`pip install` fails with `SSL: CERTIFICATE_VERIFY_FAILED` (corporate network / proxy)?** Your
> company inspects HTTPS traffic. Best: point pip to the company CA bundle,
> `pip config set global.cert /path/to/company-ca.pem`. Quick one-time workaround for the lab:
> ```bash
> pip install --trusted-host pypi.org --trusted-host pypi.python.org --trusted-host files.pythonhosted.org ansible-navigator
> ```

### 3. The `oc` command line

The RHDP lab cluster runs **OpenShift 4.18**. Use a 4.18 client. Download it:
- from the **OpenShift web console** of your lab cluster: **?** (top right) → **Command line tools**,
- or from <https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.18/>
  (`openshift-client-mac-arm64.tar.gz` for Apple Silicon, `openshift-client-mac.tar.gz` for Intel
  Macs, `openshift-client-linux.tar.gz` for Linux).

Put `oc` on your `PATH`, then check:
```bash
oc version --client
```

### 4. Get this repository

The folder must end up as `~/vmf-bootcamp-day2-lab` (in your home directory).

**Option A: git**
```bash
cd ~
git clone https://github.com/pacopeng/vmf-bootcamp-day2-lab.git
```

**Option B: no git, download the ZIP**

Open <https://github.com/pacopeng/vmf-bootcamp-day2-lab>, click **Code → Download ZIP**, then:
```bash
cd ~
unzip ~/Downloads/vmf-bootcamp-day2-lab-main.zip
mv vmf-bootcamp-day2-lab-main vmf-bootcamp-day2-lab
```

### 5. Registry login and the execution environment

Your instructors post a **robot account** for `quay.io/redhat-cop` in the bootcamp **Slack
channel (prerequisites)**.

```bash
podman login quay.io/redhat-cop
# Username: <robot username from the Slack channel>
# Password: <robot token from the Slack channel>

podman pull quay.io/redhat-cop/openshift-virtualization-migration-ee:2.6
podman images | grep openshift-virtualization-migration-ee
```

> **Use the command line, not Podman Desktop**, to log in. Podman Desktop treats
> `quay.io/redhat-cop` as a host name and fails with 403 even with correct credentials.

### 6. Shell quirks (macOS)

- **Podman not found in some terminals:** zsh loads Homebrew from `~/.zprofile`, but bash
  doesn't. If `which podman` prints nothing, run:
  ```bash
  eval "$(/opt/homebrew/bin/brew shellenv)"
  ```
- **SSH agent socket:** `ansible-navigator` tries to mount your SSH agent socket into the container,
  and on macOS that fails. Every `ansible-navigator` command in this guide starts with
  `SSH_AUTH_SOCK=""` for this reason (harmless on Linux and WSL).

---

## Good to know

- **Why not clone the collection source?** The factory playbook is called by its full name
  (`infra.openshift_virtualization_migration.migration_factory_aap`) and runs **inside the EE**,
  which already contains the collection. You only need the inventory files.
- **The EE tag `2.6`** is the **AAP** version the EE is built for, not the collection version.

## ✅ Pre-flight checklist

Run these in a **new** terminal (Windows: the **Ubuntu** terminal), with the virtual environment activated:

| Check | Command | Expected |
|---|---|---|
| Podman works | `podman info >/dev/null && echo ok` | `ok` |
| Python | `python3 --version` (inside the venv) | 3.10 or newer |
| ansible-navigator | `ansible-navigator --version` and `which -a ansible-navigator` | 25.x or newer (tested: 26.11); the first path is inside `~/.virtualenvs/openshift_virt_bootcamp/` |
| oc | `oc version --client` | 4.18.x |
| EE image present | `podman images \| grep migration-ee` | one line with tag `2.6` |
| Repository in your home directory (WSL: not `/mnt/c`) | `ls ~/vmf-bootcamp-day2-lab/inventory` | two `.yml` files |
| Internet path to quay.io | `curl -s -o /dev/null -w '%{http_code}\n' https://quay.io/v2/` | `401` (reachable) |

Problems? See [04 · Troubleshooting](04-troubleshooting.md).

---
Next: [01 · Lab 1: install and configure the factory](01-lab1-install-configure.md)
