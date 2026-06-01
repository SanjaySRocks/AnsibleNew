# Production-ready Ansible project for Linux and Windows tooling

This repository contains a simple Ansible project that installs Python, OpenJDK 17, .NET SDK 8, and Docker on Linux and Windows targets. Windows package installation is handled with Chocolatey, which is installed by a dedicated role when it is missing.

## Folder structure

```text
.
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── group_vars/
│   └── all.yml
├── inventory.ini
├── README.md
├── roles/
│   ├── chocolatey/
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── docker/
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── dotnet/
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   ├── java/
│   │   ├── defaults/main.yml
│   │   └── tasks/main.yml
│   └── python/
│       ├── defaults/main.yml
│       └── tasks/main.yml
└── site.yml
```

## File purposes

- `ansible.cfg`: Project-local Ansible defaults, including the inventory and role path.
- `collections/requirements.yml`: External Ansible collections required for Windows and Chocolatey modules.
- `inventory.ini`: Single example inventory containing Linux and Windows host groups.
- `site.yml`: Single playbook that targets all hosts and applies every role.
- `group_vars/all.yml`: Shared variables for package names, SDK versions, and role behavior.
- `roles/chocolatey`: Installs Chocolatey on Windows if it is not already present.
- `roles/python`: Installs Python on Linux and Windows.
- `roles/java`: Installs OpenJDK 17 on Linux and Windows.
- `roles/dotnet`: Installs .NET SDK 8 on Linux and Windows.
- `roles/docker`: Installs Docker on Linux and Docker Desktop on Windows.

## Requirements

Install required collections on the Ansible control node:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

Linux hosts must be reachable over SSH. Windows hosts must have WinRM enabled and reachable from the control node. Store real Windows credentials with Ansible Vault instead of plaintext inventory values.

## Run commands

Run against all hosts:

```bash
ansible-playbook -i inventory.ini site.yml
```

Run against only Linux hosts:

```bash
ansible-playbook -i inventory.ini site.yml --limit linux
```

Run against only Windows hosts:

```bash
ansible-playbook -i inventory.ini site.yml --limit windows
```

## Idempotency notes

The playbook uses package modules with `state: present`, checks for Chocolatey before installing it, enables services declaratively, and only reboots Windows for Docker Desktop when `docker_windows_allow_reboot` is set to `true` and the package task changed the host.

## Full file contents

### `ansible.cfg`

```ini
# Purpose: Keep common Ansible settings with the project so commands behave consistently.
[defaults]
inventory = inventory.ini
roles_path = roles
host_key_checking = False
retry_files_enabled = False
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
```

### `collections/requirements.yml`

```yaml
---
# Purpose: Declare external collections required by this project.
collections:
  - name: ansible.windows
  - name: chocolatey.chocolatey
```

### `inventory.ini`

```ini
# Purpose: Single inventory for both Linux and Windows targets.
# Replace hostnames/IPs and credentials with your environment-specific values.

[linux]
linux01 ansible_host=192.0.2.10 ansible_user=ubuntu ansible_become=true

[windows]
win01 ansible_host=192.0.2.20

[windows:vars]
# Windows hosts use WinRM. Prefer Ansible Vault for real credentials.
ansible_connection=winrm
ansible_port=5986
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_user=Administrator
ansible_password=ChangeMeUseAnsibleVault

[all:vars]
# Common defaults; override per host/group as needed.
ansible_python_interpreter=auto_silent
```

### `group_vars/all.yml`

```yaml
---
# Purpose: Shared variables for all hosts and roles.

# Package versions/channels used by the roles.
python_linux_package: python3
python_windows_package: python
java_linux_package_debian: openjdk-17-jdk
java_linux_package_redhat: java-17-openjdk-devel
java_windows_package: openjdk17
dotnet_sdk_version: "8.0"
docker_linux_package: docker.io
docker_windows_package: docker-desktop

# Docker Desktop installs on Windows but usually requires a reboot before first use.
docker_windows_allow_reboot: false
```

### `site.yml`

```yaml
---
# Purpose: Single playbook that installs core developer/runtime tooling on Linux and Windows.
# It relies on Ansible facts to detect each host OS and lets each role choose the right tasks.

- name: Install common development tooling on Linux and Windows
  hosts: all
  gather_facts: true

  roles:
    # Chocolatey is Windows-only and must run before Windows package roles.
    - role: chocolatey
    - role: python
    - role: java
    - role: dotnet
    - role: docker
```

### `roles/chocolatey/defaults/main.yml`

```yaml
---
# Purpose: Default variables for the Chocolatey role.

chocolatey_install_script_url: https://community.chocolatey.org/install.ps1
```

### `roles/chocolatey/tasks/main.yml`

```yaml
---
# Purpose: Install Chocolatey only on Windows hosts so later Windows package tasks can use it.

- name: Check if Chocolatey is already installed
  ansible.windows.win_stat:
    path: C:\ProgramData\chocolatey\bin\choco.exe
  register: chocolatey_exe
  when: ansible_os_family == 'Windows'

- name: Install Chocolatey on Windows when missing
  ansible.windows.win_powershell:
    script: |
      Set-ExecutionPolicy Bypass -Scope Process -Force
      [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
      Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('{{ chocolatey_install_script_url }}'))
  when:
    - ansible_os_family == 'Windows'
    - not (chocolatey_exe.stat.exists | default(false))

- name: Ensure Chocolatey is available in the current session path
  ansible.windows.win_path:
    elements:
      - C:\ProgramData\chocolatey\bin
    state: present
  when: ansible_os_family == 'Windows'
```

### `roles/python/defaults/main.yml`

```yaml
---
# Purpose: Default variables for the Python role.

python_linux_package: python3
python_windows_package: python
```

### `roles/python/tasks/main.yml`

```yaml
---
# Purpose: Install Python on Linux and Windows using the native package mechanism for each OS.

- name: Install Python on Linux
  ansible.builtin.package:
    name: "{{ python_linux_package }}"
    state: present
  become: true
  when: ansible_os_family != 'Windows'

- name: Install Python on Windows with Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ python_windows_package }}"
    state: present
  when: ansible_os_family == 'Windows'
```

### `roles/java/defaults/main.yml`

```yaml
---
# Purpose: Default variables for the Java role.

java_linux_package_debian: openjdk-17-jdk
java_linux_package_redhat: java-17-openjdk-devel
java_windows_package: openjdk17
```

### `roles/java/tasks/main.yml`

```yaml
---
# Purpose: Install OpenJDK 17 on Linux and Windows.

- name: Install OpenJDK 17 on Debian-family Linux
  ansible.builtin.apt:
    name: "{{ java_linux_package_debian }}"
    state: present
    update_cache: true
    cache_valid_time: 3600
  become: true
  when:
    - ansible_os_family == 'Debian'

- name: Install OpenJDK 17 on RedHat-family Linux
  ansible.builtin.dnf:
    name: "{{ java_linux_package_redhat }}"
    state: present
  become: true
  when:
    - ansible_os_family == 'RedHat'

- name: Install OpenJDK 17 on other Linux distributions
  ansible.builtin.package:
    name: "{{ java_linux_package_debian }}"
    state: present
  become: true
  when:
    - ansible_os_family not in ['Windows', 'Debian', 'RedHat']

- name: Install OpenJDK 17 on Windows with Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ java_windows_package }}"
    state: present
  when: ansible_os_family == 'Windows'
```

### `roles/dotnet/defaults/main.yml`

```yaml
---
# Purpose: Default variables for the .NET SDK role.

dotnet_sdk_version: "8.0"
```

### `roles/dotnet/tasks/main.yml`

```yaml
---
# Purpose: Install .NET SDK 8 on Linux and Windows.

- name: Install Microsoft package repository on Debian-family Linux
  ansible.builtin.apt:
    deb: "https://packages.microsoft.com/config/{{ ansible_distribution | lower }}/{{ ansible_distribution_major_version }}/packages-microsoft-prod.deb"
    state: present
  become: true
  when:
    - ansible_os_family == 'Debian'

- name: Install .NET SDK on Debian-family Linux
  ansible.builtin.apt:
    name: "dotnet-sdk-{{ dotnet_sdk_version }}"
    state: present
    update_cache: true
  become: true
  when:
    - ansible_os_family == 'Debian'

- name: Install .NET SDK on RedHat-family Linux
  ansible.builtin.dnf:
    name: "dotnet-sdk-{{ dotnet_sdk_version }}"
    state: present
  become: true
  when:
    - ansible_os_family == 'RedHat'

- name: Install .NET SDK on Windows with Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "dotnet-{{ dotnet_sdk_version }}-sdk"
    state: present
  when: ansible_os_family == 'Windows'
```

### `roles/docker/defaults/main.yml`

```yaml
---
# Purpose: Default variables for the Docker role.

docker_linux_package: docker.io
docker_windows_package: docker-desktop
docker_windows_allow_reboot: false
```

### `roles/docker/tasks/main.yml`

```yaml
---
# Purpose: Install Docker on Linux and Docker Desktop on Windows.

- name: Install Docker on Debian-family Linux
  ansible.builtin.apt:
    name: "{{ docker_linux_package }}"
    state: present
    update_cache: true
    cache_valid_time: 3600
  become: true
  when:
    - ansible_os_family == 'Debian'

- name: Install Docker on RedHat-family Linux
  ansible.builtin.dnf:
    name: docker
    state: present
  become: true
  when:
    - ansible_os_family == 'RedHat'

- name: Ensure Docker service is enabled and running on Linux
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true
  become: true
  when: ansible_os_family != 'Windows'

- name: Install Docker Desktop on Windows with Chocolatey
  chocolatey.chocolatey.win_chocolatey:
    name: "{{ docker_windows_package }}"
    state: present
  register: docker_windows_install
  when: ansible_os_family == 'Windows'

- name: Reboot Windows if Docker Desktop requires it and reboots are enabled
  ansible.windows.win_reboot:
    msg: Rebooting after Docker Desktop installation.
  when:
    - ansible_os_family == 'Windows'
    - docker_windows_allow_reboot | bool
    - docker_windows_install is defined
    - docker_windows_install is changed
```
