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
