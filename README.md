# Ansible Playground

This project provides a local testing environment to learn and practice Ansible. It spins up three Ubuntu nodes (`app-vm`, `db-vm`, `lb-vm`), simulating a standard multi-node cluster.

You can choose to run this environment in two ways:
1. **Docker Compose**: Incredibly fast to start up, lightweight on system resources, and bypasses hypervisor issues.
2. **Vagrant & VirtualBox**: Uses full Virtual Machines for a more realistic emulation of isolated hosts.

## Prerequisites

To run this project on your local machine, you will need to install the following tools:

### General Requirements
- **Ansible**: Must be installed on your local machine (host), as it will be executing the playbooks.

*(Note: If you are on Ubuntu/Debian, you can install Ansible with `sudo apt install ansible`)*

### For Docker Environment
- **Docker**
- **Docker Compose**

### For Vagrant Environment
- **Vagrant**
- **VirtualBox**

---

## Option 1: Using Docker (Default)

### 1. Start the Docker Cluster
Bring up the simulated virtual machines in the background:

```bash
docker compose up -d
```

These containers are configured with systemd enabled (via the `geerlingguy` images) and are fully ready to be provisioned.

### 2. Verify Connectivity
Test that Ansible can successfully reach the containers natively through Docker:

```bash
ansible cluster -m ping
```
*(Note: You do not need to specify `-i inventory.yml` because it is automatically loaded via the local `ansible.cfg` file).*

### 3. Run the Playbook
Execute the sample playbook to provision the nodes (e.g., install the `htop` package):

```bash
ansible-playbook playbook.yml
```

### 4. Tear Down
When you are finished practicing, you can destroy the environment cleanly:

```bash
docker compose down
```

---

## Alternative: Vagrant

If you prefer using full Virtual Machines instead of Docker, a `Vagrantfile` is also included in this repository.

- Start VMs: `vagrant up`
- Destroy VMs: `vagrant destroy -f`

---

## Playbooks

| Playbook | Target group | What it does |
|---|---|---|
| `playbooks/postgres.yaml` | `dbservers` | Installs PostgreSQL, configures remote access, creates DB user and database |
| `playbooks/fastapi.yaml` | `appservers` | Deploys the FastAPI backend from the monorepo, sets up virtualenv, runs DB migrations, starts uvicorn |
| `playbooks/vue.yaml` | `appservers` | Deploys the Vue.js frontend, builds with npm, serves via nginx |
| `playbooks/check-ssh.yaml` | `appservers` | Verifies SSH access to GitHub using agent forwarding |

### Running a playbook

```bash
ansible-playbook playbooks/fastapi.yaml
```

### Enabling optional nginx for FastAPI

The FastAPI playbook skips nginx installation by default (Vue handles the reverse proxy). To enable it:

```bash
ansible-playbook playbooks/fastapi.yaml -e install_nginx=true
```

Or set it permanently in `group_vars/appservers.yaml`:

```yaml
install_nginx: true
```

---

## group_vars Reference

Variables are split across three files:

### `group_vars/all.yaml` — shared across every host

```yaml
app_server_host: "192.168.56.10"   # public IP of the app server (used by Vue browser JS)

db:
  host: "192.168.56.11"            # db-vm IP
  port: "5432"
  user: "dbuser"
  password: "dbpassword"
  name: "appdb"
```

### `group_vars/appservers.yaml` — app-vm only

```yaml
app_port: 8000                     # uvicorn listen port
install_nginx: false               # set true to install nginx in fastapi.yaml

git_repo_url: "https://github.com/tsadimasteaching/cloud-platforms-fastapi-vue.git"
git_repo_branch: "main"
git_clone_tmp_dir: "/tmp/repo"
git_sparse_checkout_path: "services/backend"   # backend subdir in the monorepo

tortoise_orm_config: "src.database.config.TORTOISE_ORM"

backend_server_url: "http://127.0.0.1:{{ app_port }}"  # used by nginx proxy_pass (same machine)

vue_sparse_checkout_path: "services/frontend"  # frontend subdir in the monorepo
vue_site_location: "/var/www/vue"
node_version: "20.12.1"
```

### `group_vars/dbservers.yaml` — db-vm only

```yaml
postgresql_version_map:
  "20.04": "12"
  "22.04": "14"
  "24.04": "16"
```

PostgreSQL version is resolved automatically from this map at runtime based on `ansible_facts['distribution_version']`.

### How group_vars are loaded

A host only gets the variables from **its own group(s)**. Cross-group references do not work.

```
app-vm  → loads group_vars/all.yaml         ✓
        → loads group_vars/appservers.yaml  ✓
        → does NOT load group_vars/dbservers.yaml  ✗

db-vm   → loads group_vars/all.yaml         ✓
        → loads group_vars/dbservers.yaml   ✓
        → does NOT load group_vars/appservers.yaml  ✗
```

Variables needed by multiple groups (e.g. `db.*` credentials used by both the app and DB server) belong in `group_vars/all.yaml`.

---

## GitHub SSH Access from Playbooks

Some playbooks (e.g., `playbooks/check-ssh.yaml`) need to authenticate to GitHub via SSH from the remote VMs. This is handled through **SSH Agent Forwarding** — the VMs do not need a GitHub key themselves; your local key is forwarded through the Ansible SSH connection.

### How it works

```
your machine                 remote VM
┌─────────────────┐          ┌──────────────────────┐
│  ~/.ssh/id_rsa  │          │  no GitHub key needed │
│  ssh-agent      │◄────────►│  ssh -T git@github.com│
└─────────────────┘          └──────────────────────┘
   ForwardAgent=yes (set in ansible.cfg)
```

`ansible.cfg` already enables this:

```ini
[ssh_connection]
ssh_args = -o ForwardAgent=yes -o ControlMaster=auto -o ControlPersist=60s
```

### Steps

**1. Add your GitHub SSH key to the local agent:**

```bash
ssh-add ~/.ssh/id_ed25519    # or id_rsa, whichever key GitHub knows
ssh-add -l                   # verify it's loaded
```

**2. Verify the key works locally:**

```bash
ssh -T git@github.com
# Expected: Hi <username>! You've successfully authenticated...
```

**3. Run the playbook:**

```bash
ansible-playbook playbooks/check-ssh.yaml
```

### Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Key not in ssh-agent | `ssh-add ~/.ssh/id_ed25519` |
| `ssh-add -l` returns nothing | ssh-agent not running | `eval $(ssh-agent) && ssh-add` |
| Works locally but fails in playbook | Agent started after Ansible session | Restart terminal, re-add key, re-run |

---

## Sparse Checkout for Monorepos

Both `fastapi.yaml` and `vue.yaml` deploy only a subdirectory from a monorepo. **Sparse checkout** tells Git to only populate the working tree with a specific path, avoiding the cost of checking out the entire repository.

```
cloud-platforms-fastapi-vue/   ← full repo
├── services/
│   ├── backend/               ← fastapi.yaml deploys this
│   └── frontend/              ← vue.yaml deploys this
└── ...
```

The playbooks perform this in four steps:

```bash
# 1. Clone the repo
git clone <repo> /tmp/repo

# 2. Enable cone-mode sparse checkout
git sparse-checkout init --cone

# 3. Declare which path to materialise
git sparse-checkout set services/backend   # or services/frontend

# 4. Populate the working tree
git checkout main
```

After checkout, only the declared subdirectory exists on disk. The playbook moves it to the final destination and deletes the temporary clone.

The paths are controlled by `git_sparse_checkout_path` and `vue_sparse_checkout_path` in `group_vars/appservers.yaml`.

### Step 2 in detail — `git sparse-checkout init --cone`

Sparse checkout has two modes. Without `--cone` (the old mode), Git matches paths using `.gitignore`-style patterns — flexible but slow, because Git must compare every file in the index against every pattern on every operation.

`--cone` is a stricter, faster mode that only allows **directories** as targets. It works with a simple rule set:

- everything at the repo root is included (README, config files, etc.)
- everything inside the declared directories is included
- everything else is excluded

Because cone mode only deals with directory boundaries (not arbitrary glob patterns), Git can use a much faster prefix lookup rather than pattern-matching every file.

### Step 3 in detail — `git sparse-checkout set services/backend`

This declares exactly which directory to materialise in the working tree. Internally Git writes three rules:

```
/*                   ← include root-level files
!/*/                 ← exclude all top-level directories...
/services/backend/   ← ...except this one
```

So on disk after `git checkout` you get:

```
/tmp/repo/
├── README.md          ← root files included (cone rule)
└── services/
    └── backend/       ← the declared directory, fully included
                         services/frontend/ does NOT appear
```

`services/` itself is not fully checked out — only the specific subdirectory declared. Git never writes `services/frontend/` to disk at all, even though it exists in the repo history.
