# Failover scenarios

What happens when a part of the cluster goes away, and how to prove it.

Run `vagrant` commands from `vagrant/`, the rest from the repo root.
Have hello-app running (see README) and be logged in at `http://localhost:8000/helloZara`.

> Why `/login` and not a refresh: hello-app keeps its own session cookie, so
> `/helloZara` keeps working even with Keycloak completely down. `/login` sends the
> browser back to Keycloak; if Keycloak still knows you, you return **without a
> password prompt**. That is the real SSO failover test.

## 1. One Keycloak node dies

The SSO session is shared between nodes (Infinispan cache, and since KC 26 also
persisted in the database), so the surviving node still knows you.

1. Which node served you? (HAProxy logs the backend server per request)
   ```bash
   vagrant ssh lb1 -c "sudo tail -n 5 /var/log/haproxy.log"     # look for be_keycloak/kc1 or /kc2
   ```
2. Stop that node (example kc1), wait ~20 s (18 s graceful shutdown + health checks):
   ```bash
   vagrant ssh kc1 -c "sudo systemctl stop keycloak"
   ```
3. The survivor sees a 1-member cluster:
   ```bash
   vagrant ssh kc2 -c "sudo journalctl -u keycloak | grep ISPN000094 | tail -n 1"
   ```
   Expected: `Received new cluster view ... (1) [kc2-...]`
4. Browser: `http://localhost:8000/login` → back on `/helloZara`, **no password prompt**.
5. Start kc1 again and wait for the 2-member view before the next test:
   ```bash
   vagrant ssh kc1 -c "sudo systemctl start keycloak"
   vagrant ssh kc2 -c "sudo journalctl -u keycloak | grep ISPN000094 | tail -n 1"
   ```
   Expected: `(2) [kc2-..., kc1-...]`
6. Repeat 2–5 with kc2 stopped.

## 2. Hard crash (no graceful shutdown)

Same as 1, but kill the VM instead of stopping the service:
```bash
vagrant halt kc1 --force       # like pulling the power cable
```
Expected: a few requests may fail until HAProxy marks kc1 DOWN
(`fall 3` × `inter 5s` ≈ 15 s); after that `/login` works without a password prompt.
Bring it back with `vagrant up kc1`.

## 3. Rolling restart: zero failed requests

In terminal A, request Keycloak through HAProxy twice per second and record the status codes:
```bash
while true; do
  curl -s -o /dev/null -w '%{http_code}\n' --cacert ansible/pki/ca.crt \
    https://auth.lab.test/realms/demo/.well-known/openid-configuration
  sleep 0.5
done | tee /tmp/rolling-codes.txt
```
In terminal B:
```bash
cd ansible && ansible-playbook rolling_restart.yml
```
While it runs, use `/login` in the browser a few times. When it is done, stop terminal A (Ctrl-C) and count:
```bash
sort /tmp/rolling-codes.txt | uniq -c
```
Expected: only `200`, and every browser `/login` worked.
Why: each node is drained in HAProxy before it restarts and only re-enabled after `/health/ready`.

## 4. Load balancer dies (known gap)

```bash
vagrant ssh lb1 -c "sudo systemctl stop haproxy"
```
Expected: **total outage**, there is only one HAProxy (no keepalived / floating IP).
Start it again with `sudo systemctl start haproxy`.

## 5. Database dies (known gap)

```bash
vagrant ssh db1 -c "sudo systemctl stop postgresql"
```
Expected: new logins and admin changes fail (single PostgreSQL, no HA).
`/health/ready` on the Keycloak nodes reports the database check down.
Start it again with `sudo systemctl start postgresql`; Keycloak reconnects without a restart.
