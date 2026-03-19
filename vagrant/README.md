# Vagrant Environment Setup

This directory contains the Vagrant configuration to spin up a local cluster of Ubuntu 24.04 virtual machines (`app-vm`, `db-vm`, and `lb-vm`) using VirtualBox.

## Prerequisites

Before you begin, ensure you have the following installed on your host machine:

### 1. VirtualBox
VirtualBox is the hypervisor that will run our virtual machines.
- **Download**: [VirtualBox Downloads](https://www.virtualbox.org/wiki/Downloads)
- **Ubuntu/Debian**: `sudo apt update && sudo apt install virtualbox`
- **macOS (Homebrew)**: `brew install --cask virtualbox`

### 2. Vagrant
Vagrant automates the creation and configuration of the virtual machines.
- **Download**: Vagrant Downloads
- **Ubuntu/Debian**: Follow the official HashiCorp guide on the download page to add their apt repository, then `sudo apt install vagrant`.
- **macOS (Homebrew)**: `brew install hashicorp/tap/vagrant`

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