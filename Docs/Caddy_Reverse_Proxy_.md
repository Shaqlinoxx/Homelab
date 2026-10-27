# Caddy Reverse Proxy Setup Guide

**Secure access to Proxmox and TrueNAS using Debian 13, Caddy, Cloudflare DNS, and AdGuard Home**

This step-by-step guide follows the order of the supplied setup document and adds `nas.fortuin.online` for the NAS at `192.168.1.21`.

## Network details

| Component | Address / role |
|---|---|
| Caddy server | `192.168.1.19` |
| Proxmox web UI | `192.168.1.22:8006` |
| TrueNAS / NAS | `192.168.1.21` |
| AdGuard Home DNS | `192.168.1.18` |
| Proxmox hostname | `proxmox.fortuin.online` |
| NAS hostname | `nas.fortuin.online` |
| Platform | Debian 13 · Caddy v2.11.7 |

## Project overview

A client device connects to Caddy over HTTPS. Caddy forwards the request to the relevant internal service. Cloudflare is used for DNS-01 certificate validation, while AdGuard Home provides local DNS rewrites so home-network devices resolve both hostnames to Caddy.

---

## 1. Understand the network layout

1. A device opens `https://proxmox.fortuin.online` or `https://nas.fortuin.online`.
2. AdGuard Home resolves the hostname to Caddy at `192.168.1.19`.
3. Caddy handles HTTPS and forwards the request to the correct internal service.
4. Cloudflare DNS-01 validation allows Caddy to obtain and renew certificates without router port-forwarding for ports 80 or 443.

| Component | Address / role |
|---|---|
| Client device | Opens one of the HTTPS hostnames |
| Cloudflare | DNS-01 certificate validation |
| Caddy / Debian 13 | `192.168.1.19` · HTTPS port `443` |
| Proxmox | `192.168.1.22:8006` |
| TrueNAS / NAS | `192.168.1.21` · confirm web UI port |
| AdGuard Home | `192.168.1.18` · local DNS rewrites |

## 2. Prepare the Debian server

Connect to the Debian server using its local terminal or SSH.

### 2.1 Check the OS and IP address

```bash
cat /etc/os-release
ip addr
```

Confirm Debian 13 and verify that the Caddy server uses `192.168.1.19`. Keep this address stable using a router DHCP reservation or suitable static configuration.

### 2.2 Update Debian

```bash
sudo apt update
sudo apt upgrade -y
```

## 3. Build Caddy with Cloudflare DNS support

The standard Caddy binary may not include the third-party Cloudflare DNS provider module. A custom build is required for DNS-01 certificate validation.

### 3.1 Install build requirements

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl gnupg golang git
```

### 3.2 Install xcaddy

```bash
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
export PATH="$PATH:$(go env GOPATH)/bin"
xcaddy version
```

### 3.3 Build and verify the module

```bash
xcaddy build v2.11.7 --with github.com/caddy-dns/cloudflare
./caddy list-modules | grep cloudflare
```

Expected module: `dns.providers.cloudflare`.

### 3.4 Install the custom binary

If Caddy is already installed, back up the existing binary first. Run these commands from the directory containing the new `./caddy` binary.

```bash
sudo cp /usr/bin/caddy /usr/bin/caddy.backup
sudo install -m 755 ./caddy /usr/bin/caddy
caddy version
caddy list-modules | grep cloudflare
```

If the new binary is elsewhere, replace `./caddy` with its actual path. If backup or installation fails, stop and resolve the issue before proceeding.

## 4. Create a Cloudflare API token

1. Sign in at [Cloudflare](https://dash.cloudflare.com/).
2. Open **My Profile → API Tokens**.
3. Choose **Create Token** and configure a custom token.
4. Grant **Zone → Zone → Read** and **Zone → DNS → Edit**.
5. Under **Zone Resources**, select **Include → Specific Zone → `fortuin.online`**.
6. Create the token and store it securely.

Do not publish the token or commit it to GitHub. These permissions let the DNS provider create and remove records required for validation.

## 5. Store the token securely

### 5.1 Create the environment file

```bash
sudo nano /etc/caddy/cloudflare.env
```

Add the line below, replacing the placeholder with the real token:

```dotenv
CLOUDFLARE_API_TOKEN=YOUR_ACTUAL_TOKEN
```

Save in Nano with **Ctrl+O**, press **Enter**, then exit with **Ctrl+X**.

### 5.2 Restrict access

```bash
sudo chown root:root /etc/caddy/cloudflare.env
sudo chmod 600 /etc/caddy/cloudflare.env
```

### 5.3 Configure the systemd override

```bash
sudo mkdir -p /etc/systemd/system/caddy.service.d
sudo nano /etc/systemd/system/caddy.service.d/override.conf
```

Add:

```ini
[Service]
EnvironmentFile=/etc/caddy/cloudflare.env
```

Save the file. An empty override file will not load the token. Reload systemd:

```bash
sudo systemctl daemon-reload
```

## 6. Configure the Caddyfile

### 6.1 Open the configuration

```bash
sudo nano /etc/caddy/Caddyfile
```

### 6.2 Add both site blocks

The NAS upstream below assumes that the TrueNAS web UI listens on HTTP port `80`. Verify the actual TrueNAS web UI port and protocol; change the upstream if necessary.

```caddyfile
proxmox.fortuin.online {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    reverse_proxy https://192.168.1.22:8006 {
        transport http {
            tls_insecure_skip_verify
        }
    }
}

nas.fortuin.online {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    reverse_proxy 192.168.1.21:80
}
```

Save the file.

### 6.3 Understand the directives

- `proxmox.fortuin.online` and `nas.fortuin.online` are the hostnames entered in a browser.
- `tls` with `dns cloudflare` obtains and renews certificates using Cloudflare DNS validation.
- `{env.CLOUDFLARE_API_TOKEN}` reads the token from the service environment.
- `reverse_proxy` forwards requests to the configured internal service.
- `tls_insecure_skip_verify` disables certificate verification only for Caddy's connection to the Proxmox backend.

**Security note:** The Proxmox backend setting skips verification of the Proxmox server certificate. Use it only on a trusted network and consider configuring backend certificate trust properly later.

## 7. Configure Cloudflare DNS and local DNS

### 7.1 Check Cloudflare DNS

In Cloudflare, check the DNS records for `fortuin.online`. DNS-01 validation does not require router port-forwarding for ports 80 or 443. Avoid exposing the Proxmox management interface directly to the internet.

### 7.2 Add AdGuard Home DNS rewrites

Open AdGuard Home at its configured address and find **DNS rewrites**. Add these records:

| Domain | IP address |
|---|---|
| `proxmox.fortuin.online` | `192.168.1.19` |
| `nas.fortuin.online` | `192.168.1.19` |

Both hostnames must resolve to the Caddy server, not directly to Proxmox, TrueNAS, or AdGuard.

### 7.3 Ensure client devices use AdGuard

1. Check the router DHCP settings to see which DNS server is provided to clients.
2. Where appropriate, configure home devices to use `192.168.1.18` as their DNS resolver.
3. Devices using another resolver may not receive the local rewrites.
4. Clear DNS caches after changes.

On Windows, run:

```powershell
ipconfig /flushdns
nslookup proxmox.fortuin.online
nslookup nas.fortuin.online
```

Both names should resolve to `192.168.1.19` when the device is querying AdGuard Home.

## 8. Validate and restart Caddy

### 8.1 Validate the configuration

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Fix validation errors before restarting.

### 8.2 Reload systemd and restart

```bash
sudo systemctl daemon-reload
sudo systemctl restart caddy
sudo systemctl enable caddy
```

### 8.3 Check status and logs

```bash
sudo systemctl status caddy
sudo journalctl -u caddy -n 100 --no-pager
```

To follow live logs while testing:

```bash
sudo journalctl -u caddy -f
```

## 9. Test the project

| Test | Command / URL | Expected result |
|---|---|---|
| Caddy service | `sudo systemctl status caddy` | `Active (running)` |
| Cloudflare module | `caddy list-modules \| grep cloudflare` | `dns.providers.cloudflare` |
| Proxmox DNS | `nslookup proxmox.fortuin.online` | `192.168.1.19` |
| NAS DNS | `nslookup nas.fortuin.online` | `192.168.1.19` |
| Proxmox HTTPS | `curl -I https://proxmox.fortuin.online` | Valid HTTP response, often `200` or `302` |
| NAS HTTPS | `curl -I https://nas.fortuin.online` | Valid HTTP response, often `200` or `302` |
| Proxmox browser test | `https://proxmox.fortuin.online` | Proxmox login page |
| NAS browser test | `https://nas.fortuin.online` | TrueNAS web UI, if upstream port/protocol is correct |

A valid HTTP response means a web service responded; it does not by itself prove that every application feature works.

## 10. Troubleshooting

| Problem | What to check |
|---|---|
| Cloudflare token invalid | Check token value, expiry, permissions, zone scope, and systemd environment file. |
| Certificate issuance fails | Confirm the active Caddy binary includes `dns.providers.cloudflare` and DNS-01 validation can complete. |
| Caddy fails to start | Run `sudo caddy validate --config /etc/caddy/Caddyfile` and inspect `journalctl -u caddy`. |
| Works on laptop only | Check each device's DNS resolver, AdGuard rewrites, and connectivity to `192.168.1.19:443`. |
| Browser certificate warning | Confirm the exact hostname and that Caddy has obtained a trusted certificate. |
| Proxmox backend fails | Check connectivity from Caddy to `192.168.1.22:8006`. |
| NAS page fails | Confirm the TrueNAS web UI port/protocol and update the NAS `reverse_proxy` target. |

## 11. Security considerations

- Keep the Cloudflare API token private and restrict its file permissions.
- Never commit the token or `/etc/caddy/cloudflare.env` to GitHub.
- Do not expose Proxmox port `8006` directly to the internet.
- A trusted HTTPS certificate does not, by itself, make a service private.
- AdGuard's local rewrite directs home devices to Caddy; network and firewall controls remain important.
- The Proxmox upstream skips backend certificate verification; consider configuring backend trust properly.
- Confirm the NAS web UI's actual port and protocol before relying on its proxy entry.

## 12. Final completion checklist

- [ ] Debian 13 installed and reachable.
- [ ] Caddy running at `192.168.1.19`.
- [ ] Active Caddy binary includes `dns.providers.cloudflare`.
- [ ] Cloudflare token created with restricted permissions.
- [ ] Token stored in `/etc/caddy/cloudflare.env` with restricted permissions.
- [ ] systemd configured to load the token.
- [ ] Caddyfile routes `proxmox.fortuin.online` to `192.168.1.22:8006`.
- [ ] Caddyfile routes `nas.fortuin.online` to `192.168.1.21` on the verified NAS web UI port.
- [ ] AdGuard rewrites both hostnames to `192.168.1.19`.
- [ ] HTTPS certificates issued and Caddy service running.
- [ ] Both hostnames tested successfully.
- [ ] Other devices tested and DNS resolution verified.

## Project status / next step

Test both hostnames from more than one device. If a hostname works on one device but not another, verify that the device uses AdGuard Home and resolves the hostname to `192.168.1.19`, then check access to port `443`.
