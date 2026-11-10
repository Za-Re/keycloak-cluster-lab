# Build guide

How to build this project from an empty folder, phase by phase. Each phase says
what to add, in what order, why, and how to prove it works before moving on.

The finished code is in the repo; use it as the reference. This guide explains
the path. For day-to-day usage (setup, run, backup/restore) see the [README](../README.md).

**Rule for every phase:** build → run → run again (must report `changed=0`) → verify → next phase.

| Phase | Goal | Status |
|---|---|---|
| 0 | VMs, Ansible base, secrets, PKI | ✅ |
| 1 | Hardened base system + PostgreSQL | ✅ |
| 2 | One Keycloak node (kc1) | ✅ |
| 3 | Keycloak cluster + HAProxy | — |
| 4 | Keycloak configuration as code | — |
| 5 | End-to-end tests + docs | — |

Commands starting with `ansible` run in `ansible/`; `vagrant` commands run in `vagrant/`.

---

## Phase 0: Foundation

**Goal:** VMs you can reach with Ansible, a place for secrets, and a TLS certificate for every host.

### What to add, in order

| # | File | Purpose |
|---|---|---|
| 1 | `~/.ssh/keycloak-lab_ed25519` (not in repo) | Dedicated lab SSH key, isolated from your personal key |
| 2 | `vagrant/Vagrantfile` | lb1, db1, kc1..kcN on static IPs `192.168.56.x`; pinned box; shell provisioner adds the lab public key; input checks (`KC_NODES` 1–5, key file exists and isn't empty) |
| 3 | `ansible/ansible.cfg` | Inventory, roles path, user `vagrant`, lab key, vault password file, project-local `.known_hosts` |
| 4 | `ansible/requirements.yml` | Collections pinned to versions that support your ansible-core |
| 5 | `ansible/inventory/hosts.ini` | Static inventory: groups `loadbalancer`, `database`, `keycloak` |
| 6 | `ansible/group_vars/all/main.yml` | Shared variables (public hostname, admin CIDRs, PKI paths) |
| 7 | `ansible/group_vars/all/vault.yml.example` → `vault.yml` | The 5 secrets; real file is encrypted and gitignored |
| 8 | `ansible/roles/pki/` | Local CA + per-host certs on the laptop; copies only what each host needs |
| 9 | `ansible/group_vars/loadbalancer.yml` | Extra SAN `DNS:auth.lab.test` for lb1 only |
| 10 | `ansible/site.yml` | Main playbook; each phase adds plays |
| 11 | `.gitignore` | Keys, certs, vault password, `vault.yml`, collections, `.vagrant/`, `.known_hosts` |

### Key ideas

- **Vagrant creates machines, Ansible configures them.** No Vagrant Ansible provisioner, so
  the same playbooks work against real servers with a different inventory.
- **One dedicated SSH key** injected by a tiny shell provisioner, instead of Vagrant's per-VM
  keys under `.vagrant/` (which change on every rebuild).
- **Host key checking stays on**, but in a project-local `.known_hosts` with
  `StrictHostKeyChecking=accept-new` (trust on first use). Delete it after `vagrant destroy`.
- **Collection versions:** check `requires_ansible` on Galaxy; newer majors drop older ansible-core.
- **Secrets naming:** `vault_` prefix = "comes from the encrypted file". Generate values with
  `openssl rand -hex 24` (hex never needs escaping).
- **PKI:** the CA key never leaves the laptop. Each cert has SANs `DNS:<host>` + `IP:<ip>`
  (clients check SANs, not the CN). File-based CSRs (not `_pipe`) keep reruns at `changed=0`.
  Keys on hosts are `0640 root:ssl-cert` so service users can read them via group membership.

### Run

```bash
ssh-keygen -t ed25519 -f ~/.ssh/keycloak-lab_ed25519 -C keycloak-lab -N ""
ansible-galaxy collection install -r requirements.yml
( umask 077; openssl rand -base64 32 > .vault_pass ); chmod 600 .vault_pass
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # fill in
ansible-vault encrypt group_vars/all/vault.yml
vagrant up
echo "192.168.56.10 auth.lab.test" | sudo tee -a /etc/hosts
ansible all -m ping
ansible-playbook site.yml   # twice
```

### Verify

| Check | Command | Expected |
|---|---|---|
| VMs up | `vagrant status` | all `running (virtualbox)` |
| Ansible reaches all hosts | `ansible all -m ping` | `SUCCESS` / `"ping": "pong"` per host |
| Vault encrypted | `head -1 group_vars/all/vault.yml` | `$ANSIBLE_VAULT;1.1;AES256` |
| Certs valid | `openssl verify -CAfile pki/ca.crt pki/lb1.crt pki/kc1.crt` | `OK` per file |
| lb1 has public name | `openssl x509 -in pki/lb1.crt -noout -ext subjectAltName` | `DNS:lb1, IP Address:192.168.56.10, DNS:auth.lab.test` |
| Local key perms | `ls -l pki/*.key` | `-rw-------` |
| Host got only its files | `ansible kc1 -b -m command -a "ls -l /etc/ssl/lab"` | `ca.crt`, `server.crt` (0644), `server.key` (0640 root:ssl-cert) |
| Nothing secret in git | `git status --short` | no `pki/`, `.vault_pass`, `vault.yml`, `.known_hosts` |
| Name resolution | `getent hosts auth.lab.test` | `192.168.56.10` |

### Pitfalls we hit

- `vagrant ssh auth.lab.test` fails: `vagrant ssh` takes the **machine name** (`lb1`), not a hostname.
- `.known_hosts` landed in `vagrant/`: the path in `-o UserKnownHostsFile=.known_hosts` is
  relative to the **current folder**. Run Ansible and manual `ssh` from `ansible/`.
- Ansible only reads `ansible.cfg` from the current folder: always `cd ansible` first.

---

## Phase 1: Base system + database

**Goal:** every VM hardened the same way; PostgreSQL on db1 reachable only by kc nodes, only over TLS, backed up nightly.

### Part A: `roles/common` (all hosts)

| # | Piece | Why |
|---|---|---|
| 1 | `defaults/main.yml`: `common_timezone: Etc/UTC` | Servers in UTC: logs line up, no DST jumps |
| 2 | apt `upgrade: dist` + install `unattended-upgrades`, `chrony`, `ufw` | Patched base; time sync; firewall |
| 3 | Timezone + chrony running/enabled | TLS, token expiry and cluster failure detection depend on correct clocks |
| 4 | `/etc/apt/apt.conf.d/20auto-upgrades` | Turns daily auto-updates on |
| 5 | `/etc/apt/apt.conf.d/52unattended-upgrades-security-only` | `#clear` Ubuntu's origin lists, allow `-security` only. No auto-reboot (would hit all kc nodes at once) |
| 6 | `/etc/ssh/sshd_config.d/00-hardening.conf` + `validate: sshd -t -f %s` + reload handler | No root login, no passwords, keys only. sshd keeps the **first** value → `00-` wins. Validation prevents lockout; reload keeps sessions |
| 7 | ufw: allow 22 **then** enable with default deny incoming | Wrong order cuts your own connection |
| 8 | `blockinfile` in `/etc/hosts` from `groups['all']` | Names without DNS (needed for TLS `verify-full` against `db1`) |
| 9 | `site.yml`: play `common` on `all`, first | Base before everything else |

### Part B: `roles/postgresql` (db1)

| # | Piece | Why |
|---|---|---|
| 1 | `defaults/main.yml`: version `18`, PGDG key URL + **fingerprint**, backup settings | Explicit major version = no surprise major upgrades. 18 is Keycloak's tested version |
| 2 | Helper packages: `gpg`, `python3-debian` (for `deb822_repository`), `python3-psycopg2` (for postgres modules) | Module prerequisites on the target |
| 3 | Download PGDG key → `gpg --show-keys --with-colons` → `assert` fingerprint | HTTPS proves the server; the fingerprint proves the key. apt trusts everything this key signs |
| 4 | `deb822_repository` (`noble-pgdg`) + apt refresh when changed | Official PostgreSQL repo, modern sources format |
| 5 | Install `postgresql-18`; add `postgres` to `ssl-cert` | Read `/etc/ssl/lab/server.key` |
| 6 | `conf.d/10-lab.conf`: `listen_addresses` = private IP, `ssl=on` + lab cert, TLS ≥1.2, `scram-sha-256` | PostgreSQL keeps the **last** value → drop-in overrides Ubuntu defaults (opposite of sshd!) |
| 7 | `templates/pg_hba.conf.j2` | First match wins: `local all postgres peer`; one `hostssl keycloak keycloak <kc-IP>/32 scram-sha-256` per kc node; explicit `reject` for everything else |
| 8 | Handler: restart `postgresql@18-main`; `meta: flush_handlers` | `listen_addresses` needs a restart; flush so the next tasks talk to the configured server |
| 9 | `postgresql_user` (`no_log`) + `postgresql_db` (UTF8, `template0`) as `become_user: postgres` | Peer auth over the socket. Reruns stay `changed=0` with SCRAM (module re-hashes with the stored salt) |
| 10 | ufw 5432 from each kc IP | Second layer next to `pg_hba` |
| 11 | Backup: script + `.service` + `.timer` templates, `/var/backups/keycloak` 0700 | `pg_dump -Fc` to `*.partial`, rename on success, prune >7 days **only after** success. `UMask=0077`, sandboxed unit. Timer: `Persistent=true` catches missed runs |
| 12 | `group_vars/all/main.yml`: `keycloak_db_name`, `keycloak_db_user` | Shared with the Keycloak role in Phase 2 |
| 13 | `site.yml`: play `postgresql` on `database`, after PKI | Needs the certs |
| 14 | README: backup + restore procedure | A backup without a restore procedure is not a backup |

### Run

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml      # changed>0
ansible-playbook site.yml      # changed=0
```

### Verify: base system

| Check | Command | Expected |
|---|---|---|
| Effective SSH settings | `ansible all -b -m shell -a "sshd -T \| grep -E '^(permitrootlogin\|passwordauthentication\|authenticationmethods)'"` | `no` / `no` / `publickey` |
| Password login refused | `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password vagrant@192.168.56.21` | `Permission denied (publickey).` (no password prompt) |
| Firewall | `ansible all -b -m command -a "ufw status verbose"` | `active`, `deny (incoming)`, `22/tcp ALLOW` |
| Auto-updates scope | `ansible kc1 -b -m shell -a "unattended-upgrade --dry-run --debug 2>&1 \| grep -i origin"` | only `noble-security` |
| Time | `ansible all -m command -a "timedatectl show -p Timezone -p NTPSynchronized"` | `Etc/UTC`, `yes` |
| Names | `ansible kc1 -m command -a "getent hosts db1"` | `192.168.56.11 db1` |

### Verify: database

| Check | Command | Expected |
|---|---|---|
| Version, listen, TLS | `ansible db1 -b -m shell -a "sudo -u postgres psql -Atc 'show server_version' -c 'show listen_addresses' -c 'show ssl'; ss -tln \| grep 5432"` | `18.x`, `192.168.56.11`, `on`; listening only on `192.168.56.11:5432` |
| Firewall | `ansible db1 -b -m command -a "ufw status"` | 5432 allowed only from kc IPs |
| TLS login from kc1 | on kc1: `psql "host=db1 dbname=keycloak user=keycloak sslmode=verify-full sslrootcert=/etc/ssl/lab/ca.crt" -c "select ssl, version from pg_stat_ssl where pid = pg_backend_pid();"` | `t \| TLSv1.3` |
| No TLS refused | on kc1: `psql "host=db1 dbname=keycloak user=keycloak sslmode=disable"` | `pg_hba.conf rejects connection ... no encryption` |
| Other hosts blocked | `vagrant ssh lb1 -c "nc -zv -w3 db1 5432"` | `timed out` (firewall drops silently) |
| Backup works | `ansible db1 -b -m command -a "systemctl start keycloak-db-backup.service"` then `ansible db1 -b -m shell -a "ls -l /var/backups/keycloak; systemctl list-timers keycloak-db-backup.timer"` | a `-rw------- postgres` `.dump` file; timer NEXT ≈ 02:00 |

For the `psql` tests, install a client temporarily (`ansible kc1 -b -m apt -a "name=postgresql-client"`)
and remove it afterwards **with its dependencies**:
`ansible kc1 -b -m apt -a "name=postgresql-client-16,postgresql-client-common,libpq5 state=absent purge=true"`.
The password is `vault_db_password`: `ansible-vault view group_vars/all/vault.yml | grep vault_db_password`.

### Pitfalls we hit

- **Checks showed factory defaults:** the role hadn't been applied yet. Read the PLAY RECAP before verifying.
- **`NTPSynchronized=no` after a laptop sleep:** chrony logs `Forward time jump detected!`, discards
  samples and resyncs within minutes. Normal for VMs.
- **Own hostname resolves to `127.0.2.1`:** Vagrant adds that line. Harmless for PostgreSQL (explicit
  listen address); Keycloak must bind/advertise its private IP explicitly (Phase 2/3).
- **`[WARNING]: Module remote_tmp /root/.ansible/tmp did not exist`:** shown once per host, harmless.
- **Wrong DB password:** the DB user's password is `vault_db_password`, not the Keycloak test user's.
- **`apt remove postgresql-client` left packages behind:** it's a metapackage; its dependencies stay
  as `auto-removable`. Remove them by name (avoid a blanket `autoremove`).
- `-m command` doesn't support pipes/redirects; use `-m shell` when you need them. Ad-hoc
  `command`/`shell` always report `CHANGED`. That's expected.

---

## Phase 2: One Keycloak node (kc1)

**Goal:** Keycloak 26.8.2 running from the official tarball on kc1 under a hardened systemd unit,
talking to PostgreSQL over verified TLS, reachable only from lb1.

**Before writing config:** check every option against the docs *and* the installed version.
`kc.sh build --help` lists the build-time options; `kc.sh start --help-all` lists everything usable at start.

### What to add, in order (`roles/keycloak`)

| # | Piece | Why |
|---|---|---|
| 1 | `defaults/main.yml`: version, Java package, user, paths | `/opt/keycloak-<version>` + symlink `/opt/keycloak`: upgrade = unpack next to it, repoint |
| 2 | `openjdk-21-jre-headless` | Keycloak is a Java app. Docs: 26.6+ also supports 25; the official container image uses 21. JRE + headless = fewer packages to patch |
| 3 | System user `keycloak` (nologin, no home, in `ssl-cert`) | A compromised service gets this user, not root; `ssl-cert` lets it read `server.key` |
| 4 | Pin the tarball's SHA-256 in `defaults`; `get_url` with `checksum:` | Take the digest from the GitHub release metadata. Reruns skip the download |
| 5 | One-time on the laptop: `gpg --verify` the `.asc` against keycloak.org/keys, in a separate `--homedir` | A checksum from the same source proves integrity; the signature proves who published it |
| 6 | `unarchive` with `--strip-components=1`, `creates:`, root-owned; `data/` owned by `keycloak` 0750 | The service can't modify its own code; `data/` is the only writable path |
| 7 | Build options as a dict in `defaults` (`keycloak_build_options`) | One definition, used by the config and by the rebuild hash |
| 8 | `templates/keycloak.conf.j2` → `conf/keycloak.conf`, `0640 root:keycloak`, `no_log` | Holds the DB password |
| 9 | `kc.sh build` with `creates:` on a stamp named after a hash of version + build options | Rebuild only when needed. Stamp written after success; old stamps removed first; `data/` chowned back (build runs as root) |
| 10 | ufw: 8443 + 9000 from lb1; 7800 + 57800 from other kc nodes | Generated from the inventory (`product` filter) |
| 11 | `/etc/keycloak/bootstrap.env` (0600, first node only, while marker absent) | Bootstrap admin only for the first start; unit file never holds the password |
| 12 | `templates/keycloak.service.j2`: `start --optimized`, `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `ReadWritePaths`, `Restart=on-failure`, `LimitNOFILE`, `SuccessExitStatus=143` | Sandbox. Java exits 143 on SIGTERM, which is a normal stop |
| 13 | Enable (don't start) → `flush_handlers` → ensure `started` | Starts exactly once. A start plus a handler restart would kill Keycloak while it creates its schema |
| 14 | `uri` on `/health/ready` with `ca_path`, up to 5 min | Role ends only when the node is ready |
| 15 | Delete `bootstrap.env`, write marker | Password off the disk; never recreated |
| 16 | `site.yml`: play on group `keycloak` | Phase 2 runs it with `--limit kc1` |

### Build-time vs runtime (from `kc.sh build --help`, 26.8.2)

| Type | Options used here |
|---|---|
| **Build-time** (needs `kc.sh build`) | `db`, `health-enabled`, `metrics-enabled` |
| **Runtime** (restart is enough) | everything else: `db-url-host`, `db-url-database`, `db-username`, `db-password`, `db-tls-mode`, `db-tls-trust-store-file`, `hostname`, `http-enabled`, `http-host`, `https-port`, `https-certificate-file`, `https-certificate-key-file`, `http-management-host`, `http-management-port`, `proxy-headers`, `proxy-trusted-addresses`, `cache`, `cache-stack`, `cache-embedded-network-bind-address` |

`cache` and `cache-stack` are **runtime** in this version, unlike many older guides.

### Key ideas

- **`proxy-headers=xforwarded` fits TLS re-encryption:** HAProxy terminates the browser's TLS, reads the
  request, and opens new HTTPS to the node. `proxy-trusted-addresses` limits who may set those headers.
- **Explicit bind addresses:** the default `SITE_LOCAL` could pick VirtualBox's NAT IP, `10.0.2.15` on
  every VM, and the own hostname resolves to `127.0.2.1`. So `http-host`, `http-management-host` and
  `cache-embedded-network-bind-address` are set to the node's `192.168.56.x` IP.
- **Management interface:** port 9000 inherits HTTPS from the main interface. Health and metrics live there, never on 8443.
- **`db-tls-mode=verify-server`** replaces hand-written `sslmode` URL parameters. The docs don't say it
  checks the hostname, so we tested it (below): it does.
- **Temporary admin until Phase 4:** deleting `bootstrap.env` removes the password from disk, not the account.
  Phase 4 creates the permanent admin and deletes the temporary one. Until then no admin endpoint is
  reachable from outside. The running process also keeps the variables in memory until its next restart.

### Run

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --limit kc1      # changed>0; the ready wait may retry for a few minutes
ansible-playbook site.yml --limit kc1      # changed=0
```

### Verify

| Check | Command | Expected |
|---|---|---|
| Java | `ansible kc1 -m command -a "java -version"` | `openjdk version "21.x"` |
| Install layout | `ansible kc1 -m command -a "ls -l /opt"` | symlink `keycloak -> /opt/keycloak-26.8.2` |
| Ownership | `ansible kc1 -m shell -a "stat -c '%U %G %a %n' /opt/keycloak/bin/kc.sh /opt/keycloak/data"` | `root root 755`, `keycloak keycloak 750` |
| Service user | `ansible kc1 -b -m command -a "ps -o user= -C java"` | `keycloak` |
| Ready, from lb1 | `vagrant ssh lb1 -c "curl -s --cacert /etc/ssl/lab/ca.crt https://kc1:9000/health/ready"` | `"status": "UP"` |
| Blocked from laptop | `curl -k --connect-timeout 3 https://192.168.56.21:9000/health/ready` (and `:8443/`) | timeout |
| Bootstrap secret gone | `ansible kc1 -b -m command -a "ls -A /etc/keycloak"` | only `.bootstrap-done` |
| Build stamp | `ansible kc1 -b -m shell -a "ls -A /opt/keycloak-26.8.2 \| grep built"` | one `.built-<hash>` |
| DB over TLS | `ansible db1 -b -m shell -a "sudo -u postgres psql -Atc \"select client_addr, ssl, version from pg_stat_ssl join pg_stat_activity using (pid) where usename='keycloak' limit 1\""` | `192.168.56.21\|t\|TLSv1.3` |

**DB hostname verification (negative test).** Connect through a name that isn't in db1's certificate.
Keycloak is down for about a minute; this restores itself:

```bash
ansible kc1 -b -m shell -a "C=/opt/keycloak/conf/keycloak.conf
printf '\n192.168.56.11 dbalias\n' >> /etc/hosts
getent hosts dbalias
sed -i 's/^db-url-host=.*/db-url-host=dbalias/' \$C
T=\$(date '+%F %T'); systemctl restart keycloak
timeout 40 journalctl -u keycloak --since \"\$T\" -f --no-pager | grep -m5 -i -E 'does not match|validation failed|UnknownHost|started in' | cut -c1-230
sed -i '/dbalias/d' /etc/hosts; sed -i 's/^db-url-host=.*/db-url-host=db1/' \$C
systemctl restart keycloak; true"
```

Expected: `getent` prints the alias, then `PGjdbcHostnameVerifier: Server name validation failed`.

### Pitfalls we hit

- **A negative test must fail for the right reason.** The first hostname test "failed" with
  `UnknownHostException: dbalias`: the name didn't resolve, so TLS was never reached and nothing was
  proven. Print `getent` first and read the actual error.
- **Docs vs. your plan:** the spec said 26.8.1 and Java 21. Newest was 26.8.2, and the getting-started
  guide says Java 25. Read the docs for your exact version before pinning.
- **SHA-1 isn't the only checksum:** the downloads page offers SHA-1 only, but the GitHub release
  metadata carries a SHA-256 for each asset.
- **`.asc` warning `This key is not certified`:** GPG means nobody in your keyring vouched for the key.
  Compare the fingerprint with keycloak.org/keys yourself.
- **`usename` isn't a typo:** PostgreSQL's column is spelled that way; `username` fails with `does not exist`.
- **`[DEPRECATION WARNING]: community.general.yaml`:** use `callback_result_format = yaml` in
  `ansible.cfg` instead of `stdout_callback = yaml`.
