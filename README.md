# pw-Ansible-Assign

# Ansible Cloud Automation Project

This project demonstrates how to set up Ansible on a local Windows 11 machine using WSL (Windows Subsystem for Linux) and use it to automate the configuration of AWS EC2 instances in a cloud environment.

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Ansible Control Node (WSL Ubuntu)](#setup-ansible-control-node-wsl-ubuntu)
- [Prepare AWS EC2 Target Machine](#prepare-aws-ec2-target-machine)
- [Project Structure](#project-structure)
- [Ansible Playbooks](#ansible-playbooks)
  - [1. `install_nginx.yml`](#1-install_nginxyml)
  - [2. `manage_users.yml`](#2-manage_usersyml)
- [Execution Steps](#execution-steps)
- [Verification](#verification)

---

## Overview

This project showcases fundamental infrastructure as code (IaC) principles by automating the deployment of a web server (Nginx) and user management on a remote Amazon Linux EC2 instance. It serves as a practical example for anyone looking to get started with Ansible for cloud automation.

---

## Prerequisites

Before you begin, ensure you have the following:

1.  **Windows 11 Machine:** With WSL enabled and an Ubuntu distribution installed.
    * If not, follow Microsoft's official guide to install WSL and Ubuntu.
2.  **AWS Account:** With permissions to launch and manage EC2 instances.
3.  **Two AWS EC2 Linux Instances:** Launched and running in your AWS account. Make sure to note down their:
    * **Public IPv4 Address**
    * Associated **Key Pair (.pem file)**

---

## Setup Ansible Control Node (WSL Ubuntu)

Your WSL Ubuntu instance will serve as the machine from which you run Ansible commands.

1.  **Open your Ubuntu WSL terminal** from the Windows Start Menu.
2.  **Update package lists:**
    ```bash
    sudo apt update
    ```
3.  **Install Ansible:**
    ```bash
    sudo apt install ansible -y
    ```
4.  **Verify Ansible installation:**
    ```bash
    ansible --version
    ```
    You should see Ansible version information.

---

## Prepare AWS EC2 Target Machine

We'll prepare one of your EC2 instances to be managed by Ansible.

1.  **Identify your Target EC2 Instance:**
    * From your AWS EC2 Dashboard, note the **Public IPv4 address** of one of your Linux instances (e.g., `54.123.45.67`).
    * Identify the **private key file (.pem)** associated with this instance (e.g., `my-aws-key.pem`).

2.  **Transfer your AWS Private Key (`.pem` file) to WSL:**
    * In your WSL Ubuntu terminal, create a secure directory for SSH keys:
        ```bash
        mkdir -p ~/.ssh
        chmod 700 ~/.ssh
        ```
    * Copy your `.pem` file from its Windows location (e.g., `C:\Users\YourUser\Downloads\my-aws-key.pem`) to your WSL `~/.ssh/` directory. Remember to replace `YourUser` and `my-aws-key.pem` with your actual details:
        ```bash
        cp /mnt/c/Users/YourUser/Downloads/my-aws-key.pem ~/.ssh/
        ```
    * Set correct, restrictive permissions for the private key in WSL:
        ```bash
        chmod 400 ~/.ssh/my-aws-key.pem
        ```

3.  **Test SSH Connectivity from WSL to EC2:**
    * From your WSL Ubuntu terminal:
        ```bash
        ssh -i ~/.ssh/my-aws-key.pem ec2-user@<YOUR_EC2_PUBLIC_IP>
        ```
        (Replace `<YOUR_EC2_PUBLIC_IP>` and `my-aws-key.pem` with your actual values. Use `ubuntu` as `ansible_user` if you used an Ubuntu AMI).
    * If successful, type `exit` to return to your WSL terminal.

---

## Project Structure

All Ansible files will be stored in a dedicated project directory within your WSL Ubuntu instance.

1.  **Navigate to your WSL home directory:**
    ```bash
    cd ~
    ```
2.  **Create the project directory:**
    ```bash
    mkdir my_aws_ansible_project
    ```
3.  **Enter the project directory:**
    ```bash
    cd my_aws_ansible_project
    ```
4.  **Create your Ansible Inventory file (`inventory.ini`):**
    ```bash
    nano inventory.ini
    ```
    Add the following content, replacing placeholders with your actual EC2 details:
    ```ini
    [webservers]
    <YOUR_EC2_PUBLIC_IP> ansible_user=ec2-user ansible_ssh_private_key_file=~/.ssh/my-aws-key.pem
    ```
    Save and exit `nano` (`Ctrl+O`, Enter, `Ctrl+X`).

---

## Ansible Playbooks

We've created two Ansible playbooks to demonstrate different configuration tasks.

### 1. `install_nginx.yml`

This playbook installs and configures the Nginx web server on your Amazon Linux EC2 instance.

1.  **Create the playbook file:**
    ```bash
    nano install_nginx.yml
    ```
2.  **Add the following content:**
    ```yaml
    ---
    - name: Configure Webserver with Nginx on Amazon Linux
      hosts: webservers
      become: true
      tasks:
        - name: Install Nginx package (for RedHat/Amazon Linux)
          ansible.builtin.dnf:
            name: nginx
            state: present
            update_cache: yes

        - name: Ensure Nginx service is running and enabled at boot
          ansible.builtin.service:
            name: nginx
            state: started
            enabled: yes

        - name: Create a simple index.html file
          ansible.builtin.copy:
            content: "<h1>Hello from Ansible! Nginx is working on Amazon Linux!</h1>"
            dest: /usr/share/nginx/html/index.html
            mode: '0644'
    ```
3.  Save and exit `nano` (`Ctrl+O`, Enter, `Ctrl+X`).

### 2. `manage_users.yml`

This playbook demonstrates user and group management on the EC2 instance, creating a new user `devuser` and a `developers` group.

1.  **Create the playbook file:**
    ```bash
    nano manage_users.yml
    ```
2.  **Add the following content:**
    ```yaml
    ---
    - name: Manage Users and Groups on EC2 Instance
      hosts: webservers
      become: true
      tasks:
        - name: Ensure 'developers' group exists
          ansible.builtin.group:
            name: developers
            state: present

        - name: Create 'devuser' with /bin/bash shell and add to 'developers' group
          ansible.builtin.user:
            name: devuser
            comment: "Development User"
            shell: /bin/bash
            groups: developers
            append: yes
            state: present

        - name: Ensure 'devuser' has sudo privileges without password (for testing)
          ansible.builtin.lineinfile:
            path: /etc/sudoers.d/devuser_sudo
            create: true
            mode: '0440'
            line: 'devuser ALL=(ALL) NOPASSWD: ALL'
            validate: 'visudo -cf %s'
    ```
3.  Save and exit `nano` (`Ctrl+O`, Enter, `Ctrl+X`).

---

## Execution Steps

Navigate to your `my_aws_ansible_project` directory in your WSL Ubuntu terminal for all execution commands.

1.  **Test Connectivity (Ping):**
    ```bash
    ansible webservers -i inventory.ini -m ping
    ```
    Expected Output: `SUCCESS => {"ping": "pong"}`

2.  **Execute `install_nginx.yml` Playbook:**
    ```bash
    ansible-playbook -i inventory.ini install_nginx.yml
    ```
    Expected Output: `changed=X` (where X is > 0, typically 3 or more tasks).

3.  **Execute `manage_users.yml` Playbook:**
    ```bash
    ansible-playbook -i inventory.ini manage_users.yml
    ```
    Expected Output: `changed=X` (where X is > 0, typically 3 or more tasks).

---

## Verification

After executing the playbooks, verify the changes on your EC2 instance.

### For Nginx Installation:

1.  Open a web browser on your Windows machine.
2.  Navigate to `http://<YOUR_EC2_PUBLIC_IP>`.
3.  You should see the message: **"Hello from Ansible! Nginx is working on Amazon Linux!"**

### For User and Group Management:

1.  SSH into your EC2 instance from your WSL terminal:
    ```bash
    ssh -i ~/.ssh/my-aws-key.pem ec2-user@<YOUR_EC2_PUBLIC_IP>
    ```
2.  Verify the `developers` group:
    ```bash
    grep developers /etc/group
    ```
3.  Verify `devuser` exists and its group membership:
    ```bash
    id devuser
    ```
4.  Verify `devuser`'s sudo privileges:
    ```bash
    sudo -l -U devuser
    ```
5.  Type `exit` to return to your WSL terminal.

---

This `README.md` file should be a great asset for your assignment! Let me know if you'd like any adjustments or additions.
