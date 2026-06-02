# Ansible Linux and Windows Tooling Automation

This repository contains an Ansible project for installing common development and runtime tooling on Linux and Windows servers from a single playbook.

The playbook installs:

- Python
- OpenJDK 17
- .NET SDK 8
- Docker on Linux
- Docker Desktop on Windows
- Chocolatey on Windows, when it is missing

## Repository contents

```text
.
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── group_vars/
│   └── all.yml
├── inventory.ini
├── roles/
│   ├── chocolatey/
│   ├── docker/
│   ├── dotnet/
│   ├── java/
│   └── python/
└── site.yml
```

## How it works

- `site.yml` is the main playbook. It targets all hosts in the inventory and applies every role.
- `inventory.ini` defines the Linux and Windows servers that Ansible manages.
- `group_vars/all.yml` stores shared package names, versions, and role settings.
- `collections/requirements.yml` lists the Ansible collections required for Windows and Chocolatey modules.
- `roles/chocolatey` installs Chocolatey on Windows before the Windows package roles run.
- `roles/python`, `roles/java`, `roles/dotnet`, and `roles/docker` install the required tools for each operating system.

## Example server layout

The example inventory is written for three servers:

| Server purpose | Operating system | Inventory host name | Group |
| --- | --- | --- | --- |
| ITHEM server | Linux | `ithem_server` | `linux` |
| Development environment server | Windows | `dev_env_server` | `windows_dev` |
| UAT environment server | Windows | `uat_env_server` | `windows_uat` |

The Windows environment groups are also included under the parent `windows` group, so you can run the playbook against both Windows servers together.

## Example inventory

Update `inventory.ini` with the real IP addresses, users, and credentials for your servers before running the playbook.

```ini
[linux]
ithem_server ansible_host=192.0.2.10 ansible_user=ubuntu ansible_become=true

[windows_dev]
dev_env_server ansible_host=192.0.2.20

[windows_uat]
uat_env_server ansible_host=192.0.2.30

[windows:children]
windows_dev
windows_uat

[windows:vars]
ansible_connection=winrm
ansible_port=5986
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_user=Administrator
ansible_password=ChangeMeUseAnsibleVault

[all:vars]
ansible_python_interpreter=auto_silent
```

> Do not commit real passwords to Git. Store production credentials with Ansible Vault or another approved secrets manager.

## Requirements

### Control node

Run Ansible from a Linux/macOS control node or from Windows Subsystem for Linux.

Install Ansible and the required collections:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### Linux managed server

The Linux server must:

- Be reachable from the control node over SSH.
- Have a valid SSH user configured in `inventory.ini`.
- Allow privilege escalation if packages must be installed with `become: true`.

### Windows managed servers

The Windows servers must:

- Be reachable from the control node over WinRM.
- Allow the configured Windows account to install software.
- Have network access to Chocolatey and package download locations.

## Usage

### 1. Clone the repository

```bash
git clone <your-repository-url>
cd <your-repository-directory>
```

### 2. Install required collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### 3. Update the inventory

Edit `inventory.ini` and replace the example values with your real server details:

- Replace `192.0.2.10` with the ITHEM Linux server IP address or DNS name.
- Replace `192.0.2.20` with the development Windows server IP address or DNS name.
- Replace `192.0.2.30` with the UAT Windows server IP address or DNS name.
- Replace the example Windows username and password with secure credentials.

### 4. Test connectivity

Check all hosts:

```bash
ansible all -i inventory.ini -m ping
```

Check only the Linux server:

```bash
ansible linux -i inventory.ini -m ping
```

Check only the Windows servers:

```bash
ansible windows -i inventory.ini -m win_ping
```

### 5. Run the playbook

Run against all three servers:

```bash
ansible-playbook -i inventory.ini site.yml
```

Run only against the ITHEM Linux server:

```bash
ansible-playbook -i inventory.ini site.yml --limit ithem_server
```

Run only against the Windows development server:

```bash
ansible-playbook -i inventory.ini site.yml --limit dev_env_server
```

Run only against the Windows UAT server:

```bash
ansible-playbook -i inventory.ini site.yml --limit uat_env_server
```

Run against both Windows servers:

```bash
ansible-playbook -i inventory.ini site.yml --limit windows
```

Run against only the Windows development environment group:

```bash
ansible-playbook -i inventory.ini site.yml --limit windows_dev
```

Run against only the Windows UAT environment group:

```bash
ansible-playbook -i inventory.ini site.yml --limit windows_uat
```

## Configuration

Default variables are stored in `group_vars/all.yml` and in each role's `defaults/main.yml` file.

Common values you may want to change:

```yaml
python_linux_package: python3
python_windows_package: python
java_linux_package_debian: openjdk-17-jdk
java_linux_package_redhat: java-17-openjdk-devel
java_windows_package: openjdk17
dotnet_sdk_version: "8.0"
docker_linux_package: docker.io
docker_windows_package: docker-desktop
docker_windows_allow_reboot: false
```

Set `docker_windows_allow_reboot: true` only if Ansible is allowed to reboot Windows servers after Docker Desktop installation.

## Security notes

- Use Ansible Vault for Windows passwords and other secrets.
- Keep `ansible_winrm_server_cert_validation=ignore` only for test environments or trusted internal networks.
- Use least-privilege accounts where possible.
- Review package sources before using this playbook in production.

## Troubleshooting

### Windows connection fails

Verify that WinRM is enabled, the firewall allows WinRM traffic, and the configured credentials are correct.

### Linux package installation fails

Verify that the Linux user can run privileged package installation commands and that the target server can reach its package repositories.

### Docker Desktop needs a reboot

Docker Desktop on Windows may require a reboot before it works correctly. If reboots are allowed in your maintenance window, set:

```yaml
docker_windows_allow_reboot: true
```

Then rerun the playbook.

## License

This project is licensed under the terms of the repository license file.
