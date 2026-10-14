# Pilot Runbook: Join test servers to AD

Step-by-step checklist for the first run on test servers (`tester1`, optionally `tester2`). Examples use the domain `bob.net`.
Work through the parts in order; each part ends with a check. Don't move on until it passes.

> Replace everything in `<angle brackets>`, and the example domain `bob.net`, with your real values (remove the brackets).
> Keep your real domain, server names, IPs and passwords **out of this repository**. Edit them only on the control node.

| Part | What | Where | Time |
|---|---|---|---|
| 1 | Confirm approvals and AD objects | Windows / ADUC | 15–30 min |
| 2 | Pre-checks on tester1 | tester1 | 15 min |
| 3 | Set up the project | Control node | 15 min |
| 4 | Connectivity check and dry run | Control node | 10 min |
| 5 | Real run on tester1 | Control node | 5 min |
| 6 | Test logins, password change, sudo | Any workstation | 15 min |
| 7 | If something fails / undo | — | — |
| 8 | After the pilot | — | — |

### Values used below

| Placeholder | Meaning | Example |
|---|---|---|
| `bob.net` | Example AD domain, lowercase | your real domain |
| `BOB.NET` | Same, uppercase (Kerberos realm) | your real domain, uppercase |
| `<TESTER1_IP>` / `<TESTER2_IP>` | Test server IPs | `10.0.0.21` |
| `<youradmin>` | Local account you SSH to tester1 with (or `root`) | `wayne` |
| `<your-ad-username>` | AD account used to join (pilot only) | `jsmith` |
| `<test-user>` | AD user in a Linux group, for testing | `jsmith` |
| `<DC-NAME>` | A domain controller's full name | `dc01.bob.net` |

---

## Part 1 – Before you start

All of these must be true before you touch a server. Most are done in **Active Directory Users and Computers (ADUC)**.

- [ ] **Approval:** your manager knows you're piloting on tester1. Raise a change request if your team uses them.
- [ ] **AD admin informed:** you've agreed where the OU and groups go and who creates them.
- [ ] **Linux OU exists (create it first).** PowerShell as admin: `New-ADOrganizationalUnit -Name "Linux" -Path "DC=bob,DC=net"` — or in ADUC right-click the domain → **New → Organizational Unit**. Path: `OU=Linux,DC=bob,DC=net`.
- [ ] **Groups exist, inside the OU.** `New-ADGroup -Name "linux-prod-users" -GroupScope Global -GroupCategory Security -Path "OU=Linux,DC=bob,DC=net"`, and the same for `linux-prod-admins`.
- [ ] **A test user is in a group.** `Add-ADGroupMember -Identity "linux-prod-admins" -Members <test-user>`. Ideally a second account in `linux-prod-users` only. Wait a minute or two for it to reach all DCs.
- [ ] **Your AD account can join computers:** Domain Admin, or "Create computer objects" delegated on the Linux OU. Local/Linux admin rights are not enough.
- [ ] **DNS for tester1:** A record exists; PTR (reverse) record if possible.
- [ ] **Break-glass access to tester1:** a local account or VM console that doesn't depend on AD. Keep a session open all the time.
- [ ] **Snapshot** tester1 if it's a VM.

---

## Part 2 – Pre-checks on tester1

Log in to tester1 with your local account. If a check fails, fix it before Part 3.

**1. OS version** – must be RHEL 8, 9 or 10.

```bash
cat /etc/redhat-release
```

**2. Hostname**

```bash
hostname -f
```

Seeing just `tester1` (not `tester1.bob.net`) is fine. **Don't change hostnames on production servers**: apps, monitoring agents, licenses and certificates may depend on them. The join works with the short name.

**3. DNS points at AD**

```bash
cat /etc/resolv.conf
```

Expect something like:

```
search bob.net
nameserver 10.0.0.5
nameserver 10.0.0.6
```

Good: the `nameserver` IPs are your DCs / AD DNS servers (`ipconfig /all` on a domain PC shows them).
Problem: only public DNS (8.8.8.8, 1.1.1.1) or the router. Fix with `nmcli` on the connection; NetworkManager overwrites manual edits.

**4. The domain is discoverable** – the most important check.

```bash
sudo dnf install -y realmd
realm discover bob.net
```

Pass: shows `type: kerberos`, `server-software: active-directory`, `configured: no`.
Fail: almost always DNS (step 3) or a firewall.

**5. Find the domain controllers**

```bash
host -t SRV _ldap._tcp.bob.net
```

Output looks like `_ldap._tcp.bob.net has SRV record 0 100 389 dc01.bob.net.`
The last word on each line is a DC; use one for step 6 (without the trailing dot).
`host: command not found` → `sudo dnf install -y bind-utils`. `NXDOMAIN` → DNS problem (step 3).

**6. Ports to a DC are open.** Set the DC name, then paste the whole block at once (the `for` loop is one command):

```bash
DC=<DC-NAME>
for p in 53 88 389 464 636 3268; do
  timeout 3 bash -c "</dev/tcp/$DC/$p" 2>/dev/null && echo "$p open" || echo "$p BLOCKED"
done
```

Pass: six lines, all `open`. Any `BLOCKED` needs a firewall request (464 blocked = password changes fail).

**7. Clock agrees with AD** (must be within 5 minutes; ideally seconds)

```bash
chronyc tracking | grep -E "Leap status|System time"     # Leap status : Normal = synced
sudo chronyd -Q 'server <DC-NAME> iburst'               # prints "System clock wrong by X seconds"
```

`chronyd -Q` only asks the DC once; it doesn't change the clock or your NTP setup, and runs alongside the normal chronyd.
If it hangs (DC not answering UDP 123), press Ctrl+C and compare `date -u` with `(Get-Date).ToUniversalTime()` in PowerShell on a DC.

---

## Part 3 – Set up the project on the control node

Run as your normal user on the **control node**.

**1. Get the files** (choose one)

```bash
# A) Public repository – no GitHub login needed
cd ~
curl -L -o linux-ad-join.zip https://github.com/wayneblay/linux-ad-join/archive/refs/heads/main.zip
unzip linux-ad-join.zip
cd ~/linux-ad-join-main

# B) Setup script copied to the control node
cd ~ && bash setup-linux-ad-join.sh && cd ~/linux-ad-join
```

**2. Ansible installed?**

```bash
ansible --version
sudo dnf install -y ansible-core      # only if missing
```

**3. Inventory** – test servers by IP (delete the tester2 line to test only one):

```bash
vi inventory.ini
```

```ini
[linux_prod]
tester1 ansible_host=<TESTER1_IP>
tester2 ansible_host=<TESTER2_IP>
```

With two servers, use `--limit tester1,tester2` instead of `--limit tester1` in Parts 4 and 5, and do Parts 2 and 6 on **each** server.

Already have your own group (e.g. `[myhosts]` with two test servers)? The playbook targets the group `linux_prod`, so add this to `inventory.ini` and use `--limit myhosts`:

```ini
[linux_prod:children]
myhosts
```

**4. Domain settings**

```bash
vi group_vars/all/main.yml
```

(`i` to edit, `Esc` then `:wq` to save)

```yaml
ad_domain: bob.net
ad_realm: BOB.NET
ad_computer_ou: "OU=Linux,DC=bob,DC=net"   # your real OU path
ad_join_user: <your-ad-username>                       # username only, no DOMAIN\ or @domain
manage_ntp: false                                      # keep existing NTP (e.g. RHEL pool)
```

Leave `ad_join_password` as it is. Change `ad_login_groups` / `ad_sudo_group` only if your AD groups have different names.

**5. Store the join password, encrypted**

```bash
ansible-vault create vault.yml
```

1. Make up a **vault password** (not your AD password); you'll type it on every run.
2. Press `i` and type: `vault_ad_join_password: 'YourADPassword'` (keep the single quotes).
3. `Esc`, `:wq`, Enter.

`create` saves the file **already encrypted**, so there's nothing more to do. Check:

```bash
head -1 vault.yml     # must show $ANSIBLE_VAULT;1.1;AES256
```

(Only if you made a plain-text `vault.yml` with `cp`/`vi` do you need `ansible-vault encrypt vault.yml`.)

---

## Part 4 – Connectivity check and dry run

From the project folder on the control node.

> Connecting as **root**? Use `-u root` and drop `-K` in every command below.

**1. Reach tester1**

```bash
ansible -i inventory.ini tester1 -u <youradmin> -m ping
```

Pass: `"ping": "pong"`.

**2. Get root on tester1**

```bash
ansible -i inventory.ini tester1 -u <youradmin> -b -K -m command -a "whoami"
```

Pass: `root`.

**3. Dry run** – shows what would change, changes nothing.

```bash
ansible-playbook -i inventory.ini ad-join.yml -u <youradmin> -K --ask-vault-pass --limit tester1 --check --diff
```

What it means: run `ad-join.yml` on the servers in `inventory.ini`, log in as you, ask for your sudo password (`-K`) and vault password, only on tester1 (`--limit`), and only *show* the changes (`--check --diff`).

Expect: packages and config files as `changed`, the join step as `skipping`. Any red `fatal` → see Part 7.

---

## Part 5 – Real run on tester1

Keep your break-glass session open.

```bash
ansible-playbook -i inventory.ini ad-join.yml -u <youradmin> -K --ask-vault-pass --limit tester1 -e ad_test_user=<test-user>
```

Success (1–3 minutes):

- [ ] "Check the clock is synchronised" → `ok`
- [ ] "Join the domain" → `changed`
- [ ] "Verify the join" → `ok`
- [ ] "Show test user lookup" prints `uid=...(<test-user>) ... groups=...linux-prod-...`
- [ ] PLAY RECAP for tester1 shows `failed=0`

If it stops with `fatal`, local logins keep working; fix the problem and run the same command again (it's safe to re-run).

---

## Part 6 – Test logins, password change and sudo

From a **new** terminal (break-glass session still open):

```bash
ssh <test-user>@<TESTER1_IP>     # AD password; home folder created on first login
id                               # lists linux-prod-admins or linux-prod-users
pwd                              # /home/<test-user>
sudo -l                          # admins: (ALL) ALL after AD password; users: refused
```

- [ ] An AD account in **neither** group gets "Permission denied", even with the right password.
- [ ] Optional: `passwd` changes the AD password (must meet AD policy); log back in with the new one, and check it on a Windows sign-in.
- [ ] Write down what passed and failed. That's your evidence for the manager and AD admin.

---

## Part 7 – If something fails

| Where it failed / error text | Likely cause | Fix |
|---|---|---|
| `ping` fails: `UNREACHABLE` | SSH from control node | `ssh <youradmin>@<TESTER1_IP>` by hand; check IP in `inventory.ini` |
| `Missing sudo password` / `Incorrect sudo password` | `-K` password | Re-run with `-K`; your sudo password on tester1 |
| `Decryption failed` / `no vault secrets found` | Vault password | Re-run with `--ask-vault-pass`; `ansible-vault view vault.yml` |
| `no hosts matched` | `--limit` name not in group `linux_prod` | Check `inventory.ini`; see Part 3 step 3 |
| "Check the clock is synchronised" | chrony not synced | `chronyc sources`; NTP reachable? Or `manage_ntp: true` + DC names |
| "Confirm the domain is discoverable" / `No such realm found` | DNS | Fix resolv.conf (Part 2 step 3) |
| "Join the domain": `Insufficient permissions` | Account can't create computers in the OU | Domain Admin, or delegate rights on the Linux OU |
| "Join the domain": `Couldn't authenticate` / `Preauthentication failed` | Wrong user or password | `ad_join_user` (username only); `ansible-vault edit vault.yml` |
| "Join the domain": `No such object` | OU path wrong | Copy `distinguishedName` from the OU's Attribute Editor tab in ADUC |
| `Clock skew too great` | Clock > 5 min off | Fix time sync and re-run |
| SSH `Permission denied` for a group member | Cache or name mismatch | `sudo sss_cache -E; sudo systemctl restart sssd`; group names in `main.yml` must match AD exactly |
| `id <test-user>`: `no such user` | SSSD can't reach AD | `sudo journalctl -u sssd -n 50`; ports 389/3268 |
| Login works, `passwd` fails | Port 464 or AD password policy | Check port 464; password length/history/minimum age |

### Undo on tester1

Revert the VM snapshot if you have one. Otherwise, from the break-glass session:

```bash
sudo realm leave bob.net
sudo rm -f /etc/sudoers.d/ad-admins
sudo rm -f /etc/ssh/sshd_config.d/40-ad-auth.conf     # RHEL 9+
sudo systemctl reload sshd
```

Then delete the `tester1` computer object from the Linux OU in ADUC. Local accounts are unaffected.

---

## Part 8 – After the pilot

- [ ] Share the Part 6 results with your manager and the AD admin.
- [ ] Leave tester1 running a day or two; have a teammate log in.
- [ ] Create a dedicated join account (`svc_linuxjoin`, rights only on the Linux OU — see README), then update `ad_join_user` and the vault so your personal password isn't stored.
- [ ] Confirm Password Writeback is enabled in Entra Connect (hybrid AD + Entra).
- [ ] Make sure every production server has a working local break-glass account.
- [ ] Roll out 2–3 servers at a time with `--limit`, testing logins after each batch.
- [ ] Tell the team how to log in, and that `passwd` now changes their AD password.
