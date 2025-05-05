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
- **Windows Server VM**
- **Rocky Linux 9 VM**

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
### 💻 Running Ansible on Windows using WSL

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
   sudo apt update && sudo apt upgrade -y
   sudo apt install ansible -y
   ```

4. You can now run Ansible like any other Linux system:
   ```bash
   ansible --version
   ```

WSL is a great way to get started with Ansible without needing a full Linux VM.

### Setup windows host for ansible
When configuring windows WinRM needs to be activated. 
> This may pose a security risk so make sure to configure the firewall properly in a production settings

```powershell
Get-WindowsCapability -Name OpenSSH.Server* -Online |
    Add-WindowsCapability -Online
Set-Service -Name sshd -StartupType Automatic -Status Running

$firewallParams = @{
    Name        = 'sshd-Server-In-TCP'
    DisplayName = 'Inbound rule for OpenSSH Server (sshd) on TCP port 22'
    Action      = 'Allow'
    Direction   = 'Inbound'
    Enabled     = 'True'  # This is not a boolean but an enum
    Profile     = 'Any'
    Protocol    = 'TCP'
    LocalPort   = 22
}
New-NetFirewallRule @firewallParams

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

# Set default back to cmd.exe
Remove-ItemProperty -Path HKLM:\SOFTWARE\OpenSSH -Name DefaultShell
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

## 🧪 Demo Plan

### 1. Ad-hoc Commands

```bash
# Install ansible
sudo apt install -y ansible
# Ping all hosts
ansible all -i inventory.yml -m ping
# Install htop package on linux hosts
# This uses yum which is the package manager on Rocky Linux
ansible linux -i inventory.yml -m yum -a "name=htop state=present"
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
ansible-playbook -i inventory.yml linux.yaml
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
ansible-playbook -i inventory.yml all.yaml
```

### 4. Using Facts

```yaml
- name: Show OS info
  hosts: all
  tasks:
    - debug:
        var: ansible_facts['os_family']
```

### 5. Example Role Structure

```
roles/
└── common/
    ├── tasks/
    │   └── main.yaml
    └── files/
        └── example.txt
```

`roles/common/tasks/main.yaml`:
```yaml
---
- name: Ensure example file exists
  copy:
    src: example.txt
    dest: /tmp/example.txt
```

Playbook using the role:
```yaml
- name: Use common role
  hosts: all
  become: true
  roles:
    - common
```

### 6. Example Inventory File

`inventory.ini`:
```yaml
all:
  hosts:
    rocky01:
      ansible_host: 192.168.122.135
      ansible_user: root
      ansible_ssh_private_key_file: ~/.ssh/id_rsa
    win01:
      ansible_host: 192.168.122.136
      ansible_user: administrator
      # There is probably more secure ways to connect then using password
      ansible_password: "secret"
  children:
    windows:
      hosts:
        win01:
    linux:
      hosts:
        rocky01:
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
