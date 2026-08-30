# 3xRPi-ops — Ansible for the RPi fleet

Config management for the Raspberry Pi fleet (Ubuntu) on the LAN.
Separate from `~/3xRPi` (that repo is monitoring *docs*; this one is *ops*).

> **Nowy tutaj? Zacznij od `docs/SCIAGA.md`** — jedna kartka po polsku: mapa repo,
> pięć komend, pułapki tej floty i stan na dziś. Kurs Ansible od zera:
> `docs/wprowadzenie-ansible.md`. Ten plik (README) jest po angielsku i jest
> **referencją techniczną**, nie materiałem do nauki — trzyma uzasadnienia decyzji.

## Fleet

| host   | IP            | user | MAC (wlan0)       | OS          | node_exporter |
|--------|---------------|------|-------------------|-------------|---------------|
| rpi-01 | 192.168.0.170 | mwd  | 88:a2:9e:27:38:7a | Ubuntu 26.04 | up           |
| rpi-02 | 192.168.0.172 | mwd  | 88:a2:9e:27:39:49 | **Ubuntu 24.04** | **absent** |
| rpi-03 | 192.168.0.100 | mwd  | 88:a2:9e:27:2c:be | Ubuntu 26.04 | up           |

**Addresses verified 2026-08-30 00:02 by SSH login on all three.** This is the
FOURTH drift; every earlier mapping (`.212/.213/.145`, `.102/.106/.151`,
`.170/.169/.249`) is dead. MAC-based DHCP reservations on the router are the only
known fix — see the note under "Making the boards twins" about what is NOT the fix.

**The fleet is no longer homogeneous.** rpi-01 and rpi-03 run Ubuntu 26.04 with
kernel `7.0.0-1016-raspi`; rpi-02 came back on a fresh, never-provisioned Ubuntu
24.04 image (kernel `6.8.0-1047-raspi`, 131 pending updates, no `pip`, no
node_exporter, `sudo` needs a password). A `site.yml` run therefore **fails on
rpi-02** — see the node_exporter caveat below. Full 67-field comparison:
`~/3xRPi/docs/POROWNANIE-FLOTY-2026-08-30.md`.

Hardware, all three: Raspberry Pi 5 Model B Rev 1.1, 16 GB RAM, Cortex-A76 x4 @
2.4 GHz, **Wi-Fi only — `eth0` is down on every board.** All three still report the
same hostname `MWDRPi` (cloned SD image), so do not use `hostname` to tell them
apart — and note the MAC identifies the BOARD, not the system on its card.

## Layout

```
ansible.cfg                inventory + password-prompt defaults
inventory/
  hosts.ini                the 3 Pi's, group [rpi]
  group_vars/rpi.yml       ansible_user, python interpreter
  host_vars/rpi-0*.yml     per-host load profiles for the stress demo
playbooks/
  site.yml                 baseline + node_exporter (standing config)
  identity.yml             ONE-OFF: unique hostname/machine-id/host keys + SSH key
  update.yml               apt update + safe upgrade (+ opt-in reboot)
  stress.yml               differentiated CPU/RAM/IO/net load per host (monitoring demo)
roles/
  baseline/                packages, timezone, unattended-upgrades
  identity/                de-clone the boards: hostname, machine-id, host keys, SSH key
  node_exporter/           ensure existing :9100 exporter is up (adopts upstream install)
  app/                     PLACEHOLDER — deployment target undecided
```

## Making the boards twins (but distinguishable)

The fleet was built from one SD image, and on 2026-08-29 a third clone was
added: rpi-03's card stopped booting, so its board was re-flashed from a copy
of rpi-02's card. The result is a fleet that is **identical in all the ways
that hurt** — same hostname `MWDRPi`, same `/etc/machine-id`, same SSH host
keys — and different in none of the ways that help.

`playbooks/identity.yml` fixes exactly that, and nothing else:

| What | Why it matters |
|---|---|
| one admin account (`mwd`) with a deployed SSH key | one login method for the whole fleet, no more password prompts |
| hostname **`MWDRPi-1` / `MWDRPi-2` / `MWDRPi-3`** | tells the boards apart from the inside, which `MWDRPi` on all three does not. Set in `inventory/group_vars/rpi.yml` as `fleet_hostname`, derived from the alias (`rpi-0N` -> `MWDRPi-N`). **Note the alias and the hostname are deliberately different**: the alias binds to the board MAC and is what `host` labels in `prometheus.yml` use |
| fresh `/etc/machine-id` per board | systemd derives its **DHCP DUID** from it; clones present one identity to the router and can fight over a single lease |
| fresh `/etc/ssh/ssh_host_*` per board | a `dd` clone copies these, so today one key answers under three addresses |

```bash
# once, on this machine
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_rpi -C "3xRPi fleet"

cd ~/3xRPi-ops
ansible-playbook playbooks/identity.yml --limit rpi-01 --check --diff   # look first
ansible-playbook playbooks/identity.yml --limit rpi-01                  # then apply
```

Do **one board at a time**. A new machine-id means a new DHCP DUID, so a board
may come back on a different address after its reboot; find it again with
`~/3xRPi/scripts/find_pi.sh` and drop the stale key with `ssh-keygen -R <ip>`.
Re-running the playbook is a no-op — the guarded steps are stamped in
`/etc/fleet-identity`.

Once every board answers on the key, switch the project off passwords:
set `fleet_disable_password_auth: true`, re-run, then drop `ask_pass` from
`ansible.cfg` and point `group_vars/rpi.yml` at the private key. Not before —
these boards are headless.

### What this playbook will not do

It does not touch netplan. Every board is **Wi-Fi only** (`eth0` is `down` on
all of them, traffic goes over `wlan0`), so a broken network config leaves no
way back in short of a monitor and a keyboard. Address stability belongs on
the router instead, as DHCP reservations for the **wlan0** MACs:

```
rpi-01  88:a2:9e:27:38:7a
rpi-02  88:a2:9e:27:39:49
rpi-03  88:a2:9e:27:2c:be
```

Note the trap if those reservations appear not to work: systemd-networkd sends
a DUID as DHCP option 61 by default, and many routers match reservations on
that rather than on the MAC. Giving each board its own machine-id (above) makes
those DUIDs distinct, which is the prerequisite for reservations to behave. If
they still drift after that, the remaining fix is `dhcp-identifier: mac` in
netplan — worth doing with a screen attached, not remotely.

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

# reboot — one board at a time, and only those that ask for it
ansible-playbook playbooks/reboot.yml
ansible-playbook playbooks/reboot.yml -e force=true      # all of them, unconditionally
ansible-playbook playbooks/reboot.yml --limit rpi-02     # just one

# monitoring demo: put a DIFFERENT load on each Pi for 5 min (async, self-stops)
ansible-playbook playbooks/stress.yml
ansible-playbook playbooks/stress.yml -e duration=120   # shorter
ansible rpi -b -a "pkill -f stress-ng"                  # stop early
```

Each Pi gets its own load profile from `inventory/host_vars/rpi-0*.yml`, so the
node_exporter graphs show three distinct traces. Full walkthrough:
[docs/demo-obciazenie.md](docs/demo-obciazenie.md).

You'll be prompted for `SSH password:` and `BECOME password:` (both are `mwd`'s
password / sudo password on the Pi's).

### node_exporter — adopts what is there, installs what is missing

Recon (2026-07-02) settled what's actually on the Pi's: the official upstream
node_exporter **v1.11.1**, binary at `/usr/local/bin/node_exporter`, run by a
systemd unit named **`node_exporter`** (not the apt `prometheus-node-exporter`),
enabled and listening on :9100 on all three hosts.

That upstream build is *newer* than Ubuntu's apt package, so migrating to apt
would be a downgrade plus a monitoring gap for no gain. Where the binary is
already present the role therefore **adopts** the existing service: it only
ensures `node_exporter` is started, enabled and listening, and does not touch
the binary or the unit. A clean run there is all `ok` / no `changed`.

**Since 2026-08-30 the role also installs it where it is missing.** Until then it
only adopted, which is why a `site.yml` run died on rpi-02 — that board came back
on a fresh image and never had node_exporter at all. The install path is
transcribed from `install_node_exporter()` in the original hand-written
`setup_rpi_monitoring.sh` (recovered from `rpi-03:~/` on 30-08, now kept at
`~/3xRPi/scripts/setup_rpi_monitoring.sh`), so a board provisioned by Ansible is
indistinguishable from the two set up by hand: upstream tarball for the detected
arch, **verified against the release's `sha256sums.txt`**, binary installed to
`/usr/local/bin/node_exporter`, a system account `node_exporter` with no login
and no home, and a systemd unit with `Restart=on-failure`.

Two deliberate details:

- **Do NOT run `setup_rpi_monitoring.sh` on a fleet board to get node_exporter.**
  It predates the 2026-07-02 decision that the Pi's run only node_exporter: its
  `main()` calls `install_prometheus` unconditionally and `start_services` does
  `systemctl enable --now prometheus node_exporter`. There is a `--no-grafana`
  flag but no `--no-prometheus`. The role exists precisely so you do not have to
  touch that script.
- Set `node_exporter_install_missing: false` if you want the old behaviour, where
  a missing binary is a hard error rather than something the role fixes.

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
   for ip in 192.168.0.102 192.168.0.106; do ssh-copy-id mwd@$ip; done
   ```
   Confirmed still outstanding on 2026-08-09: `~/.ssh` holds only
   `known_hosts`, and `ssh -o BatchMode=yes mwd@…` returns
   `Permission denied (publickey,password)` on both hosts.
   Then set `ask_pass = False` in `ansible.cfg` and point
   `ansible_ssh_private_key_file` at the key in `group_vars/rpi.yml`.
2. Once on keys, add opt-in **SSH hardening** to the baseline role (disable
   `PasswordAuthentication`) — safe only after keys work.
3. Decide what the **app** role deploys and flesh it out.
