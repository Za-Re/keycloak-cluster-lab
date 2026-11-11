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
| 3 | Keycloak cluster + HAProxy | ✅ |
| 4 | Keycloak configuration as code | ✅ |
| 5 | End-to-end tests + docs | 🔄 |

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

---

## Phase 3: Cluster + HAProxy

**Goal:** kc1 + kc2 form one cluster; HAProxy on lb1 is the only way in, over HTTPS, with
re-encryption to verified Keycloak nodes, a path allowlist, and restarts without downtime.

Built in small commits, one per step below.

### What to add, in order

| # | Piece | Why |
|---|---|---|
| 1 | `keycloak.conf.j2`: `cache-embedded-mtls-enabled=true`, `spi-sticky-session-encoder--infinispan--should-attach-route=false`, `shutdown-delay` (default `18s`) | Node-to-node traffic encrypted (default, stated explicitly). Sticky sessions not recommended since 26.6; `false` also silences the 26.8 deprecation warning. On stop the node reports not-ready but keeps serving, so HAProxy drops it first |
| 2 | Run `site.yml` without `--limit` | kc2 joins. No new code: inventory, `pg_hba`, firewall and role were already generic |
| 3 | Move `keycloak_https_port`, `keycloak_management_port` to `group_vars/all` | Role defaults are only visible to hosts running that role; lb1 needs the ports too |
| 4 | `roles/haproxy`: defaults, handler (reload), template, tasks; play in `site.yml` after Keycloak | See "HAProxy config" below |
| 5 | Path allowlist, `:80` → 301 to HTTPS, HSTS; ufw 80 + 443 | Only what browsers/apps need is public |
| 6 | Admin socket (`haproxy_admin_socket` in `group_vars/all`) + `rolling_restart.yml` | Drain → restart → wait ready → enable, one node at a time (`serial: 1`) |

### HAProxy config (2.8, checked against the 2.8 manual)

- **TLS in:** Mozilla "intermediate" v6.0: TLS 1.2+, AEAD ciphers, `no-tls-tickets`; HTTP/2 via `alpn`.
  Cert + key in one root-only `0600` PEM (HAProxy reads it as root, before switching user).
- **Headers:** `http-request set-header` for `X-Forwarded-For/Proto/Host/Port`. `set-header` removes
  the client's copy first, so they can't be spoofed. Keycloak trusts them only from lb1.
- **TLS out:** `ssl verify required ca-file ca.crt` + `verifyhost kcN` per server, so the cert must be our CA's
  **and** for that exact node. `verifyhost` also applies to health checks.
- **Health:** `option httpchk GET /health/ready`, `check check-ssl port 9000 inter 5s fall 3 rise 2 slowstart 60s`.
  `check-ssl` is required because `port` moves checks to another port. Down after 15 s → `shutdown-delay` must be longer.
- **Balancing:** `roundrobin`, no stickiness. Sessions live in the cluster cache.
- **Paths (keycloak.org/server/reverseproxy):** public `/realms/`, `/resources/`, `/.well-known/`;
  `/admin/` and `/realms/master/` only from `admin_allowed_cidrs`; everything else 403.
  Paths containing `..`, `//`, `%2e`, `%2f`, `%5c` get 400: Keycloak may normalize them into a path the
  prefix check didn't see. (`normalize-uri` would be cleaner but is experimental in 2.8.)
- **Safety:** `validate: haproxy -c -f %s` on the template; reload, not restart.

### Run

```bash
ansible-playbook site.yml      # changed>0 (kc2, lb1)
ansible-playbook site.yml      # changed=0
ansible-playbook rolling_restart.yml
```

### Verify

| Check | Command | Expected |
|---|---|---|
| Cluster of 2 | `ansible keycloak -b -m shell -a "journalctl -u keycloak --no-pager \| grep ISPN000094 \| tail -1 \| cut -c1-260"` | view ends with `(2) [kc1-..., kc2-...]` |
| Issuer through HAProxy | `curl -s --cacert pki/ca.crt https://auth.lab.test/realms/master/.well-known/openid-configuration \| python3 -m json.tool \| grep issuer` | `https://auth.lab.test/realms/master` |
| Old TLS refused | `curl -s --cacert pki/ca.crt --tlsv1.1 --tls-max 1.1 https://auth.lab.test/; echo $?` | `35` |
| Backends healthy | `ansible lb1 -b -m shell -a "echo 'show stat' \| nc -U /run/haproxy-admin.sock \| cut -d, -f1,2,18 \| grep be_keycloak"` | kc1, kc2, BACKEND all `UP` |
| Allowlist | `curl -s -o /dev/null -w '%{http_code}' --cacert pki/ca.crt --path-as-is https://auth.lab.test<path>` | `/` `/metrics` `/health/ready` → 403; `/realms//master/` `/realms/../admin/` → 400; `/admin/master/console/` → 200 from laptop |
| Admin blocked elsewhere | `ansible kc1 -m shell -a "curl -s -o /dev/null -w '%{http_code}' --resolve auth.lab.test:443:192.168.56.10 https://auth.lab.test/admin/master/console/"` | `403` |
| Redirect + HSTS | `curl -sI http://auth.lab.test/realms/x`; `curl -sI --cacert pki/ca.crt https://auth.lab.test/` | `301` to `https://`; `strict-transport-security: max-age=63072000` |
| Temporary admin works | token request to `/realms/master/protocol/openid-connect/token` with `client_id=admin-cli` | `{"access_token":"eyJ...` |
| Zero downtime | loop `curl` every 0.2 s printing non-200, while running `rolling_restart.yml` | no output |

### Known gaps (accepted for now)

- **No mTLS HAProxy → Keycloak.** Keycloak doesn't check who connects on 8443; firewall (only lb1) is the control.
  Would need `https-client-auth` (build-time) and a `clientAuth` cert for lb1.
- **Config changes restart all nodes at once:** the keycloak role's handler isn't rolling. In production:
  apply config, then `rolling_restart.yml`.
- **Auto-updates never run:** the bento box ships `apt-daily.timer` and `apt-daily-upgrade.timer` disabled, so
  the Phase 1 unattended-upgrades config is correct but unused. Enabling them failed (`Job failed`); not fixed yet.

### Pitfalls we hit

- **apt lock error (`Could not get lock ... held by process`):** a cancelled `ansible-playbook` left `apt-get`
  running on the VM. Wait for it to finish, then rerun.
- **`curl` cleans up paths itself:** use `--path-as-is` to really send `..` and `//`.
- **Servers start as UP in HAProxy:** absence of `DOWN` log lines isn't proof; ask the admin socket (`show stat`).
- **kc2 re-downloaded the tarball once** (file newer than the install). Harmless: checksum verified,
  unpack skipped via `creates:`, no restart.

---

## Phase 4: Configuration as code

**Goal:** everything inside Keycloak (admins, realms, roles, users, clients) defined in Ansible,
idempotent, driven by variables. `roles/keycloak_config`, using `community.general.keycloak_*` modules.

### Design decisions

- **Ansible gets its own identity:** client `ansible-admin` in `master` with a service account
  (client-credentials grant, realm role `admin`). The human admin has TOTP, which Ansible can't type.
  Secret: `vault_kc_ansible_client_secret`. Production would add least privilege and a secret manager.
- **Runs on the first kc node** (`hosts: keycloak[0]`), against `https://kc1:8443`. The modules
  verify TLS only with the system trust store (no CA-file option); kc1 has the lab CA, the laptop doesn't.
  No `become`: API calls don't need root. Once is enough: config lives in the shared database.
- **One login block:** `module_defaults` for `group/community.general.keycloak` sets the auth
  parameters for every module. `omit` works inside `module_defaults` (tested).
- **Bootstrap fallback:** the role tests "does ansible-admin get a token **and** does the admin API
  accept it?". If not (first run), it logs in as `temp-admin` to create/fix `ansible-admin`.
  Deleting `temp-admin` is the **last** task, so a first run never cuts off its own login.

### What to add, in order

| # | Piece | Why |
|---|---|---|
| 1 | `group_vars/all`: `keycloak_bootstrap_admin_user` (moved from roles/keycloak) | Both roles need it |
| 2 | `vault.yml` + `.example`: `vault_kc_ansible_client_secret` | Ansible's client secret |
| 3 | `defaults`: `keycloak_config_url`, `keycloak_config_client_id`, `keycloak_config_admin_user` | |
| 4 | Login check: token request (`uri`, `status_code: [200, 401]`) → admin API call with it → `set_fact` | Chooses ansible-admin or the temp-admin fallback |
| 5 | `ansible-admin` client: confidential, service account only, default scope `kc-admin-api` (`client_scopes_behavior: patch`) + realm role `admin` | Automation identity |
| 6 | master: `brute_force_protected: true` | Lock accounts after repeated failures |
| 7 | master: user `admin`, password from vault, `required_actions: [CONFIGURE_TOTP]`, realm role `admin` | Human admin, MFA enforced on first login |
| 8 | `defaults`: `keycloak_config_tenants` (`demo`, `tenant-a`, `tenant-b`), password policy, token lifespan, events expiration | Everything per-tenant is driven by these |
| 9 | `tasks/tenant.yml`, included per tenant (`loop_var: tenant`) inside the login block | Included tasks inherit `module_defaults` |
| 10 | Realm: `ssl_required: all`, no registration, brute force, policy `length(12) and notUsername and passwordHistory(3)`, access token 300 s, `revoke_refresh_token` + `refresh_token_max_reuse: 0`, login + admin events with 7-day expiry (`attributes.adminEventsExpiration`, no module option), admin event details off (may contain secrets) | Realm hardening from the spec. Policy ids checked in `/admin/serverinfo` |
| 11 | Realm role `hello-reader`; user `demo-user` (verified email, first/last name, vault password) with that role | First/last name avoid an "update profile" stop on first login |
| 12 | Client `fastapi-app`: confidential, standard flow only, `pkce.code.challenge.method: S256`, exact `redirect_uris` + `post.logout.redirect.uris`, `full_scope_allowed: false`, hardcoded `tenant` claim mapper | Matches the hello-app (PKCE S256, roles from `realm_access` in the access token, logout redirect) |
| 13 | `keycloak_client_rolescope` (no `client_scope_id` = realm roles): only `hello-reader` | With full scope off, the only role that can reach its tokens |
| 14 | Delete `temp-admin` (**last** task in main.yml) | Temporary account gone |
| 15 | `site.yml`: play `keycloak_config` on `keycloak[0]` | After HAProxy |

All tasks touching secrets have `no_log: true` (including those that only inherit the login from `module_defaults`).

Attribute and mapper names checked in the 26.8.2 source: `OIDCConfigAttributes.java`
(`pkce.code.challenge.method`, `post.logout.redirect.uris`), `HardcodedClaim.java`
(`oidc-hardcoded-claim-mapper`, `claim.value`), `OIDCAttributeMapperHelper.java` (`claim.name`,
`jsonType.label`, `*.token.claim`), `RealmAttributes.java` (`adminEventsExpiration`).

### Verify

| Check | Expected |
|---|---|
| Rerun | `keycloak_config_use_client: true`, `changed=0` |
| ansible-admin token roles (decode JWT payload) | contains `admin` |
| Password grant as `temp-admin` | `Invalid user credentials` (deleted) |
| Password grant as `admin` (correct password) | `Account is not fully set up` (TOTP pending; a wrong password would say "Invalid user credentials") |
| `GET /admin/realms/master` with ansible-admin token | `bruteForceProtected: True` |
| Browser `https://auth.lab.test/admin/` as `admin` | QR code for TOTP, then the console |
| Realm settings via `GET /admin/realms/<tenant>` | `sslRequired all`, policy, `accessTokenLifespan 300`, `revokeRefreshToken true`, events on, `adminEventsExpiration 604800`; Keycloak's own attributes still present |
| `demo-user` via admin API (`users?username=demo-user&exact=true`, `role-mappings/realm`) | email verified, no required actions, roles `hello-reader` + `default-roles-<tenant>` |
| Example token: `clients/<id>/evaluate-scopes/generate-example-access-token?userId=<id>&scope=openid profile email` | `iss` = `https://auth.lab.test/realms/<tenant>`, `tenant` = realm name, `realm_access.roles` = **only** `hello-reader`, `email` present |
| Password grant on `fastapi-app` | `unauthorized_client` / `invalid_client` |
| `/auth` without `code_challenge` | redirect with `error=invalid_request` (`code_challenge_method` missing) |
| `/auth` with another `redirect_uri` | `Invalid parameter: redirect_uri` |

**Browser trust:** import `ansible/pki/ca.crt` as an authority (Chrome: `chrome://certificate-manager/`,
or `certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Keycloak Lab CA" -i pki/ca.crt`). HSTS means
there's no "proceed anyway" button. Remove it after the lab (`certutil ... -D -n "Keycloak Lab CA"`).

### Pitfalls we hit

- **26.8.2 admin API audience check:** a valid `ansible-admin` token still got `401` on `/admin/...`.
  Since 26.8.2 tokens need the `kc-admin-api` client scope; only built-in clients (`admin-cli`, admin
  console) get it on upgrade (upgrading guide, 26.8.2 breaking changes). Lesson: test with a real admin
  call, not just "got a token".
- **`no_log` hides the error:** rerun the failing module ad hoc in check mode (`ansible kc1 -C -m ...`) to see it.
- **A wrong client secret also gives 401**, so it falls back to `temp-admin`; once that's deleted, a
  "temp-admin login failed" error usually means `vault_kc_ansible_client_secret` doesn't match.
- **keycloak_user doesn't compare passwords:** changing `vault_kc_admin_password` later won't update Keycloak.
- **`admin-cli` tokens have no `email` or roles:** it issues lightweight access tokens (only `sub`, `sid`,
  `exp`…). Check users and roles via the admin API, and app tokens via `evaluate-scopes`.

---

## Phase 5: End-to-end tests + docs

**Goal:** prove the whole chain with a real app, check every security boundary from outside,
lint the code, and write it down. Almost no new code: this phase is mostly tests.

### What to add, in order

| # | Piece | Why |
|---|---|---|
| 1 | hello-app `.env`: `OIDC_ISSUER=https://auth.lab.test/realms/demo`, `OIDC_CLIENT_SECRET` = `vault_kc_fastapi_client_secret` | Points the existing app at the cluster; no app code changes |
| 2 | Start the app with `SSL_CERT_FILE=<repo>/ansible/pki/ca.crt uvicorn app.main:app --port 8000` | httpx trusts only certifi; `SSL_CERT_FILE` (httpx ≥ 0.28 reads it) makes it trust the lab CA |
| 3 | `ansible/.ansible-lint` (`exclude_paths: [collections/]`) | Installed collections are third-party code with test playbooks that can't pass syntax-check |
| 4 | `ansible/.ansible-lint-ignore`: `args[module]` for the two `keycloak_config` task files | False positive: lint doesn't see `auth_keycloak_url` from `module_defaults`. Scoped to 2 files, not global |
| 5 | Fix findings: trailing spaces, task names capitalized, `{{ tenant }}` at the **end** of names, `# noqa: no-handler` on the PGDG apt refresh | A handler would run at the end of the play; the install right after needs the fresh cache |
| 6 | HAProxy: `acl admin_path path /admin` (exact) | `/admin` without slash matched neither ACL → 403 |
| 7 | `docs/FAILOVER.md` | Failover, rolling restart and restore tests, run once everything is built |

Run `ansible-lint` in `ansible/` after **every** change, not only at the end (install: `sudo apt install ansible-lint`, matches apt's ansible-core).

### Verify

| Check | Command | Expected |
|---|---|---|
| Lint | `ansible-lint` | `Passed: 0 failure(s)`, profile `production` |
| Idempotency | `ansible-playbook site.yml` twice | `changed=0` on every host |
| Cluster view | `vagrant ssh kc1 -c "sudo journalctl -u keycloak \| grep ISPN000094 \| tail -n 1"` | `(2) [kc1-..., kc2-...]` |
| App login | browser `http://localhost:8000/helloZara` → login as `demo-user` | greeting with role `hello-reader`; `/logout` shows Keycloak's confirm page |
| Admin from laptop | `curl -s -o /dev/null -w '%{http_code}\n' --cacert pki/ca.crt https://auth.lab.test/admin/` (and `/admin`) | `302` |
| Admin from elsewhere | same from kc1 with `--resolve auth.lab.test:443:192.168.56.10` for `/admin`, `/admin/`, `/realms/master/` | `403` |
| Nodes not reachable directly | `curl -m 3 ... https://192.168.56.21:{8443,9000}/health/ready` from the laptop | `000` (timeout, ufw drops) |
| DB refuses lb1 | `vagrant ssh lb1 -c "timeout 3 bash -c '</dev/tcp/192.168.56.11/5432'"` | fails (ufw) |
| DB refuses no TLS | on kc1 (temporary `postgresql-client`): `psql 'host=db1 dbname=keycloak user=keycloak sslmode=disable'` | `pg_hba.conf rejects connection ... no encryption` |
| DB accepts TLS | same with `sslmode=verify-full sslrootcert=/etc/ssl/lab/ca.crt` | password prompt |

Failover, rolling restart and backup/restore: [FAILOVER.md](FAILOVER.md).

### Pitfalls we hit

- **`SSL_CERT_FILE` in `.env` does nothing:** pydantic-settings reads `.env` into the app's settings
  only, it doesn't export environment variables. Set it in the shell that starts uvicorn.
- **Each browser has its own trust store:** Chrome on Linux uses `~/.pki/nssdb`, Firefox (and snap
  browsers) their own. Restart the browser fully after importing. `certutil -L` without
  `-d sql:$HOME/.pki/nssdb` fails with `SEC_ERROR_BAD_DATABASE` (wrong database, not a failed import).
- **Account console ≠ admin console:** `/realms/master/account` is every user's self-service page
  (profile, 2FA). The admin UI is `/admin/` → `/admin/master/console/`.
- **ansible-lint scans everything under the project**, including `collections/` (4000+ files) → exclude it.
- **Test with the exact URLs people type:** curl tested `/admin/`, the browser used `/admin` → 403.
