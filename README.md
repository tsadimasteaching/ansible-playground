# Ansible Playground

This project provides a local testing environment to learn and practice Ansible. It spins up three Ubuntu 24.04 nodes (`app-vm`, `db-vm`, `lb-vm`) using Docker Compose, simulating a standard multi-node cluster.

By using Docker instead of full Virtual Machines, the environment is incredibly fast to start up, lightweight on system resources, and bypasses hypervisor issues.

## Prerequisites

To run this project on your local machine, you will need to install the following tools:

1. **Docker**: To run the simulated VMs.
2. **Docker Compose**: To orchestrate the 3-node cluster.
3. **Ansible**: Yes, Ansible **must be installed** on your local machine (host), as it will be executing the playbooks and connecting to the Docker containers via the local Docker daemon.

*(Note: If you are on Ubuntu/Debian, you can install Ansible with `sudo apt install ansible`)*

## How to Run

### 1. Start the Cluster
Bring up the simulated virtual machines in the background using Docker Compose:

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
