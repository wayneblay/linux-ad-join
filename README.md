# Linux Server Login with Active Directory (RHEL + Ubuntu, SSSD)

Lets team members log in to our Linux production servers (RHEL 7–10, Ubuntu 22.04+) with their **own Active Directory username and password**.
They can change their password on any server (or in Windows), and the change works on every server.
AD is the single source of truth — nothing is copied or synced to the servers.

**First time?** Follow [RUNBOOK.md](RUNBOOK.md) — a step-by-step checklist for piloting on one or two test servers.

> **Example values used throughout:** domain `bob.net` (realm `BOB.NET`), computer OU `OU=Linux,DC=bob,DC=net`,
> test servers `tester1` and `tester2`. Replace them with your real values **on the control node only** —
> this repository is public, so never commit real domain names, IPs or passwords.

---

## Contents
1. [How it works](#how-it-works) · [Supported systems](#supported-systems)
2. [What we gain](#what-we-gain)
3. [Prerequisites](#prerequisites)
4. [Active Directory setup](#active-directory-setup)
5. [Hybrid AD + Entra ID notes](#hybrid-ad--entra-id-notes)
6. [Network / firewall](#network--firewall)
7. [Control node (where Ansible runs)](#control-node-where-ansible-runs)
8. [Configure the playbook](#configure-the-playbook)
9. [Run it](#run-it)
10. [Test](#test)
11. [Day-to-day use](#day-to-day-use)
12. [Troubleshooting](#troubleshooting)
13. [Rollout plan & timeline](#rollout-plan--timeline)
14. [Repository layout](#repository-layout)

---

## How it works

Each Linux server is **joined to the AD domain** using SSSD and realmd (built into RHEL, no extra cost).

- When someone logs in, the server checks their password **with a domain controller in real time** (Kerberos).
- `passwd` on a Linux server changes the **AD password** (subject to AD password policy).
- If a password has expired, SSH prompts for a new one at login.
- Who can log in is controlled by **AD groups**:
  - `linux-prod-users` → can log in
  - `linux-prod-admins` → can log in **and** use `sudo`
- Being a Domain Admin does **not** grant access. Only membership in these groups does.
- The **join account** is only used once per server, to join it. After that, the server uses its own computer account (`TESTER1$`) and each person logs in with their own AD account.


## Supported systems

| OS | Versions | Notes |
|---|---|---|
| RHEL / Rocky / Alma | 8, 9, 10 | Main target |
| RHEL / CentOS | 7 | Works, but RHEL 7 is past end of life — plan an upgrade. See [RHEL 7 notes](#rhel-7-notes) |
| Ubuntu | 22.04 and newer | See [Ubuntu notes](#ubuntu-notes) |

The playbook detects each server's OS and uses the right steps for it, so one inventory and one run cover a mixed fleet.
It also checks first that Ansible can actually manage each server, installs Python 3 on servers that have none,
and stops with exact instructions if your Ansible version is too new for a server's Python (see [RHEL 7 notes](#rhel-7-notes)).

| Step | RHEL 8/9/10 | RHEL 7 | Ubuntu 22.04+ |
|---|---|---|---|
| Packages | `dnf` | `yum` | `apt` |
| SSSD logins + home dirs | `authselect` | `authconfig` | `pam-auth-update` |
| SSH config | drop-in (9+) / main file (8) | main file | drop-in |
| SSH service | `sshd` | `sshd` | `ssh` |

Domain join, `sssd.conf`, group-based access, sudo, logins and `passwd` work the same on all of them.

## What we gain

- **One login** on all servers — no local accounts to create or maintain.
- **Self-service passwords** — change once, works everywhere.
- **Access by group** — add someone to a group in AD and they have access.
- **Instant offboarding** — disable the account in AD and they are locked out of every server.
- **Optional SSO** — users on domain-joined Windows PCs can SSH without typing a password (Kerberos/GSSAPI).
- **Home directories** are created automatically on first login.

---

## Prerequisites

| Item | Notes |
|---|---|
| RHEL 7–10 (or Rocky/Alma), Ubuntu 22.04+ | Playbook checks this — see [Supported systems](#supported-systems) |
| On-prem Active Directory (AD DS) | Same AD the Windows PCs use |
| DNS | Servers must use AD DNS (or DNS that forwards to it) |
| Time sync | Clock must be within 5 minutes of the DCs. The RHEL NTP pool is fine if the DCs are also accurate |
| Network access to DCs | See [Network / firewall](#network--firewall) |
| A control node | Any Linux machine with SSH access to the servers |
| Ansible | Free — `ansible-core` from RHEL repos, no license needed |

### Find your AD domain name
The AD domain name is often **different** from your email domain. On a domain-joined Windows PC:

```cmd
echo %USERDNSDOMAIN%
```

or in PowerShell (with the AD module): `(Get-ADDomain).DNSRoot`

Then confirm from a Linux server:

```bash
sudo dnf install -y realmd
realm discover bob.net
```

You should see `type: kerberos` and `server-software: active-directory`.

---

## Active Directory setup

Nothing is installed on the domain controllers and no schema changes are needed.
SSSD generates consistent Linux UIDs/GIDs from each user's AD SID.

Run these in **PowerShell as administrator**, on a domain controller or on a machine with the AD module (RSAT).
Each part of the domain becomes a `DC=`: `bob.net` → `DC=bob,DC=net`.
Prefer clicking? Do the same in **Active Directory Users and Computers**.

### 1. Create an OU for the Linux servers

Create this first: the groups below are placed inside it, so it must exist before them.

```powershell
New-ADOrganizationalUnit -Name "Linux" -Path "DC=bob,DC=net"
```

### 2. Create the security groups

```powershell
New-ADGroup -Name "linux-prod-users"  -GroupScope Global -GroupCategory Security -Path "OU=Linux,DC=bob,DC=net"
New-ADGroup -Name "linux-prod-admins" -GroupScope Global -GroupCategory Security -Path "OU=Linux,DC=bob,DC=net"
```

Avoid spaces in group names. If you keep groups in an existing OU (e.g. `OU=Groups,DC=bob,DC=net`), use that path instead. Whatever `-Path` names must already exist.

### 3. Add people to the groups

```powershell
Add-ADGroupMember -Identity "linux-prod-admins" -Members jsmith
Add-ADGroupMember -Identity "linux-prod-users"  -Members mjones, akumar
```

Allow a minute or two for the change to reach all domain controllers.

### 4. Check it worked

```powershell
Get-ADOrganizationalUnit -Filter 'Name -eq "Linux"' | Select DistinguishedName   # OU=Linux,DC=bob,DC=net
Get-ADGroupMember "linux-prod-admins" | Select Name
Get-ADGroupMember "linux-prod-users"  | Select Name
```

No passwords are involved in steps 1–4.

### 5. Join account

To join a server, the playbook needs an AD account that is allowed to create computer objects in the Linux OU
(the same right used when a Windows PC is joined to the domain).

- **Pilot:** use your own AD admin account (Domain Admin, or rights delegated on the Linux OU). No service account needed.
- **Before full rollout:** create a dedicated account so your personal password isn't stored:

```powershell
New-ADUser -Name "svc_linuxjoin" -SamAccountName "svc_linuxjoin" `
  -Path "OU=Linux,DC=bob,DC=net" `
  -AccountPassword (Read-Host -AsSecureString "Enter password") `
  -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true
```

PowerShell prompts for the password (nothing appears as you type). Paste the whole command at once; the backticks continue it over several lines.

**Do not make it a Domain Admin.** Delegate rights on the Linux OU only:

1. **Active Directory Users and Computers** → right-click the **Linux** OU → **Delegate Control**
2. Add `svc_linuxjoin`
3. **Create a custom task to delegate** → **Only the following objects in the folder** → tick **Computer objects**
4. Tick **Create selected objects in this folder** and **Delete selected objects in this folder**
5. Under permissions tick **General** and **Property-specific**, then grant:
   - Reset password
   - Read and write account restrictions
   - Validated write to DNS host name
   - Validated write to service principal name

Then set `ad_join_user: svc_linuxjoin` and put its password in the vault.

### 6. DNS
AD's SRV records already exist. Make sure each Linux server has an **A record** (e.g. `tester1.bob.net`) and ideally a **PTR record**.

---

## Hybrid AD + Entra ID notes

If you use on-prem AD **and** Microsoft Entra ID (synced with Entra Connect / Cloud Sync), this works as-is.
The Linux servers talk to **on-prem AD**, not Entra.

| Where the user changes their password | Reaches the Linux servers? |
|---|---|
| Ctrl+Alt+Del on a Windows PC | Yes, immediately |
| `passwd` on a Linux server | Yes, immediately (and syncs up to Entra) |
| Microsoft 365 / Entra portal (SSPR) | **Only if Password Writeback is enabled** |

- **Password Writeback** must be on in Entra Connect, or cloud password resets won't reach the servers (requires Entra ID P1 or higher).
- **Accounts and groups must exist in on-prem AD.** Cloud-only Entra users can't log in. Create the Linux groups in on-prem AD.
- **Entra MFA / Conditional Access does not apply to SSH.** If MFA is needed on server logins, add it separately (e.g. bastion host or Duo).

Ask the AD/Entra admin:
1. Is Password Writeback enabled?
2. Are team accounts synced from on-prem AD (not cloud-only)?
3. What is the on-prem AD domain name?

---

## Network / firewall

Every login and password change is checked against a domain controller, so the servers must reach the DCs on these ports:

| Port | Protocol | Purpose |
|---|---|---|
| 53 | TCP/UDP | DNS — find the domain controllers |
| 88 | TCP/UDP | Kerberos — password check |
| 123 | UDP | Time sync (only needed if servers sync time from the DCs) |
| 389 | TCP/UDP | LDAP — user and group lookups |
| 464 | TCP/UDP | Kerberos password change (`passwd`) |
| 636 | TCP | LDAPS |
| 3268 | TCP | Global Catalog |

What breaks if blocked:
- **88** blocked → nobody can log in
- **464** blocked → logins work, password changes fail (most commonly missed)
- **53 / 389** blocked → server can't join the domain

### Test from a Linux server

Find the DC names first:

```bash
host -t SRV _ldap._tcp.bob.net      # last word on each line is a DC, e.g. dc01.bob.net.
```

Then paste this whole block at once (the `for` loop is one command):

```bash
DC=dc01.bob.net
for p in 53 88 389 464 636 3268; do
  timeout 3 bash -c "</dev/tcp/$DC/$p" 2>/dev/null && echo "$p open" || echo "$p BLOCKED"
done
```

All `open` → no firewall request needed. Any `BLOCKED` → request those ports (TCP **and** UDP for 53, 88, 464) from all servers to the DCs.
RHEL's own `firewalld` allows outbound traffic by default, so no change is needed on the servers.

---

## Control node (where Ansible runs)

Any Linux machine that can SSH to the servers: an admin/jump box, a small RHEL VM (1 vCPU / 2 GB RAM), or a Linux/macOS workstation (Windows → use WSL).
Nothing needs to be installed on the target servers.

```bash
# 1. Install Ansible (free, from RHEL AppStream)
sudo dnf install -y ansible-core

# 2. Get the files - no GitHub login needed
cd ~
curl -L -o linux-ad-join.zip https://github.com/wayneblay/linux-ad-join/archive/refs/heads/main.zip
unzip linux-ad-join.zip
cd ~/linux-ad-join-main

# 3. SSH key access to each server (root, or an account with sudo) - skip if already set up
ssh-keygen -t ed25519
ssh-copy-id root@<TESTER1_IP>
```

- Keep the Ansible connection account **local** (not AD) so you can still manage servers if AD is down. It doubles as break-glass access.
- The control node holds keys to every production server — lock it down.

**Licensing:** `ansible-core` is free and open source. Red Hat Ansible Automation Platform (paid) or AWX (free) add a web UI and are not needed here.

---

## Configure the playbook

Edit these on the control node (`vi <file>`; `i` to edit, `Esc` then `:wq` to save).

### `group_vars/all/main.yml`

| Setting | Example | Notes |
|---|---|---|
| `ad_domain` | `bob.net` | lowercase |
| `ad_realm` | `BOB.NET` | same name, uppercase |
| `ad_computer_ou` | `OU=Linux,DC=bob,DC=net` | must match the OU you created (copy from step 4 above) |
| `ad_join_user` | `jsmith` (pilot) or `svc_linuxjoin` | username only — no `BOB\` or `@bob.net` |
| `ad_login_groups` | `linux-prod-users`, `linux-prod-admins` | must match AD group names exactly |
| `ad_sudo_group` | `linux-prod-admins` | |
| `ad_sudo_nopasswd` | `false` | keep `false` in production |
| `manage_ntp` | `false` | `false` keeps the existing NTP setup (e.g. RHEL pool); `true` switches to the DCs |
| `ntp_servers` | `dc01.bob.net` | only used when `manage_ntp: true` |
| `use_fully_qualified_names` | `false` | log in as `jsmith`, not `jsmith@bob.net` |
| `enable_gssapi_sso` | `true` | passwordless SSH from domain PCs |

### `inventory.ini`

Use the IP if the control node can't resolve the server name:

```ini
[linux_prod]
tester1 ansible_host=<TESTER1_IP>
tester2 ansible_host=<TESTER2_IP>
```

Mixed OSes — group them however you like; `linux_prod` must include them all:

```ini
[rhel]
tester1 ansible_host=<TESTER1_IP>
rhel7box ansible_host=<RHEL7_IP>

[ubuntu]
ubu1 ansible_host=<UBUNTU_IP> ansible_user=<your-ubuntu-user> ansible_become=true

[linux_prod:children]
rhel
ubuntu
```

Already have your own group (e.g. `[myhosts]`)? The playbook targets `linux_prod`, so add:

```ini
[linux_prod:children]
myhosts
```

### Vault (join account password)

```bash
ansible-vault create vault.yml
```

1. Make up a **vault password** (not your AD password) — you type it on every run.
2. Press `i` and type: `vault_ad_join_password: 'YourADPassword'` (keep the single quotes).
3. `Esc`, `:wq`, Enter.

`create` saves the file **already encrypted**. Check with `head -1 vault.yml` → `$ANSIBLE_VAULT;1.1;AES256`.
Later: `ansible-vault view vault.yml` / `ansible-vault edit vault.yml` (e.g. after changing your AD password).

> Never commit `vault.yml`. It is listed in `.gitignore`.

---

## Run it

```bash
# 1. Check Ansible can reach the test servers
ansible -i inventory.ini tester1,tester2 -u root -m ping

# 2. Dry run - shows what would change, changes nothing
ansible-playbook -i inventory.ini ad-join.yml -u root --ask-vault-pass --limit tester1,tester2 --check --diff

# 3. Real run, with a test user lookup
ansible-playbook -i inventory.ini ad-join.yml -u root --ask-vault-pass --limit tester1,tester2 -e ad_test_user=jsmith

# 4. Later: remaining servers in batches
ansible-playbook -i inventory.ini ad-join.yml -u root --ask-vault-pass --limit server3,server4
```

Connecting with a non-root account that uses sudo? Use `-u youradmin -K` instead of `-u root` (`-K` asks for the sudo password).

The PLAY RECAP at the end must show `failed=0` for each server. If one fails, the others stay joined; fix it and re-run with `--limit` on just that server.
The playbook is **idempotent** — safe to re-run, and it's how you add new servers later.

**Hostnames:** `hostname -f` showing just `tester1` is fine — the join works with the short name. Don't rename production servers; apps, monitoring agents, licenses and certificates may depend on the name.

### What the playbook does
1. Checks Ansible can manage each server (installs Python 3 if missing; stops with instructions on an Ansible/Python mismatch)
2. Checks the OS is supported and loads its settings from `vars/`
3. Installs `realmd`, `sssd`, `adcli` and friends with `dnf`, `yum` or `apt`
4. Keeps the existing NTP setup (or points chrony at the DCs if `manage_ntp: true`) and checks the clock is synchronised
5. Joins the domain (password from vault, hidden from logs; on failure shows realm's reason)
6. Deploys `sssd.conf` (group-based access, `passwd` → AD, cached credentials)
7. Turns on SSSD logins and auto-created home directories (`authselect` / `authconfig` / `pam-auth-update`)
8. Grants sudo to `linux-prod-admins` (validated with `visudo`)
9. Configures SSH for AD passwords and expired-password prompts
10. Verifies the join and optionally looks up a test user

---

## Test

Keep a break-glass (local/root) session open to each server while testing.

```bash
ssh jsmith@<TESTER1_IP>      # AD password; home dir created automatically
id                           # shows AD groups
sudo -l                      # admins only
passwd                       # changes the AD password (must meet AD policy)
```

Then log in to `tester2` with the new password — that proves one password works everywhere.
An AD account in **neither** group must get "Permission denied".

---

## Day-to-day use

| Task | How |
|---|---|
| Give someone access | Add them to `linux-prod-users` in AD |
| Give admin/sudo | Add them to `linux-prod-admins` |
| Apply group change now | `sudo sss_cache -u jsmith` on the server (otherwise up to ~90 min) |
| Remove access | Remove from group or disable in AD |
| Kill active sessions | `sudo loginctl terminate-user jsmith` |
| Add a new server | Add to `inventory.ini`, run playbook with `--limit newhost` |

### Things to know
- **Password policy:** `passwd` follows AD policy, including minimum password age.
- **Login name:** use the sAMAccountName (`jsmith`), not the email/UPN.
- **Cached credentials:** a server offline during a password change accepts the old password until it reconnects.
- **SSH keys bypass AD:** if users add their own `authorized_keys`, disabling them in AD won't lock them out. Disallow personal keys or manage them centrally.
- **Break-glass:** keep one local admin account with a vaulted password on every server.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `New-ADGroup`: "Directory object not found" | The OU in `-Path` doesn't exist yet — create the OU first |
| `New-ADOrganizationalUnit` not recognized | No AD module — run on a DC or install RSAT |
| `realm discover bob.net` fails | Server isn't using AD DNS — check `/etc/resolv.conf` (Ubuntu: `resolvectl status`) |
| "This Ansible ... cannot manage" | Ansible too new for the server's Python (usually RHEL 7) — see [RHEL 7 notes](#rhel-7-notes) |
| "has no Python" in a dry run | Expected — a real run installs python3 first |
| Ubuntu: `Permission denied` connecting as root | Use `ansible_user=<sudo user> ansible_become=true` and `-K` — see [Ubuntu notes](#ubuntu-notes) |
| `realm join` / "Join the domain" fails | Join account rights on the OU; `ad_join_user` and vault password; OU path |
| `no hosts matched` | `--limit` name isn't in group `linux_prod` in `inventory.ini` |
| "Permission denied" for valid user | User in an allowed group? `sss_cache -E; systemctl restart sssd` |
| Clock / Kerberos errors | Compare with a DC: `sudo chronyd -Q 'server dc01.bob.net iburst'`; must be < 5 min. If the DCs drift, set `manage_ntp: true` |
| User not found | `id jsmith`; `journalctl -u sssd`; add `debug_level = 6` to `sssd.conf` |
| Login works, `passwd` fails | Port 464 blocked, or AD password policy (length, history, min age) |
| Group change not applied | `sss_cache -u jsmith` |
| Cloud password reset doesn't work on servers | Password Writeback not enabled in Entra Connect |

More detail and an undo procedure: [RUNBOOK.md](RUNBOOK.md#part-7--if-something-fails).

---

## Ubuntu notes

- **Connect as a sudo user.** Ubuntu usually blocks root SSH logins, so set the user per host and add `-K` to the command (Ansible asks for the sudo password):

  ```ini
  [ubuntu]
  ubu1 ansible_host=<IP> ansible_user=<your-ubuntu-user> ansible_become=true
  ```

- **DNS check is different.** `/etc/resolv.conf` shows `127.0.0.53` (systemd-resolved), not your DNS servers. Check the real ones with:

  ```bash
  resolvectl status | grep -A2 "DNS Servers"
  ```

  They must be your AD DNS servers / DCs. `realm discover bob.net` is still the key test.

- **Time sync:** with `manage_ntp: false` Ubuntu keeps its default `systemd-timesyncd`. With `manage_ntp: true` the playbook installs `chrony` (which replaces timesyncd) and points it at the DCs.
- **Dry runs** may show `...ignoring` on the package step if the `python3-apt` helper isn't installed yet; the real run installs it automatically.

## RHEL 7 notes

- **End of life:** Red Hat's normal support ended June 2024. The playbook works on RHEL 7 and prints a reminder; plan an upgrade.
- **Ansible version matters.** RHEL 7 has Python 2.7, and ansible-core **2.17 and newer** can't manage Python 2.7 servers. Check on the control node:

  ```bash
  ansible --version          # first line, e.g. "ansible [core 2.14.x]"
  ```

  **2.16 or lower** (what RHEL 9's `dnf install ansible-core` gives you) works as is. If yours is newer, the playbook stops on the RHEL 7 server *before changing anything* and tells you to install an older ansible-core alongside the current one:

  ```bash
  python3 -m venv ~/ansible-legacy
  ~/ansible-legacy/bin/pip install 'ansible-core>=2.15,<2.16'
  ~/ansible-legacy/bin/ansible-playbook -i inventory.ini ad-join.yml --ask-vault-pass --limit <rhel7-server>
  ```

- **Repositories:** the server must be able to install packages (`yum repolist` shows repos), e.g. through an Extended Life Cycle Support subscription or your internal mirror.

---

## Rollout plan & timeline

| Step | Work | Time |
|---|---|---|
| 1 | AD prep: OU, groups, members; confirm firewall ports | ½ day |
| 2 | Control node with Ansible | 1–2 hours |
| 3 | Pilot on 2 test servers; test login, password change, sudo | ½–1 day |
| 4 | Create `svc_linuxjoin`; roll out to remaining servers in batches | ½ day |
| 5 | Brief the team, document | 1–2 hours |

**Total effort:** ~2–4 working days. Allow 1–2 weeks of calendar time if change approval or firewall requests are needed.

**Cost:** none — uses tools included with RHEL plus free Ansible.

**Risk:** low — piloted on test servers first, rolled out in batches, local break-glass account kept on every server.

---

## Repository layout

```
.
├── README.md
├── RUNBOOK.md                  # step-by-step pilot checklist
├── ad-join.yml                 # the playbook
├── inventory.ini               # list of servers
├── vault.yml.example           # shows the one line the vault needs
├── group_vars/
│   └── all/
│       └── main.yml            # domain, OU, groups, options
├── vars/                       # per-OS packages and tools
│   ├── RedHat.yml              # RHEL 8, 9, 10
│   ├── RedHat-7.yml            # RHEL 7
│   └── Debian.yml              # Ubuntu 22.04+
└── templates/
    ├── sssd.conf.j2            # SSSD config (all OSes)
    └── sshd-ad.conf.j2         # SSH drop-in (RHEL 9+, Ubuntu)
```
