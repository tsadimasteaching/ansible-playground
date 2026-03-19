# Vagrant Environment Setup

This directory contains the Vagrant configuration to spin up a local cluster of Ubuntu 24.04 virtual machines (`app-vm`, `db-vm`, and `lb-vm`) using VirtualBox.

## Prerequisites

Before you begin, ensure you have the following installed on your host machine:
1. VirtualBox
2. Vagrant

## Basic Commands

Navigate to this directory (`vagrant/`) in your terminal to run the following commands.

### 1. Start the VMs (`vagrant up`)
To create and start all the virtual machines defined in the `Vagrantfile`:
```bash
vagrant up
```

### 2. Check VM Status (`vagrant status`)
To see the current state of all VMs (e.g., running, powered off, not created):
```bash
vagrant status
```

### 3. Access a VM (`vagrant ssh`)
To connect to a specific VM via SSH, use the `vagrant ssh` command followed by the machine's configured name (`app-vm`, `db-vm`, or `lb-vm`):
```bash
vagrant ssh app-vm
```

### 4. Tear Down the VMs (`vagrant destroy`)
When you are finished and want to completely remove the VMs, free up RAM, and delete their virtual hard drives:
```bash
vagrant destroy
```
*(Tip: You can append `-f` to force destruction without being prompted for confirmation).*