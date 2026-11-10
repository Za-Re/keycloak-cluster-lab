# keycloak-Lab

Implementation of a Keycloak cluster on local VMs (Vagrant + VirtualBox), configured with Ansible.

```
browser --HTTPS 443--> HAProxy (lb1) --HTTPS 8443--> kc1..kcN --PostgreSQL/TLS--> db1
```

| Host | IP | RAM |
|------|----|-----|
| lb1  | 192.168.56.10 | 1 GB |
| db1  | 192.168.56.11 | 1 GB |
| kcN  | 192.168.56.2N | 2 GB |

Vagrant only creates the machines. Ansible runs from the laptop against the static
`ansible/inventory/hosts.ini`, so the same playbooks work against actual servers.

```
vagrant/    lab VMs
ansible/    VMs config
```

Building it yourself step by step? Follow [docs/build-guide.md](docs/build-guide.md).

Run `vagrant` commands from `vagrant/` and `ansible*` commands from `ansible/`
(Ansible only reads `ansible.cfg` from the current directory).

## How to Run

### 1. Create dedicated SSH key for the lab

```bash
ssh-keygen -t ed25519 -f ~/.ssh/keycloak-lab_ed25519 -C keycloak-lab -N ""
```

A separate key lives in `~/.ssh`, outside the repo. No passphrase so Ansible can use it without an agent. This is acceptable for throwaway lab VMs, not for real servers.

On first boot, `vagrant up` runs a small script inside each VM that adds this
public key to the `vagrant` user's `authorized_keys`. Ansible then logs in with
the matching private key (set in `ansible.cfg`).

### 2. Install Ansible Collections

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

### 3. Create Secrets (ansible-vault)

```bash
# in ansible/
( umask 077; openssl rand -base64 32 > .vault_pass )
chmod 600 .vault_pass
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# set every CHANGE_ME in group_vars/all/vault.yml (openssl rand -hex 24)
ansible-vault encrypt group_vars/all/vault.yml          # gitignored: everyone creates their own
```

Edit later with `ansible-vault edit group_vars/all/vault.yml`.

### 4. VMs

```bash
cd vagrant
vagrant up            # create/start all VMs (KC_NODES=5 vagrant up for more kc nodes)
vagrant halt          # stop them, keep disks
vagrant destroy -f    # delete them; then: rm -f ../ansible/.known_hosts
```

More kc nodes: `KC_NODES=N` **and** add the hosts to `ansible/inventory/hosts.ini`
(IP pattern `192.168.56.2N`).

`rm .known_hosts` after `vagrant destroy`

### 5. Laptop name resolution

Add to `/etc/hosts` on the laptop:

```
192.168.56.10 auth.lab.test
```

### 6. Run

```bash
cd ansible
ansible all -m ping
ansible-playbook site.yml
```

## PKI

`roles/pki` builds a local CA on the laptop in `ansible/pki/` (gitignored, dir 0700, keys 0600)
and issues an ECDSA P-256 cert per host with SANs `DNS:<host>` + `IP:<ip>`
(lb1 also gets `DNS:auth.lab.test`). Each host receives only:

| File on host | Mode |
|---|---|
| `/etc/ssl/lab/ca.crt` | 0644 |
| `/etc/ssl/lab/server.crt` | 0644 |
| `/etc/ssl/lab/server.key` | 0640 root:ssl-cert (service users join `ssl-cert`) |

The CA private key never leaves the laptop.

## Database (db1)

PostgreSQL 18 from the official PostgreSQL apt repository (key fingerprint verified).
Listens on 192.168.56.11 only, TLS on (lab cert). `pg_hba.conf` allows only
`hostssl keycloak keycloak <kc-IP>/32 scram-sha-256`; everything else is rejected.
ufw opens 5432 to the kc nodes only.

PostgreSQL minor updates are not installed by unattended-upgrades (security pocket
only); they arrive with the next `ansible-playbook site.yml` run.

### Backups

`keycloak-db-backup.timer` runs nightly at 02:00 → `/var/backups/keycloak/keycloak-<UTC time>.dump`
(`pg_dump` custom format, 0600). Dumps older than 7 days are deleted, only after a successful new dump.

```bash
# on db1 (vagrant ssh db1)
systemctl list-timers keycloak-db-backup.timer      # next/last run
sudo systemctl start keycloak-db-backup.service     # back up now
journalctl -u keycloak-db-backup.service            # result of past runs
sudo ls -l /var/backups/keycloak/
```

### Restore

Keycloak must not write to the database during a restore.

```bash
# 1. laptop, in ansible/: stop Keycloak on all nodes
ansible keycloak -b -m ansible.builtin.systemd_service -a "name=keycloak state=stopped"

# 2. db1: recreate an empty database and load the dump into it
sudo -u postgres dropdb keycloak
sudo -u postgres createdb --owner=keycloak --encoding=UTF8 --template=template0 keycloak
sudo -u postgres pg_restore --dbname=keycloak --no-owner --role=keycloak \
  /var/backups/keycloak/keycloak-<UTC time>.dump

# 3. laptop: start Keycloak again
ansible keycloak -b -m ansible.builtin.systemd_service -a "name=keycloak state=started"
```

`--no-owner --role=keycloak`: restored tables belong to the `keycloak` user,
whoever owned them when the dump was taken.
