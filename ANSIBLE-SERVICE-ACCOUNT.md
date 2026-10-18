# Ansible Service Account Setup

Run Ansible through a dedicated, key-only service account instead of SSHing in as `root`.
This works alongside the AD join: `ansible-srv` is a local account, so the AD group rules don't affect it.

```
[ CONTROL NODE ]                                   [ EVERY SERVER ]
User: ansible-mgr  (one per admin, no sudo)        User: ansible-srv  (local, password locked)
Holds: private key, protected by a passphrase ===> Holds: public key, accepted only from the control node IP
Logs:  every playbook run (~/ansible.log)          Sudo:  passwordless, file validated with visudo
```

| | Before | After |
|---|---|---|
| How Ansible connects | `-u root` (Ubuntu: your user + `-K`) | `ansible-srv` with a key, set once in the inventory |
| Root SSH logins needed | Yes | No — can be turned off once this works |
| Who ran a playbook | Unknown | Each admin's own `ansible-mgr`-style login + Ansible log |

---

## Phase 1 — Control node (RHEL 9)

### 1. Install Ansible (as your normal admin account)

```bash
sudo dnf install -y ansible-core        # from RHEL AppStream — no EPEL needed
ansible --version
```

Ubuntu control node: `sudo apt update && sudo apt install -y ansible-core`.

### 2. Create the manager account — no sudo

`ansible-mgr` only runs playbooks; it doesn't need root on the control node. Don't add it to `wheel` / `sudo`.

```bash
sudo useradd -m -s /bin/bash ansible-mgr
sudo passwd ansible-mgr
```

> For accountability, give **each admin their own** account (e.g. `wayne-ansible`) rather than sharing one.

### 3. Create the key — with a passphrase

That key will have passwordless root on every server, so protect it:

```bash
sudo -iu ansible-mgr
ssh-keygen -t ed25519 -C "ansible-mgr@control-node"     # set a passphrase when asked
ls -l ~/.ssh                                            # id_ed25519 must be -rw------- (600)
```

At the start of each work session, unlock it once:

```bash
eval "$(ssh-agent)"
ssh-add ~/.ssh/id_ed25519      # type the passphrase once
```

> Only for unattended jobs (cron/CI) consider a key without a passphrase — and then the `from=` restriction below is essential.

### 4. Turn on Ansible's own log

The servers only see `ansible-srv` running Python, so the useful record is on the control node:

```bash
cat > ~/.ansible.cfg <<'EOF'
[defaults]
log_path = ~/ansible.log
EOF
```

Passwords stay out of the log: the join step uses `no_log`.

### 5. Copy the project into this account

```bash
cd ~
curl -L -o linux-ad-join.zip https://github.com/wayneblay/linux-ad-join/archive/refs/heads/main.zip
unzip -q linux-ad-join.zip && cd linux-ad-join-main
```

Copy your existing `inventory.ini`, `group_vars/all/main.yml` and `vault.yml` into it.

---

## Phase 2 — Create `ansible-srv` on every server

### Option A — the playbook (recommended)

Run it from your **current** admin account — the one that already reaches the servers as root, the way you ran `ad-join.yml` — pointing it at `ansible-mgr`'s public key:

```bash
# make ansible-mgr's public key readable to your account (it's public, so this is safe)
sudo cp /home/ansible-mgr/.ssh/id_ed25519.pub /tmp/ansible-mgr.pub && sudo chmod 644 /tmp/ansible-mgr.pub

cd ~/linux-ad-join-main          # your current working folder
ansible-playbook -i inventory.ini create-service-account.yml -u root \
  -e svc_pubkey_file=/tmp/ansible-mgr.pub -e control_node_ip=<CONTROL_NODE_IP> --limit tester1
```

- Test on one server first (`--limit tester1`), then drop `--limit` for the rest.
- Ubuntu servers that connect as your sudo user: add `-K`.
- Find the control node's IP with `hostname -I`.

It creates `ansible-srv` with a locked password, installs the key (accepted only from the control node IP), fixes SELinux labels on RHEL, and adds a sudoers file that's validated with `visudo` before it's saved. Safe to re-run.

### Option B — by hand, on each server

```bash
sudo useradd -m -s /bin/bash ansible-srv
sudo passwd -l ansible-srv                                  # key login only

sudo mkdir -p /home/ansible-srv/.ssh
echo 'from="<CONTROL_NODE_IP>" <PASTE ansible-mgr PUBLIC KEY>' | sudo tee /home/ansible-srv/.ssh/authorized_keys
sudo chown -R ansible-srv:ansible-srv /home/ansible-srv/.ssh
sudo chmod 700 /home/ansible-srv/.ssh
sudo chmod 600 /home/ansible-srv/.ssh/authorized_keys
sudo restorecon -Rv /home/ansible-srv/.ssh                  # RHEL: fixes SELinux labels

echo 'ansible-srv ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible-srv
sudo chmod 440 /etc/sudoers.d/ansible-srv
sudo visudo -cf /etc/sudoers.d/ansible-srv                  # must say "parsed OK"
```

Get the public key with `cat ~/.ssh/id_ed25519.pub` (as `ansible-mgr` on the control node).

---

## Phase 3 — Test, then switch over

### 1. Test

As `ansible-mgr` on the control node:

```bash
ssh ansible-srv@<SERVER_IP> 'sudo whoami'                          # prints: root  (no password asked)
ansible -i inventory.ini linux_prod -u ansible-srv -b -m command -a whoami
```

Every server should answer `root`.

### 2. Put it in the inventory

Add this to `inventory.ini`, so no `-u` or `-K` is needed any more:

```ini
[linux_prod:vars]
ansible_user=ansible-srv
ansible_become=true
```

On Ubuntu hosts, remove the old per-host `ansible_user=<your-ubuntu-user> ansible_become=true`.

### 3. Run playbooks without `-u root` / `-K`

```bash
ansible-playbook -i inventory.ini ad-join.yml --ask-vault-pass --limit tester1 --check --diff
ansible-playbook -i inventory.ini ad-join.yml --ask-vault-pass --limit tester1
```

### 4. Optional — turn off root SSH logins

Only after Phase 3 works on a server **and** you have break-glass access (local admin account or VM console):

```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sshd -t && sudo systemctl reload sshd        # Ubuntu: reload ssh
```

Keep your existing session open while testing a new login.

---

## Day to day

| Task | How |
|---|---|
| Start a session | `eval "$(ssh-agent)"; ssh-add` |
| See what was run | `less ~/ansible.log` |
| Add a new server | Add it to `inventory.ini`, run `create-service-account.yml` once with your admin account (`-u root -e svc_pubkey_file=/tmp/ansible-mgr.pub --limit <server>`), then `ad-join.yml` |
| Rotate the key | `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new` → re-run `create-service-account.yml -e svc_pubkey_file=~/.ssh/id_ed25519_new.pub` (replaces the old key) → test with the new key → delete the old key files |
| Someone leaves | Remove their control-node account; rotate the key if they could read it |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong key, `from=` IP doesn't match, or bad permissions | Check `hostname -I` on the control node matches `from=`; check `.ssh` is 700 and `authorized_keys` 600 |
| Works on Ubuntu, fails on RHEL | SELinux labels | `sudo restorecon -Rv /home/ansible-srv/.ssh` |
| Asked for a password on `sudo` | Sudoers file missing or invalid | `sudo visudo -cf /etc/sudoers.d/ansible-srv` |
| Asked for the key passphrase every time | ssh-agent not running | `eval "$(ssh-agent)"; ssh-add` |
| `ssh -v` shows "key not accepted" from a jump host | `from=` blocks other IPs | Run Ansible from the control node only, or add that IP: `from="IP1,IP2"` |
