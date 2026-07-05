# Ansible Lab — Project 7

Automated baseline configuration of `client-vm` using Ansible, run from a
control node (`server-vm`) over SSH.

## Architecture

- **Control node:** `server-vm` (10.10.10.10) — Ansible 2.20.1
- **Managed node:** `client-vm` (10.10.10.20) — Ubuntu Server, targeted via
  `inventory.ini`
- **Network:** VirtualBox internal network `labnet` (10.10.10.0/24) for
  control-to-managed communication, plus a NAT adapter on `client-vm` for
  internet access (required for `apt` package installs)

## Files

| File            | Purpose                                              |
|-----------------|-------------------------------------------------------|
| `ansible.cfg`   | Disables deprecation warnings and host key checking  |
| `inventory.ini` | Defines `client-vm` under the `[managed]` group      |
| `setup.yml`     | Baseline setup playbook (see below)                  |

## Playbook: `setup.yml`

```yaml
---
- name: Basic server setup
  hosts: managed
  become: yes

  tasks:
    - name: Install useful tools
      apt:
        name:
          - curl
          - net-tools
        state: present
        update_cache: yes

    - name: Ensure UFW is installed
      apt:
        name: ufw
        state: present

    - name: Allow SSH through UFW
      ufw:
        rule: allow
        name: OpenSSH

    - name: Enable UFW
      ufw:
        state: enabled

    - name: Print success message
      debug:
        msg: "Server setup complete on {{ inventory_hostname }}"
```

## Running It

```bash
cd ~/ansible-lab
ansible-playbook -i inventory.ini setup.yml
```

## Troubleshooting Log

Three real issues came up bringing this playbook from draft to a clean run:

**1. YAML typo — `becomes` instead of `become`**
Ansible rejected the play with `[ERROR]: 'becomes' is not a valid attribute
for a Play`. Fixed by correcting the keyword to `become: yes`.

**2. Sudo required an interactive password**
Once the YAML was valid, tasks failed with:
sudo: interactive authentication is required
Ansible has no way to type a sudo password interactively mid-playbook. Fixed
by granting passwordless sudo to the managed-node user via `visudo`:
landry5545 ALL=(ALL) NOPASSWD: ALL

**3. `client-vm` had no internet route**
With sudo fixed, `apt` cache updates timed out repeatedly:
Failed to update apt cache after 5 retries
`client-vm` only had the internal `labnet` adapter (`enp0s3`), so it could
reach `server-vm` but not the internet. Fixed by:
- Adding a second NAT adapter in VirtualBox (`enp0s8`)
- Discovering `/etc/netplan/50-cloud-init.yaml` had incorrectly applied the
  same static `10.10.10.20/24` address to `enp0s8` as `enp0s3`
- Correcting that file so `enp0s8` uses `dhcp4: true`, letting it pull a
  proper NAT address (`10.0.3.x`) and reach the internet

After these fixes, `ansible-playbook -i inventory.ini setup.yml` completed
with `ok=6, changed=2, failed=0`.

## Next Steps

- Advanced playbook: install Docker, configure UFW rules
- Expand inventory as more managed nodes are added
