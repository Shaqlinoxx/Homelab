# Caddy Reverse Proxy Homelab

A concise guide to the homelab reverse proxy running on **Debian 13** with **Caddy**, **Cloudflare DNS-01 validation**, and **AdGuard Home**. It provides HTTPS access to internal services through memorable hostnames without exposing their management ports directly to the internet.

## Network overview

| Component | Address / purpose |
|---|---|
| Caddy server | `192.168.1.19` — HTTPS reverse proxy on port `443` |
| Proxmox | `192.168.1.22:8006` |
| TrueNAS / NAS | `192.168.1.21` |
| AdGuard Home | `192.168.1.18` — local DNS rewrites |
| Domain | `fortuin.online` |
| Platform | Debian 13 · Caddy v2.11.7 |

## How it works

1. A device opens a hostname such as `https://proxmox.fortuin.online`.
2. AdGuard Home resolves that hostname to the Caddy server (`192.168.1.19`).
3. Caddy presents the HTTPS certificate and forwards the request to the matching internal service.
4. Caddy uses Cloudflare DNS-01 validation to issue and renew certificates. This validation does not require forwarding ports 80 or 443 from the router.

## Caddyfile

Edit the configuration:

```bash
sudo nano /etc/caddy/Caddyfile
```

Use the following site blocks. The NAS target is `192.168.1.21`; this assumes its web interface is listening on HTTP port `80`. If your TrueNAS web UI uses a different port or HTTPS, change the upstream accordingly.

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

### What the directives do

- `tls { dns cloudflare ... }` uses the Cloudflare DNS provider for DNS-01 certificate validation.
- `{env.CLOUDFLARE_API_TOKEN}` reads the API token from Caddy's service environment.
- `reverse_proxy` forwards requests to the specified internal service.
- `tls_insecure_skip_verify` disables certificate verification **only on Caddy's connection to the Proxmox backend**. It is used here because Proxmox commonly presents its own certificate; consider configuring backend certificate trust properly later.

## Cloudflare token and systemd

Create the environment file:

```bash
sudo nano /etc/caddy/cloudflare.env
```

Add your token privately (never commit the real token to GitHub):

```dotenv
CLOUDFLARE_API_TOKEN=YOUR_ACTUAL_TOKEN
```

Restrict access and configure the systemd override:

```bash
sudo chown root:root /etc/caddy/cloudflare.env
sudo chmod 600 /etc/caddy/cloudflare.env
sudo mkdir -p /etc/systemd/system/caddy.service.d
sudo nano /etc/systemd/system/caddy.service.d/override.conf
```

Override contents:

```ini
[Service]
EnvironmentFile=/etc/caddy/cloudflare.env
```

Then reload systemd after saving the override.

```bash
sudo systemctl daemon-reload
```

The Cloudflare token should have **Zone → Zone → Read** and **Zone → DNS → Edit** permissions scoped to the `fortuin.online` zone.

## DNS setup

### Cloudflare
Keep the domain managed by Cloudflare for DNS-01 validation. Do not publish the API token in this repository.

### AdGuard Home
Create local DNS rewrites so client devices resolve each hostname to the **Caddy server**, not to the upstream service:

| Hostname | Local DNS rewrite |
|---|---|
| `proxmox.fortuin.online` | `192.168.1.19` |
| `nas.fortuin.online` | `192.168.1.19` |

Devices must use AdGuard Home (`192.168.1.18`) as their DNS resolver to receive these local rewrites.

## Validate and restart

After editing the Caddyfile:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl daemon-reload
sudo systemctl restart caddy
sudo systemctl enable caddy
sudo systemctl status caddy
```

Inspect recent or live logs if needed:

```bash
sudo journalctl -u caddy -n 100 --no-pager
sudo journalctl -u caddy -f
```

Test name resolution from a Windows client:

```powershell
ipconfig /flushdns
nslookup proxmox.fortuin.online
nslookup nas.fortuin.online
```

Both names should resolve to `192.168.1.19` when the client uses AdGuard Home. Then test the sites:

```bash
curl -I https://proxmox.fortuin.online
curl -I https://nas.fortuin.online
```

A valid HTTP response (often `200` or `302`) indicates that the request reached a web service; it does not by itself prove every application feature works.

## Troubleshooting

| Symptom | Check |
|---|---|
| Caddy will not start | Run `sudo caddy validate --config /etc/caddy/Caddyfile`, then inspect `journalctl -u caddy`. |
| Cloudflare token error | Confirm the token value, permissions, zone scope, and systemd environment file. |
| Certificate issuance fails | Verify `dns.providers.cloudflare` is present in the active Caddy binary. |
| Works on one device only | Check that the device uses AdGuard, resolves the hostname to `192.168.1.19`, and can reach port `443`. |
| NAS page does not load | Confirm the TrueNAS web UI port/protocol and adjust the `nas.fortuin.online` upstream. |
| Proxmox backend fails | Check connectivity from Caddy to `192.168.1.22:8006`. |

## Security notes

- Never commit `CLOUDFLARE_API_TOKEN` or any other secret to GitHub.
- Keep Proxmox management port `8006` off the public internet.
- A trusted HTTPS certificate does not make a service private; local DNS and network/firewall controls still matter.
- The Proxmox upstream currently skips backend certificate verification. Use this only on a trusted network and consider a properly trusted backend certificate.
- Confirm the TrueNAS web UI's actual listening port and protocol before relying on the NAS proxy block.

## Completion checklist

- [ ] Caddy is running on Debian 13 at `192.168.1.19`.
- [ ] The active Caddy binary includes `dns.providers.cloudflare`.
- [ ] Cloudflare token is stored securely in `/etc/caddy/cloudflare.env`.
- [ ] Both Caddy site blocks validate and Caddy restarts successfully.
- [ ] AdGuard rewrites both hostnames to `192.168.1.19`.
- [ ] Both hostnames work from more than one client device.
