# 3xRPi-ops — Ansible for the RPi fleet

Config management for three Raspberry Pi hosts (Ubuntu Server) on the LAN.
Separate from `~/3xRPi` (that repo is monitoring *docs*; this one is *ops*).

## Fleet

| host   | IP            | user |
|--------|---------------|------|
| rpi-01 | 192.168.0.212 | mwd  |
| rpi-02 | 192.168.0.213 | mwd  |
| rpi-03 | 192.168.0.145 | mwd  |

## Layout

```
ansible.cfg                inventory + password-prompt defaults
inventory/
  hosts.ini                the 3 Pi's, group [rpi]
  group_vars/rpi.yml       ansible_user, python interpreter
playbooks/
  site.yml                 baseline + node_exporter (standing config)
  update.yml               apt update + safe upgrade (+ opt-in reboot)
roles/
  baseline/                packages, timezone, unattended-upgrades
  node_exporter/           ensure existing :9100 exporter is up (adopts upstream install)
  app/                     PLACEHOLDER — deployment target undecided
```

## Prerequisites (need sudo — run these yourself)

Auth is by **password** (no SSH keys yet), which means Ansible needs `sshpass`,
and a modern Ansible is nicer than the apt 2.10.8 currently installed:

```bash
sudo apt update
sudo apt install -y sshpass pipx
pipx ensurepath
pipx install --include-deps ansible        # modern ansible-core + collections
exec $SHELL -l                             # pick up the new PATH
ansible --version                          # confirm you're on the pipx one
```

`sshpass` is mandatory for password auth; without it `ask_pass` fails with
*"you must install the sshpass program"*. If you skip the pipx upgrade the
project still works on apt 2.10.8 (everything here is builtin-only).

## Usage

Run from the repo root. `ansible.cfg` auto-prompts for the SSH password and the
sudo (become) password — no flags needed.

```bash
cd ~/3xRPi-ops

# reachability
ansible rpi -m ping

# dry run first — ALWAYS for node_exporter (see caveat below)
ansible-playbook playbooks/site.yml --check --diff

# apply
ansible-playbook playbooks/site.yml

# patch the fleet
ansible-playbook playbooks/update.yml
ansible-playbook playbooks/update.yml -e reboot_if_required=true
```

You'll be prompted for `SSH password:` and `BECOME password:` (both are `mwd`'s
password / sudo password on the Pi's).

### node_exporter — adopts the existing upstream install

Recon (2026-07-02) settled what's actually on the Pi's: the official upstream
node_exporter **v1.11.1**, binary at `/usr/local/bin/node_exporter`, run by a
systemd unit named **`node_exporter`** (not the apt `prometheus-node-exporter`),
enabled and listening on :9100 on all three hosts.

That upstream build is *newer* than Ubuntu's apt package, so migrating to apt
would be a downgrade plus a monitoring gap for no gain. The role therefore
**adopts** the existing service: it only ensures `node_exporter` is started,
enabled, and listening — it does **not** install a package or own the
binary/unit. A clean run is all `ok` / no `changed`.

If you ever want Ansible to fully own the exporter (reproducible from code),
that's the "full IaC" path — `get_url` the release, template the systemd unit,
restart via a handler. Intentionally not done here.

## Environment constraints (baked into how this repo works)

- **sudo needs a password** on the Pi's (no NOPASSWD) → `become_ask_pass`.
- **No SSH keys yet** → password auth + `sshpass`. See "Next steps" to fix.
- Controller is **WSL2 Ubuntu**, which reaches the LAN `192.168.0.0/24` fine.
- If you ever add a **Docker**-based app role: the WSL Docker daemon has bridge
  networking off (`network_mode: host` only). The Pi's are separate hosts —
  verify their Docker config before assuming the same.

## Next steps

1. **Deploy SSH keys** (kills the password prompts, big quality-of-life win):
   ```bash
   ssh-keygen -t ed25519 -C "vision@Vision-DellLaPrv" -f ~/.ssh/id_ed25519 -N ""
   for ip in 192.168.0.212 192.168.0.213 192.168.0.145; do ssh-copy-id mwd@$ip; done
   ```
   Then set `ask_pass = False` in `ansible.cfg` and point
   `ansible_ssh_private_key_file` at the key in `group_vars/rpi.yml`.
2. Once on keys, add opt-in **SSH hardening** to the baseline role (disable
   `PasswordAuthentication`) — safe only after keys work.
3. Decide what the **app** role deploys and flesh it out.
