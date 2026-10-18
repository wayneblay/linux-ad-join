# Ansible Service Account Setup

Run Ansible through a dedicated, key-only service account (`ansible-srv`) on every server, instead of SSHing in as `root`.
This works alongside the AD join: `ansible-srv` is a local account, so the AD group rules don't affect it.

```
[ CONTROL NODE ]                                   [ EVERY SERVER ]
User: your own admin account                       User: ansible-srv  (local, password locked)
Holds: private key, protected by a passphrase ===> Holds: your public key, accepted only from the control node IP
Logs:  every playbook run (~/ansible.log)          Sudo:  passwordless, file validated with visudo
```

| | Before | After |
|---|---|---|
| How Ansible connects | `-u root` (Ubuntu: your user + `-K`) | `ansible-srv` with your key, set once in the inventory |
| Root SSH logins needed | Yes | No — can be turned off once this works |
| Record of what ran | None | Ansible log on the control node |

> **One admin running Ansible?** Use your own account on the control node, as below.
> **Several admins?** Give each their own control-node account — see [More than one admin](#more-than-one-admin).

---

## Phase 1 — Control node (as your own account)

### 1. Ansible is installed

```bash
ansible --version                        # RHEL 9: sudo dnf install -y ansible-core if missing (no EPEL needed)
```

### 2. Your SSH key — with a passphrase

That key will have passwordless root on every server, so protect it. Check whether you already have one:

```bash
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

- **No key yet:** create one and set a passphrase when asked:
  ```bash
  ssh-keygen -t ed25519 -C "ansible@control-node"
  ```
- **Key exists without a passphrase** (e.g. the one you use for root today): add one — the key itself doesn't change, so nothing on the servers breaks:
  ```bash
  ssh-keygen -p -f ~/.ssh/id_ed25519
  ```

The private key `~/.ssh/id_ed25519` must be `-rw-------` (600). It never leaves the control node — only the `.pub` file is copied to the servers.

At the start of each work session, unlock it once:

```bash
eval "$(ssh-agent)"
ssh-add ~/.ssh/id_ed25519      # type the passphrase once
```

> Only for unattended jobs (cron/CI) consider a key without a passphrase — and then the `from=` restriction below is essential.

### 3. Turn on Ansible's own log

The servers only see `ansible-srv` running Python, so the useful record of what ran is on the control node:

```bash
cat > ~/.ansible.cfg <<'EOF'
[defaults]
log_path = ~/ansible.log
EOF
```

Passwords stay out of the log: the join step uses `no_log`.

---

## Phase 2 — Create `ansible-srv` on every server

### Option A — the playbook (recommended)

From your project folder on the control node, using your **current** access (root, or your sudo user on Ubuntu) — the same way you ran `ad-join.yml`:

```bash
cd ~/linux-ad-join-main          # your working folder
hostname -I                      # note the control node's IP

# one server first
ansible-playbook -i inventory.ini create-service-account.yml -u root \
  -e control_node_ip=<CONTROL_NODE_IP> --limit tester1

# then every server in the inventory (add -K if Ubuntu servers are included)
ansible-playbook -i inventory.ini create-service-account.yml -u root \
  -e control_node_ip=<CONTROL_NODE_IP>
```

By default it installs **your** public key, `~/.ssh/id_ed25519.pub`. Using a different key file? Add `-e svc_pubkey_file=/path/to/key.pub`.

In a mixed inventory, Ubuntu servers keep their own `ansible_user=...` from the inventory (it overrides `-u root`), so one command covers all; `-K` asks for the sudo password once.

On each server it:

1. creates `ansible-srv` with a locked password (key login only)
2. creates its `.ssh` folder (700)
3. installs your **public** key (600), accepted only from the control node IP (`from=`)
4. fixes SELinux labels on RHEL
5. adds `/etc/sudoers.d/ansible-srv`, checked with `visudo` before it's saved

Safe to re-run. Each run **replaces** `ansible-srv`'s key file, which is also how you rotate the key.

### Option B — by hand, on each server

```bash
sudo useradd -m -s /bin/bash ansible-srv
sudo passwd -l ansible-srv                                  # key login only

sudo mkdir -p /home/ansible-srv/.ssh
echo 'from="<CONTROL_NODE_IP>" <PASTE YOUR PUBLIC KEY>' | sudo tee /home/ansible-srv/.ssh/authorized_keys
sudo chown -R ansible-srv:ansible-srv /home/ansible-srv/.ssh
sudo chmod 700 /home/ansible-srv/.ssh
sudo chmod 600 /home/ansible-srv/.ssh/authorized_keys
sudo restorecon -Rv /home/ansible-srv/.ssh                  # RHEL: fixes SELinux labels

echo 'ansible-srv ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/ansible-srv
sudo chmod 440 /etc/sudoers.d/ansible-srv
sudo visudo -cf /etc/sudoers.d/ansible-srv                  # must say "parsed OK"
```

Get your public key with `cat ~/.ssh/id_ed25519.pub` on the control node.

---

## Phase 3 — Test, then switch over

### 1. Test

From your account on the control node:

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

Keep your existing session open while testing a new login. After this, `create-service-account.yml` for **new** servers is run once with `-u root` before root login is turned off on them.

---

## Day to day

| Task | How |
|---|---|
| Start a session | `eval "$(ssh-agent)"; ssh-add` |
| See what was run | `less ~/ansible.log` |
| Add a new server | Add it to `inventory.ini`, run `create-service-account.yml -u root -e control_node_ip=<IP> --limit <server>` once, then `ad-join.yml` |
| Rotate the key | `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new` → re-run `create-service-account.yml -e svc_pubkey_file=~/.ssh/id_ed25519_new.pub -e control_node_ip=<IP>` (replaces the old key) → test with the new key → make it your default key |

## More than one admin

When others start running Ansible, give each person their own control-node account so the log shows who ran what — no sudo needed on it:

```bash
sudo useradd -m -s /bin/bash <name>-ansible
sudo passwd <name>-ansible
# as that user: ssh-keygen -t ed25519, and set up ~/.ansible.cfg as in Phase 1
```

Then install their public key on the servers. `create-service-account.yml` installs **one** key and replaces the file, so with several admins either list all keys in one file and pass it with `-e svc_pubkey_file=...`, or add each extra key as a new line in `/home/ansible-srv/.ssh/authorized_keys` (with the same `from="..."` prefix).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied (publickey)` | Wrong key, `from=` IP doesn't match, or bad permissions | Check `hostname -I` on the control node matches `from=`; check `.ssh` is 700 and `authorized_keys` 600 |
| Works on Ubuntu, fails on RHEL | SELinux labels | `sudo restorecon -Rv /home/ansible-srv/.ssh` |
| Asked for a password on `sudo` | Sudoers file missing or invalid | `sudo visudo -cf /etc/sudoers.d/ansible-srv` |
| Asked for the key passphrase every time | ssh-agent not running | `eval "$(ssh-agent)"; ssh-add` |
| "Could not find or access ... id_ed25519.pub" | No key yet, or a different file name | Create one (Phase 1 step 2) or pass `-e svc_pubkey_file=...` |
| Key refused from another machine | `from=` allows only the control node | Run Ansible from the control node, or allow more IPs: `from="IP1,IP2"` |
