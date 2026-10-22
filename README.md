# Homelab Ingress with Fly.io, WireGuard, nginx, and Dokku

This repository documents the public-ingress setup used to expose applications running on a home Dokku server without opening inbound ports on the home router.

The basic idea is:

```text
Internet
   |
   v
Cloudflare DNS/proxy
   |
   v
Fly.io public edge / TLS
   |
   v
Stateless nginx Machine(s) on Fly
   |
   | Fly private 6PN network
   v
Fly WireGuard gateway
   |
   | encrypted WireGuard tunnel
   v
Home server (`box`)
   |
   v
Dokku nginx
   |
   +--> trackanything.io --> Track Anything container
   +--> djmagic.io       --> DJ Magic container
   +--> future domains   --> other Dokku apps
```

The important architectural point is that the home server does **not** connect directly to a particular Fly Machine. The home server is a WireGuard peer on the Fly organization's private IPv6 network (6PN). Any authorized Fly Machine in that organization can reach the home server through its private `fdaa:` address.

That makes the Fly nginx Machines disposable and replaceable. Fly can stop, replace, or recreate them without requiring the WireGuard configuration on the home server to change.

## Why this setup exists

Goals:

- Host multiple small projects on a home server.
- Avoid inbound router port forwarding.
- Keep the home ISP/DHCP address out of the public routing design.
- Use Fly as a small, stable public ingress layer.
- Use one WireGuard tunnel for many homelab applications.
- Let Dokku continue doing hostname-based routing to individual applications.
- Keep the Fly layer stateless and boring.

The home server initiates the WireGuard relationship with Fly. Its ordinary LAN DHCP address can therefore change without changing the public application architecture.

## Components

### Home server

Current server:

```text
hostname: box
OS: Ubuntu 26.04
Dokku: 0.38.30
Docker: 29.8.1
```

Track Anything currently runs through Dokku with its application listening on port 8080. Dokku nginx is reachable from the WireGuard interface on port 8080.

### Fly.io

Fly provides two separate pieces of this design:

1. Public ingress/TLS through Fly Proxy.
2. The organization's private IPv6 network (6PN), into which the home server is connected using WireGuard.

The ingress application currently uses the Fly app name:

```text
trackanything
```

Long term, this can be treated as a general-purpose homelab ingress app and renamed/recreated under a name such as `homelab-ingress` if desired.

### Cloudflare

Cloudflare will eventually hold the public DNS records for application domains such as:

```text
trackanything.io
djmagic.io
```

Cloudflare is not required to prove the Fly-to-homelab tunnel itself. It is added after the `*.fly.dev` route works end-to-end.

### Dokku

Dokku performs the final application selection using the HTTP `Host` header.

For example:

```text
Host: trackanything.io -> Track Anything
Host: djmagic.io       -> DJ Magic
```

The Fly nginx proxy deliberately preserves this header.

---

# 1. Install WireGuard on the home server

On `box`:

```bash
sudo apt update
sudo apt install -y wireguard
wg --version
```

## Create the Fly WireGuard peer

On a machine authenticated with the Fly CLI:

```bash
fly wireguard create
```

Fly generates a WireGuard configuration. In this installation it looked conceptually like:

```ini
[Interface]
PrivateKey = REDACTED
Address = fdaa:6:5dee:a7b:910f:53be:b775:e002/120
DNS = fdaa:6:5dee::3

[Peer]
PublicKey = REDACTED
AllowedIPs = fdaa:6:5dee::/48
Endpoint = sjc2.gateway.6pn.dev:51820
PersistentKeepalive = 15
```

**Never commit the real WireGuard configuration or private key to Git.**

The important values are:

- The home server receives its own private Fly IPv6 address.
- `AllowedIPs` points the organization's Fly private network through the tunnel.
- The peer endpoint is a Fly WireGuard gateway, **not an application Machine**.
- `PersistentKeepalive = 15` keeps the NAT mapping/tunnel active.

Copy the generated config to the home server. For example:

```bash
scp box-fly.conf administrator@box:/tmp/box-fly.conf
```

Then on `box`:

```bash
sudo mv /tmp/box-fly.conf /etc/wireguard/fly.conf
sudo chown root:root /etc/wireguard/fly.conf
sudo chmod 600 /etc/wireguard/fly.conf
```

Bring up the tunnel:

```bash
sudo wg-quick up fly
```

Inspect it:

```bash
sudo wg show
ip addr show fly
```

A working connection should show a recent handshake and transferred bytes, for example:

```text
latest handshake: 9 seconds ago
transfer: 436 B received, 272 B sent
persistent keepalive: every 15 seconds
```

## Start WireGuard automatically at boot

Once the tunnel has been proven manually:

```bash
sudo systemctl enable wg-quick@fly
```

Verify:

```bash
systemctl is-enabled wg-quick@fly
```

Expected:

```text
enabled
```

---

# 2. Verify Dokku through the WireGuard address

Before introducing a Fly proxy, prove that the service is reachable through the home server's Fly private address.

For Track Anything, the application exposes:

```text
GET /healthz
```

A healthy response is HTTP 200 with:

```text
ok
```

From `box`, test its WireGuard address directly while supplying the hostname Dokku expects:

```bash
curl -v \
  -H 'Host: trackanything.box' \
  http://[fdaa:6:5dee:a7b:910f:53be:b775:e002]:8080/healthz
```

Expected result:

```text
HTTP/1.1 200 OK
...
ok
```

This proves:

```text
WireGuard interface
      |
      v
Dokku nginx
      |
      v
Track Anything
      |
      v
SQLite
```

A normal laptop on the LAN will generally **not** be able to curl this `fdaa:` address unless that laptop is itself connected/routed into Fly's private network. That is expected.

---

# 3. Create the Fly ingress proxy

The Fly side is intentionally tiny. It contains only:

```text
Dockerfile
nginx.conf
fly.toml
README.md
```

The nginx container is stateless. It accepts traffic from Fly Proxy and forwards it over Fly's private network to the home server.

## Dockerfile

```dockerfile
FROM nginx:1.29-alpine

COPY nginx.conf /etc/nginx/nginx.conf
```

## nginx.conf

```nginx
events {}

http {
    server {
        listen 8080;
        listen [::]:8080;

        # DJMagic catalog uploads are up to 100 MB. Stream them to the homelab
        # instead of buffering here, and give slow connections time to finish.
        client_max_body_size 101m;
        proxy_request_buffering off;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;

        location / {
            proxy_pass http://[fdaa:6:5dee:a7b:910f:53be:b775:e002]:8080;

            proxy_http_version 1.1;

            proxy_set_header Host $host;

            # Fly Proxy provides the original visitor IP.
            # Overwrite this header before forwarding it to the homelab.
            proxy_set_header Fly-Client-IP $http_fly_client_ip;

            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
            proxy_set_header X-Forwarded-Host $host;
        }
    }
}
```

nginx's default `client_max_body_size` is 1 MB, which would reject large uploads here before they reach the homelab. Each app's limit still applies behind this one (DJMagic's Dokku nginx allows 101 MB; Track Anything keeps Dokku's 1 MB default).

The critical line for multi-app hosting is:

```nginx
proxy_set_header Host $host;
```

The proxy does **not** decide whether a request belongs to Track Anything, DJ Magic, or another application. It forwards the original hostname to Dokku, and Dokku makes that decision.

The `proxy_pass` address is the home server's WireGuard/6PN address, not its LAN address.

## fly.toml

Current configuration:

```toml
app = "trackanything"
primary_region = "sjc"

[build]

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1

  [[http_service.checks]]
    interval = "30s"
    timeout = "5s"
    grace_period = "10s"
    method = "GET"
    path = "/healthz"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory = "256mb"
```

`sjc` is used as the primary region. The WireGuard peer in this installation also uses an SJC Fly gateway.

The important availability settings are:

```toml
auto_stop_machines = "stop"
auto_start_machines = true
min_machines_running = 1
```

This allows spare Machines to stop while ensuring one Machine in the primary region remains available.

The Fly health check is intentionally end-to-end. `/healthz` is not served by the Fly nginx container itself. It is proxied to the real Track Anything application at home.

Therefore a successful Fly health check proves approximately:

```text
Fly Machine
   |
   v
6PN
   |
   v
WireGuard
   |
   v
box
   |
   v
Dokku
   |
   v
Track Anything
   |
   v
SQLite
```

---

# 4. Deploy the proxy

From the ingress repository:

```bash
fly deploy
```

To keep two disposable ingress Machines available:

```bash
fly scale count 2
```

Inspect them with:

```bash
fly status
```

Destroying an ingress Machine is not a data-loss event. These Machines contain no application state and do not own the WireGuard tunnel. If one is accidentally destroyed, recreate the desired count with `fly scale count`.

---

# 5. Verify public Fly ingress

Before configuring a real domain or Cloudflare, use the Fly-provided hostname:

```bash
curl -v https://trackanything.fly.dev/healthz
```

A successful test returns:

```text
HTTP/2 200
...
ok
```

This proves the complete path:

```text
Internet client
     |
     | HTTPS
     v
Fly public edge
     |
     v
Fly nginx Machine
     |
     | 6PN
     v
Fly WireGuard gateway
     |
     | WireGuard
     v
box
     |
     v
Dokku nginx
     |
     v
Track Anything
     |
     v
SQLite
```

At this point the difficult part of the ingress setup is working.

---

# 6. How multiple Fly Machines work

The home server is **not** connected to either nginx Machine.

Do not think of the topology as:

```text
box --WireGuard--> nginx Machine A
```

Think of it as:

```text
                       +-- nginx Machine A
                       |
box --WireGuard--> 6PN +-- nginx Machine B
                       |
                       +-- other Fly Machines/apps
```

The WireGuard endpoint is Fly gateway infrastructure. The home server becomes a peer on the organization's private Fly network.

Both nginx Machines can independently connect to:

```text
fdaa:6:5dee:a7b:910f:53be:b775:e002
```

Therefore Fly can route one request through Machine A and another through Machine B. If Machine A is replaced, the replacement can also reach the same home server address.

No WireGuard reconfiguration on `box` is required merely because an application Machine is stopped, destroyed, replaced, or recreated.

---

# 7. Hosting multiple homelab projects

This design is intentionally reusable.

The desired long-term topology is:

```text
trackanything.io -----+
                      |
djmagic.io -----------+----> Cloudflare
                      |          |
future-project.com ---+          v
                              Fly Proxy
                                  |
                         nginx Machine(s)
                                  |
                             6PN/WireGuard
                                  |
                                  v
                                 box
                                  |
                             Dokku nginx
                              /       \
                             /         \
                    Track Anything    DJ Magic
```

The same Fly proxy Machines and the same WireGuard tunnel can serve multiple applications.

For each application:

1. Point/configure its public domain through the shared Fly ingress.
2. Make sure Fly has the appropriate domain/certificate configuration.
3. Configure that hostname for the corresponding Dokku app.
4. Preserve the original `Host` header through nginx.

Conceptually, Dokku configuration will look like:

```bash
dokku domains:set trackanything trackanything.io
dokku domains:set djmagic djmagic.io
```

Then requests arrive as:

```text
Host: trackanything.io
```

or:

```text
Host: djmagic.io
```

and Dokku routes them to the correct container.

There is no need for one WireGuard tunnel per application.

There is also no architectural requirement for one Fly ingress app per homelab application. A shared general-purpose ingress app is a natural fit for this design.

---

# 8. Real-domain rollout

Once the `*.fly.dev` health check works, add a real application domain in this order:

1. Configure the hostname on the appropriate Dokku app.
2. Configure the hostname/certificate on the Fly ingress app.
3. Configure Cloudflare DNS to send that hostname to Fly.
4. Verify `https://<domain>/healthz`.
5. Verify the actual application in a browser.
6. Configure trusted client-IP handling as appropriate for the final Cloudflare/Fly proxy chain.

For Track Anything, the target is:

```text
https://trackanything.io
```

Do not remove the working `*.fly.dev` path until the real-domain path has been verified.

---

# 9. Boot/recovery behavior

The home side should recover without manual intervention after a reboot.

Relevant services include:

```text
Docker
Dokku/nginx/app containers
wg-quick@fly
Litestream
```

WireGuard is enabled with:

```bash
sudo systemctl enable wg-quick@fly
```

Litestream should likewise be enabled independently as a systemd service.

After the entire public setup is complete, perform an intentional reboot of `box` and verify all of the following:

```bash
systemctl is-active docker
systemctl is-active wg-quick@fly
systemctl is-active litestream
sudo wg show
sudo dokku ps:report trackanything
```

Then, from outside `box`, verify:

```bash
curl -v https://trackanything.io/healthz
```

The goal is that a normal power loss/reboot requires no SSH session and no manual tunnel/app restart.

---

# 10. Security / repository hygiene

This repository can safely contain:

- `Dockerfile`
- `nginx.conf`
- `fly.toml`
- documentation

It should **not** contain:

- WireGuard private keys
- `/etc/wireguard/fly.conf`
- Fly access tokens
- Cloudflare API tokens
- Tigris credentials
- SMTP credentials
- application secrets

A useful `.gitignore` is:

```gitignore
*.conf.private
*.key
*.pem
.env
.env.*
wireguard/
secrets/
```

Do not rely on `.gitignore` as permission to keep real credentials inside the repository directory. Prefer keeping the actual WireGuard configuration only under `/etc/wireguard/` on `box` and in an appropriate secrets/password backup system.

The `fdaa:` private address used by the proxy is not a secret in the same sense as a private key. The WireGuard private key is.

---

# 11. Useful diagnostics

## Home WireGuard status

```bash
sudo wg show
```

Look for:

```text
latest handshake: ...
transfer: ... received, ... sent
```

## WireGuard interface

```bash
ip addr show fly
```

## Restart tunnel

```bash
sudo systemctl restart wg-quick@fly
```

Then:

```bash
sudo wg show
```

## Fly status

```bash
fly status
```

## Fly logs

```bash
fly logs
```

## Test application directly from box through its Fly address

```bash
curl -v \
  -H 'Host: trackanything.box' \
  http://[fdaa:6:5dee:a7b:910f:53be:b775:e002]:8080/healthz
```

## Test public Fly ingress

```bash
curl -v https://trackanything.fly.dev/healthz
```

## Dokku app status

```bash
sudo dokku ps:report trackanything
```

## Application logs

```bash
sudo dokku logs trackanything -t
```

---

# 12. Failure model

## A Fly nginx Machine dies

Expected behavior: another running/started ingress Machine can proxy to `box`. The home WireGuard configuration does not change.

The proxy Machine is stateless and replaceable.

## Both Fly nginx Machines stop

Fly Proxy can autostart eligible Machines according to the app configuration. `min_machines_running = 1` is intended to keep one primary-region Machine warm.

## A Fly nginx Machine is destroyed

Recreate the desired pool size:

```bash
fly scale count 2
```

No application data is lost.

## Home LAN DHCP address changes

The public architecture should continue to work because the home server initiates WireGuard outward and the Fly proxy targets the server's Fly private address rather than its `192.168.1.x` address.

## Home server reboots

`wg-quick@fly`, Docker/Dokku, and Litestream should be enabled at boot. Verify this with the intentional reboot acceptance test described above.

## Home Internet/power goes down

Fly ingress may remain alive, but its end-to-end `/healthz` check will fail because the backend cannot be reached. Once the home server/network returns and WireGuard reconnects, the path should recover without changing the Fly Machines.

---

# 13. Current known-good checkpoint

At the time this document was created:

- The home server's WireGuard tunnel successfully handshakes with Fly.
- The tunnel transfers traffic.
- `wg-quick@fly` is enabled at boot.
- Track Anything responds to `/healthz` through the home server's Fly private address.
- The Fly nginx ingress has deployed successfully in `sjc`.
- The public Fly hostname successfully reaches Track Anything through the complete chain.
- `https://trackanything.fly.dev/healthz` returns HTTP 200 and `ok`.
- The Fly nginx Machines are stateless and can both reach the same home server through 6PN.
- Real-domain/Cloudflare configuration is the next step.

---

# 14. Design principle

The durable parts are:

```text
public domain
      |
shared Fly ingress
      |
Fly organization private network
      |
WireGuard peer
      |
Dokku
```

Individual Fly proxy Machines are intentionally disposable.

Individual homelab applications remain ordinary Dokku applications selected by hostname.

That separation is what makes this setup useful beyond Track Anything: adding another project should mostly be a matter of adding a domain/certificate and a Dokku app, not building another networking stack.
