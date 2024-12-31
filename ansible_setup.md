# Preparing the Control Node and Managed Nodes for Ansible

This document provides a detailed step-by-step procedure to configure the **control node** and **managed nodes** for playing Ansible playbooks. The Ansible user will be named `ansible`. Following community best practices ensures a secure and robust setup.

---

## Prerequisites

- A control node with Ansible installed (Linux recommended).
- One or more managed nodes (Linux-based) with SSH access enabled.
- Root or sudo access on both control and managed nodes.

---

## Step 1: Prepare the Control Node

1. **Install Ansible**:
   Ensure Ansible is installed on the control node.
   ```bash
   sudo apt update
   sudo apt install -y ansible
   ```
   For other distributions, refer to the [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

2. **Create the Ansible User**:
   Use the community-recommended approach with `adduser` for creating the `ansible` user:
   ```bash
   sudo adduser ansible --shell /bin/bash --gecos "Ansible User" --disabled-password
   ```
   Set a secure password for the `ansible` user:
   ```bash
   sudo passwd ansible
   ```

3. **Grant Sudo Access to Ansible**:
   Add the `ansible` user to the `sudoers` file for passwordless sudo access:
   ```bash
   echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
   sudo chmod 440 /etc/sudoers.d/ansible
   ```

4. **Generate SSH Keys for Ansible**:
   Switch to the `ansible` user and generate SSH keys:
   ```bash
   sudo -i -u ansible
   ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
   ```

5. **Install Python (if not present)**:
   Ansible requires Python to be installed on the control node.
   ```bash
   sudo apt install -y python3 python3-pip
   ```

---

## Step 2: Prepare the Managed Nodes

1. **Create the Ansible User**:
   On each managed node, use `adduser` to create the `ansible` user:
   ```bash
   sudo adduser ansible --shell /bin/bash --gecos "Ansible User" --disabled-password
   ```
   Set a secure password for the `ansible` user:
   ```bash
   sudo passwd ansible
   ```

2. **Grant Sudo Access to Ansible**:
   Allow the `ansible` user passwordless sudo access:
   ```bash
   echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
   sudo chmod 440 /etc/sudoers.d/ansible
   ```

3. **Install Python**:
   Ensure Python is installed on each managed node:
   ```bash
   sudo apt install -y python3 python3-pip
   ```

4. **Configure SSH Access**:
   Copy the SSH public key from the control node to each managed node:
   ```bash
   ssh-copy-id -i /home/ansible/.ssh/id_rsa ansible@<managed_node_ip>
   ```
   Test the connection:
   ```bash
   ssh ansible@<managed_node_ip>
   ```

---

## Step 3: Configure Ansible Inventory

1. **Edit the Inventory File**:
   Create or edit the Ansible inventory file, typically located at `/etc/ansible/hosts`.
   ```plaintext
   [managed_nodes]
   node1 ansible_host=192.168.1.101 ansible_user=ansible
   node2 ansible_host=192.168.1.102 ansible_user=ansible
   ```

2. **Test Connectivity**:
   Use the `ping` module to test connectivity between the control node and managed nodes:
   ```bash
   ansible -m ping all
   ```

   Expected output:
   ```plaintext
   node1 | SUCCESS => {
       "changed": false,
       "ping": "pong"
   }
   node2 | SUCCESS => {
       "changed": false,
       "ping": "pong"
   }
   ```

---

## Best Practices

1. **Use a Centralized Inventory**:
   Keep your inventory file organized, and use group variables (`group_vars`) for configuration.

2. **Secure SSH Keys**:
   Limit the use of the `ansible` user for automation only and secure the private key with appropriate permissions.
   ```bash
   chmod 600 ~/.ssh/id_rsa
   ```

3. **Use Ansible Vault**:
   Encrypt sensitive data like passwords and API keys using Ansible Vault.
   ```bash
   ansible-vault create secrets.yml
   ```

4. **Keep Ansible Updated**:
   Regularly update Ansible to benefit from the latest features and security patches.
   ```bash
   sudo apt update && sudo apt upgrade ansible
   ```

5. **Limit Privileges**:
   Only grant `NOPASSWD` privileges to `ansible` if required by your playbooks. Use role-based access control where applicable.

---

## Illustrative Example

### Scenario
You are setting up:
1. A control node with IP `192.168.1.100`.
2. Two managed nodes with IPs `192.168.1.101` and `192.168.1.102`.
3. An Ansible playbook to install NGINX on both managed nodes.

### Steps

#### 1. Prepare the Control Node
Run the following commands on the control node:
```bash
sudo apt update && sudo apt install -y ansible
sudo adduser ansible --shell /bin/bash --gecos "Ansible User" --disabled-password
sudo passwd ansible
sudo echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
sudo chmod 440 /etc/sudoers.d/ansible
sudo -i -u ansible
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

#### 2. Prepare the Managed Nodes
Run the following commands on each managed node (`192.168.1.101` and `192.168.1.102`):
```bash
sudo adduser ansible --shell /bin/bash --gecos "Ansible User" --disabled-password
sudo passwd ansible
sudo apt update && sudo apt install -y python3 python3-pip
sudo echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
sudo chmod 440 /etc/sudoers.d/ansible
```

Copy the SSH key from the control node to each managed node:
```bash
ssh-copy-id -i /home/ansible/.ssh/id_rsa ansible@192.168.1.101
ssh-copy-id -i /home/ansible/.ssh/id_rsa ansible@192.168.1.102
```

#### 3. Configure Inventory
Edit `/etc/ansible/hosts` on the control node:
```plaintext
[managed_nodes]
node1 ansible_host=192.168.1.101 ansible_user=ansible
node2 ansible_host=192.168.1.102 ansible_user=ansible
```

Test connectivity:
```bash
ansible -m ping all
```

#### 4. Create an Ansible Playbook
Create a playbook file `install_nginx.yml`:
```yaml
---
- name: Install NGINX on managed nodes
  hosts: managed_nodes
  become: true
  tasks:
    - name: Ensure NGINX is installed
      apt:
        name: nginx
        state: present
```

#### 5. Run the Playbook
Execute the playbook from the control node:
```bash
ansible-playbook install_nginx.yml
```

Expected output:
```plaintext
PLAY [Install NGINX on managed nodes] *****************************************
TASK [Gathering Facts] ********************************************************
ok: [node1]
ok: [node2]

TASK [Ensure NGINX is installed] **********************************************
changed: [node1]
changed: [node2]

PLAY RECAP ********************************************************************
node1                     : ok=2    changed=1    unreachable=0    failed=0    
node2                     : ok=2    changed=1    unreachable=0    failed=0    
```

---

This example demonstrates setting up a control node and managed nodes with Ansible, configuring SSH, and running a playbook to install NGINX while adhering to best practices.
