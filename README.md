# 🚀 Intro to Ansible - 30 Minute Workshop

## What is Ansible?

Ansible is an open-source **automation tool** used for **configuration management**, **application deployment**, and **orchestration**. It uses **simple YAML files** and doesn’t require agents on target machines.

> Think of Ansible as a smart assistant that can run the same instructions on many machines, reliably and repeatedly.

---

## Key Concepts

| Concept | Explanation |
|--------|-------------|
| **Inventory** | A file listing the machines you want to manage (by IP, hostname, or group). |
| **Module** | A unit of work in Ansible (e.g., install a package, copy a file). |
| **Playbook** | A YAML file describing tasks to run on target machines. |
| **Task** | A single action in a playbook. |
| **Role** | A reusable, structured way to organize playbooks. |
| **Facts** | System information collected automatically by Ansible. |
| **Ad-hoc commands** | One-liners to run a quick command without a playbook. |

## ⚙️ Lab Setup

You will be controlling from:
- **Debian 12 Ansible control node** or **Windows with WSL**

Target machines:
- **Windows Server 2025 VM**
- **Debian 12 VM**


### 💻 Running Ansible control node on Windows using WSL

If you’re using a Windows laptop, you can still run Ansible by using **WSL (Windows Subsystem for Linux)**. This gives you a real Linux shell right on Windows.

**Setup Steps:**
1. Open PowerShell as Administrator and run:
  ```powershell
  wsl --install
  ```
  This installs Ubuntu by default.

2. Once installed, launch Ubuntu from the Start Menu.

3. Update packages and install Ansible:
  ```bash
  # https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip
  sudo apt install -y python3-venv curl sshpass
  python3 -m venv ~/venv
  source ~/venv/bin/activate
  curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python3 get-pip.py
  python3 -m pip install ansible
  ansible-galaxy collection install community.windows
  ```

4. You can now run Ansible like any other Linux system:
  ```bash
  ansible --version
  ```

WSL is a great way to get started with Ansible without needing a full Linux VM.

### 💻 Running Ansible control node on Debian based distribution

**Setup Steps:**
1. Install Ansible:
  ```bash
  # https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip
  sudo apt install -y python3-venv curl sshpass
  python3 -m venv ~/venv
  source ~/venv/bin/activate
  curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && python3 get-pip.py
  python3 -m pip install ansible
  ansible-galaxy collection install community.windows
  ```
2. You can now run Ansible like any other Linux system:
  ```bash
  ansible --version
  ```

WSL is a great way to get started with Ansible without needing a full Linux VM.


### Setup windows host for ansible
When configuring windows WinRM needs to be activated. 
> This may pose a security risk so make sure to configure the firewall properly in a production settings

```powershell
# Start the sshd service
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
if (!(Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" -ErrorAction SilentlyContinue | Select-Object Name, Enabled)) {
    Write-Output "Firewall Rule 'OpenSSH-Server-In-TCP' does not exist, creating it..."
    New-NetFirewallRule -Name 'OpenSSH-Server-In-TCP' -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
} else {
    Write-Output "Firewall rule 'OpenSSH-Server-In-TCP' has been created and exists."
}

$shellParams = @{
    Path         = 'HKLM:\SOFTWARE\OpenSSH'
    Name         = 'DefaultShell'
    Value        = 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
    PropertyType = 'String'
    Force        = $true
}
New-ItemProperty @shellParams

# Set default to powershell.exe
$shellParams = @{
    Path         = 'HKLM:\SOFTWARE\OpenSSH'
    Name         = 'DefaultShell'
    Value        = 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe'
    PropertyType = 'String'
    Force        = $true
}
New-ItemProperty @shellParams
```
### 💻 Running Ansible on Windows using WSL
Ansible is normally run on a linux control node but requires installation as well

**Setup Steps:**
1. Update packages and install Ansible:
   ```bash
   sudo apt update
   sudo apt install ansible sshpass pip python3-winrm -y
   ```

2. You can now run Ansible like any other Linux system:
   ```bash
   ansible --version
   ```

### Setup linux host for ansible
Connection to linux host can be authenticated via SSH keys
```bash
# Add ansible control node public SSH key to linux host
ssh-copy-id <user>@<ip of linux host>
```
### Install cockpit for easy management of VMs on debian
If you're running a linux machine and want to create VMs you can create cockpit to easily manage them without running a hypervisor OS

**Setup steps:**
```bash
# Change user to root
sudo su
# Source /etc/os-release
. /etc/os-release
# Add backports repository
echo "deb http://deb.debian.org/debian ${VERSION_CODENAME}-backports main" > \
    /etc/apt/sources.list.d/backports.list
# Update package list
apt update
# Install cockpit and cockpit-machines
apt install -t ${VERSION_CODENAME}-backports cockpit cockpit-machines
```

## 🧪 Demo Plan

### 1. Ad-hoc Commands
Linux and windows clients needs to be handled separately
```bash
# Ping all linux hosts
ansible linux -i inventory.yml -m ping

# Ping all windows hosts
ansible windows -i inventory.yml -m win_ping

# Install htop package on linux hosts
# This uses apt which is the package manager on Debian Linux
ansible linux -i inventory.yml -m apt -a "name=htop state=present"
# Remove htop
ansible linux -i inventory.yml -m apt -a "name=htop state=absent"
```

### 2. Simple Playbook - Install and Configure

```yaml
# linux.yaml
- name: Run linux play
  hosts: linux
  become: true
  tasks:
    - name: Install htop
      ansible.builtin.package:
        name: htop
        state: present
```

```bash
ansible-playbook -i inventory.yml playbooks/linux.yml
```

### 3. Windows Module Example

```yaml
- name: Windows firewall rule example
  hosts: windows
  tasks:
    - name: Enable WinRM rule
      ansible.windows.win_firewall_rule:
        name: "Allow WinRM"
        enable: yes
```

```bash
ansible-playbook -i inventory.yml playbooks/windows.yml
```

### 4. Using Facts

```yaml
- name: Show OS info
  hosts: all
  tasks:
    - debug:
        var: ansible_facts['os_family']
```
```bash
ansible-playbook -i inventory.yml playbooks/all.yml
```
### 5. Example Role Structure

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yaml
    └── templates/
        ├── site.html.j2
        └── site.cfg.j2
```

`roles/nginx/tasks/main.yaml`:
```yaml
---
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present

- name: Deploy NGINX site config
  ansible.builtin.template:
    src: site.cfg.j2
    dest: /etc/nginx/sites-available/{{ nginx_site_name }}
    mode: '0640'
    owner: "{{ nginx_user }}"
    group: "{{ nginx_group }}"
  notify: Reload nginx

- name: Enable site
  ansible.builtin.file:
    src: /etc/nginx/sites-available/{{ nginx_site_name }}
    dest: /etc/nginx/sites-enabled/{{ nginx_site_name }}
    state: link
    force: true

- name: Disable default site
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
    force: true

- name: Create web root
  ansible.builtin.file:
    path: "{{ item.web_root }}"
    state: directory
    mode: '0755'
    owner: "{{ nginx_user }}"
    group: "{{ nginx_group }}"

- name: Deploy index.html
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ nginx_web_root }}/index.html"
    mode: '0644'
    owner: "{{ nginx_user }}"
    group: "{{ nginx_group }}"

- name: Ensure NGINX is running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true

```

`roles/nginx/templates/site.html.j2`:
```html+jinja
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ nginx_site_title }}</title>
</head>
<body>
    <h1>Welcome to {{ nginx_site_title }}</h1>
    <p>This page is served by NGINX and deployed with Ansible!</p>
</body>
</html>
```

`roles/nginx/handlers/main.yml`:
```html+jinja
- name: Reload nginx
  service:
    name: nginx
    state: reloaded
```

`roles/nginx/templates/site.cfg.j2`:
```jinja
server {
    listen 80;
    server_name {{ nginx_hostname }};
    root {{ nginx_web_root }};

    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
```bash 
ansible-playbook -i inventory.yml playbooks/role.yml
```
Playbook using the role:

`playbooks/role.yml`
```yaml
- name: Run nginx role
  hosts: linux
  become: true
  roles:
    - nginx

```

### 6. Example Inventory File

`inventory.yml`:
```yaml
all:
  hosts:
    deb12_02:
      ansible_host: 192.168.137.130
      ansible_user: root
      ansible_python_interpreter: /usr/bin/python3
    win2025_01:
      ansible_host: 192.168.137.129
      ansible_user: jonas
      ansible_password: Secret1!
      ansible_connection: ssh
      ansible_shell_type: powershell
  children:
    windows:
      hosts:
        win2025_01:
    linux:
      hosts:
        deb12_02:
```
## 💡 Takeaways

- Ansible is agentless, secure, and easy to start.
- You define **what** you want, not **how** to do it.
- YAML is readable and encourages collaboration.
- Works across **Linux and Windows**!

## ✅ Tips

- Use `--check` to dry run a playbook
- Group hosts in inventory with `[groupname]`
- Use `become: true` to run as root/sudo
- You can use things like taskfile.dev to run ansible commands for ease of use

## 🧰 Tools & Resources

- [Ansible Docs](https://docs.ansible.com/)
- [Managing Windows hosts with Ansible](https://docs.ansible.com/ansible/latest/os_guide/intro_windows.html)
- [Ansible on azure](https://learn.microsoft.com/en-us/azure/developer/ansible/getting-started-cloud-shell?tabs=ansible)
