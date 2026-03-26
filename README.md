# Ansible Playground

This project provides a local testing environment to learn and practice Ansible. It spins up three Ubuntu 24.04 nodes (`app-vm`, `db-vm`, `lb-vm`), simulating a standard multi-node cluster.

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

## Alternative: Vagrant

If you prefer using full Virtual Machines instead of Docker, a `Vagrantfile.rb` is also included in this repository.

- Start VMs: `vagrant up`
- Destroy VMs: `vagrant destroy -f`

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

## How group_vars Work

The `group_vars/` directory contains one YAML file per inventory group. Ansible automatically loads and **merges** all group_vars files that apply to a host before running any play.

```
group_vars/
├── all.yaml          # loaded for every host
├── appservers.yaml   # loaded for app-vm only
├── dbservers.yaml    # loaded for db-vm only
└── lbservers.yaml    # loaded for lb-vm only
```

A host only gets the variables from **its own group(s)**. Cross-group references do not work — if `app-vm` is only in `appservers`, it will never see variables defined in `group_vars/dbservers.yaml`.

```
app-vm  → loads group_vars/all.yaml         ✓
        → loads group_vars/appservers.yaml  ✓
        → does NOT load group_vars/dbservers.yaml  ✗
```

Variables that need to be visible across multiple groups (e.g. database credentials used by both the app server and the DB server) belong in `group_vars/all.yaml`.

---

## Sparse Checkout for Monorepos

The `playbooks/fastapi.yaml` playbook deploys only the `services/backend` subdirectory from a monorepo that also contains a frontend and other components. **Sparse checkout** tells Git to only populate the working tree with a specific path, avoiding the cost of checking out the entire repository.

```
cloud-platforms-fastapi-vue/   ← full repo
├── services/
│   ├── backend/               ← only this is needed on the app server
│   └── frontend/              ← skipped
└── ...
```

The playbook performs this in four steps:

```bash
# 1. Clone the repo (index + objects, no working tree files yet for sparse paths)
git clone <repo> /tmp/repo

# 2. Enable cone-mode sparse checkout
git sparse-checkout init --cone

# 3. Declare which path to materialise
git sparse-checkout set services/backend

# 4. Populate the working tree
git checkout main
```

After checkout, only `services/backend/` exists on disk. The playbook then moves it to the final destination and deletes the temporary clone.

The path to sparse-checkout is controlled by `git_sparse_checkout_path` in `group_vars/appservers.yaml`, so it can be changed without touching the playbook.

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
