# Linux Server Login with Active Directory (RHEL + SSSD)

Lets team members log in to our RHEL production servers with their **own Active Directory username and password**.
They can change their password on any server (or in Windows), and the change works on every server.
AD is the single source of truth — nothing is copied or synced to the servers.

**First time?** Follow [RUNBOOK.md](RUNBOOK.md) — a step-by-step checklist for piloting on one test server.

---

## Contents
1. [How it works](#how-it-works)
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
| RHEL 8, 9 or 10 (or Rocky/Alma) | Playbook checks this |
| On-prem Active Directory (AD DS) | Same AD the Windows PCs use |
| DNS | Servers must use AD DNS (or DNS that forwards to it) |
| Time sync | Clock must be within 5 minutes of the DCs. RHEL NTP pool is fine if the DCs are also accurate |
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
realm discover ad.yourcompany.com
```

You should see `type: kerberos` and `server-software: active-directory`.

---

## Active Directory setup

Nothing is installed on the domain controllers and no schema changes are needed.
SSSD generates consistent Linux UIDs/GIDs from each user's AD SID.

### 1. Create the security groups

```powershell
New-ADGroup -Name "linux-prod-users"  -GroupScope Global -GroupCategory Security -Path "OU=Groups,DC=corp,DC=example,DC=com"
New-ADGroup -Name "linux-prod-admins" -GroupScope Global -GroupCategory Security -Path "OU=Groups,DC=corp,DC=example,DC=com"

Add-ADGroupMember -Identity "linux-prod-users"  -Members jsmith, mjones
Add-ADGroupMember -Identity "linux-prod-admins" -Members jsmith
```

Avoid spaces in group names.

### 2. Create an OU for the Linux servers

```powershell
New-ADOrganizationalUnit -Name "Linux" -Path "OU=Servers,DC=corp,DC=example,DC=com"
```

### 3. Create the join service account

```powershell
New-ADUser -Name "svc_linuxjoin" -SamAccountName "svc_linuxjoin" `
  -Path "OU=Service Accounts,DC=corp,DC=example,DC=com" `
  -AccountPassword (Read-Host -AsSecureString "Password") `
  -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true
```

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

### 4. DNS
AD's SRV records already exist. Make sure each Linux server has an **A record** and ideally a **PTR record**.

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
| 123 | UDP | Time sync |
| 389 | TCP/UDP | LDAP — user and group lookups |
| 464 | TCP/UDP | Kerberos password change (`passwd`) |
| 636 | TCP | LDAPS |
| 3268 | TCP | Global Catalog |

What breaks if blocked:
- **88** blocked → nobody can log in
- **464** blocked → logins work, password changes fail (most commonly missed)
- **53 / 389** blocked → server can't join the domain

### Test from a Linux server

```bash
DC=dc01.corp.example.com
for p in 53 88 389 464 636 3268; do
  timeout 3 bash -c "</dev/tcp/$DC/$p" 2>/dev/null && echo "$p open" || echo "$p BLOCKED"
done
```

All `open` → no firewall request needed. Any `BLOCKED` → request those ports (TCP **and** UDP for 53, 88, 123, 464) from all servers to the DCs.
RHEL's own `firewalld` allows outbound traffic by default, so no change is needed on the servers.

---

## Control node (where Ansible runs)

Any Linux machine that can SSH to the servers: an admin/jump box, a small RHEL VM (1 vCPU / 2 GB RAM), or a Linux/macOS workstation (Windows → use WSL).
Nothing needs to be installed on the target servers.

```bash
# 1. Install Ansible (free, from RHEL AppStream)
sudo dnf install -y ansible-core

# 2. SSH key access to every server (account with sudo)
ssh-keygen -t ed25519
for h in prod-app01 prod-app02 prod-db01; do ssh-copy-id youradmin@$h.corp.example.com; done

# 3. Get this repo and test connectivity
git clone https://github.com/wayneblay/linux-ad-join.git && cd linux-ad-join
ansible -i inventory.ini linux_prod -u youradmin -m ping
```

- Keep the Ansible connection account **local** (not AD) so you can still manage servers if AD is down. It doubles as break-glass access.
- The control node holds keys to every production server — lock it down.

**Licensing:** `ansible-core` is free and open source. Red Hat Ansible Automation Platform (paid) or AWX (free) add a web UI and are not needed here.

---

## Configure the playbook

### `group_vars/all/main.yml`

| Setting | Example | Notes |
|---|---|---|
| `ad_domain` | `ad.acme.com` | lowercase |
| `ad_realm` | `AD.ACME.COM` | same name, uppercase |
| `ad_computer_ou` | `OU=Linux,OU=Servers,DC=ad,DC=acme,DC=com` | each domain part becomes a `DC=` |
| `ad_join_user` | `svc_linuxjoin` | |
| `ad_login_groups` | `linux-prod-users`, `linux-prod-admins` | |
| `ad_sudo_group` | `linux-prod-admins` | |
| `ad_sudo_nopasswd` | `false` | keep `false` in production |
| `manage_ntp` | `false` | `false` keeps the existing NTP setup (e.g. RHEL pool); `true` switches to the DCs |
| `ntp_servers` | your DC hostnames | only used when `manage_ntp: true` |
| `use_fully_qualified_names` | `false` | log in as `jsmith` not `jsmith@domain` |
| `enable_gssapi_sso` | `true` | passwordless SSH from domain PCs |

### `inventory.ini`
List all servers under `[linux_prod]`.

### Vault (join account password)

```bash
cp vault.yml.example vault.yml     # put the svc_linuxjoin password in it
ansible-vault encrypt vault.yml
```

> Never commit an **unencrypted** `vault.yml`. It is listed in `.gitignore`; remove it from there only if you want to commit the encrypted file.

---

## Run it

```bash
# 1. Dry run on one non-critical host
ansible-playbook -i inventory.ini ad-join.yml -u youradmin -K --ask-vault-pass \
  --limit prod-app01.corp.example.com --check --diff

# 2. Real run on that host, with a test user lookup
ansible-playbook -i inventory.ini ad-join.yml -u youradmin -K --ask-vault-pass \
  --limit prod-app01.corp.example.com -e ad_test_user=jsmith

# 3. Everything (or in batches with --limit)
ansible-playbook -i inventory.ini ad-join.yml -u youradmin -K --ask-vault-pass
```

`-K` asks for the sudo password on the servers (omit if passwordless sudo).
The playbook is **idempotent** — safe to re-run, and it's how you add new servers later.

### What the playbook does
1. Checks the OS is RHEL 8/9/10 family
2. Installs `realmd`, `sssd`, `adcli`, `oddjob-mkhomedir`, `krb5-workstation`, `chrony`, etc.
3. Keeps the existing NTP setup (or points chrony at the DCs if `manage_ntp: true`) and checks the clock is synchronised
4. Joins the domain (password from vault, hidden from logs)
5. Deploys `sssd.conf` (group-based access, `passwd` → AD, cached credentials)
6. Enables auto-created home directories (`authselect sssd with-mkhomedir`)
7. Grants sudo to `linux-prod-admins` (validated with `visudo`)
8. Configures SSH for AD passwords and expired-password prompts (RHEL 8 and 9+ layouts)
9. Verifies the join and optionally looks up a test user

---

## Test

Keep your existing root/SSH session open while testing.

```bash
ssh jsmith@prod-app01        # AD password; home dir created automatically
id                           # shows AD groups
sudo -l                      # admins only
passwd                       # changes the AD password (must meet AD policy)
```

Then log in to a second server with the new password.

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
| `realm join` fails | `realm discover <domain>`; DNS; join account rights on the OU |
| "Permission denied" for valid user | User in an allowed group? `sss_cache -E; systemctl restart sssd` |
| Clock / Kerberos errors | Compare `date -u` on the server with a DC (`w32tm /query /status`); must be < 5 min apart. If the DCs drift, set `manage_ntp: true` |
| User not found | `id jsmith`; `journalctl -u sssd`; add `debug_level = 6` to `sssd.conf` |
| Login works, `passwd` fails | Port 464 blocked, or AD password policy (length, history, min age) |
| Group change not applied | `sss_cache -u jsmith` |
| Cloud password reset doesn't work on servers | Password Writeback not enabled in Entra Connect |

---

## Rollout plan & timeline

| Step | Work | Time |
|---|---|---|
| 1 | AD prep: groups, OU, join account; confirm firewall ports | ½ day |
| 2 | Control node with Ansible | 1–2 hours |
| 3 | Pilot on 1–2 non-critical servers; test login, password change, sudo | ½–1 day |
| 4 | Roll out to remaining servers in batches | ½ day |
| 5 | Brief the team, document | 1–2 hours |

**Total effort:** ~2–4 working days. Allow 1–2 weeks of calendar time if change approval or firewall requests are needed.

**Cost:** none — uses tools included with RHEL plus free Ansible.

**Risk:** low — piloted on one server first, rolled out in batches, local break-glass account kept on every server.

---

## Repository layout

```
.
├── README.md
├── RUNBOOK.md                  # step-by-step pilot checklist
├── ad-join.yml                 # the playbook
├── inventory.ini               # list of servers
├── vault.yml.example           # template for the encrypted join password
├── group_vars/
│   └── all/
│       └── main.yml            # domain, OU, groups, DCs, options
└── templates/
    ├── sssd.conf.j2            # SSSD config
    └── sshd-ad.conf.j2         # SSH drop-in for RHEL 9+
```
