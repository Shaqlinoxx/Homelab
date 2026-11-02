# TrueNAS Nextcloud Behind Caddy — Setup Guide

This guide documents the Nextcloud and Caddy reverse-proxy setup for this homelab. It explains what each service does, how the hostname and ports fit together, what the Caddyfile contains, and how to test or troubleshoot the configuration.

> **Important:** Keep a backup copy of `/etc/caddy/Caddyfile` before editing it. Do not replace working Proxmox or TrueNAS site blocks just to add Nextcloud. Add or update only the relevant block.

## 1. Homelab layout

| Service | Local address | Hostname used in the browser |
|---|---|---|
| Caddy reverse proxy | `192.168.1.19` | Receives the friendly hostnames on HTTPS port `443` |
| TrueNAS SCALE | `192.168.1.21:443` | `nas.fortuin.online` |
| Nextcloud app on TrueNAS | `192.168.1.21:30027` | `cloud.fortuin.online` |
| Proxmox | `192.168.1.22:8006` | `proxmox.fortuin.online` |
| AdGuard DNS | `192.168.1.18` | Local DNS resolver / rewrite service |

The Nextcloud port `30027` is the port assigned to this particular installation. TrueNAS may use a different port in another installation. Always use the port displayed in the installed Nextcloud app's Network Configuration.

### How a Nextcloud request travels

1. A browser requests `https://cloud.fortuin.online`.
2. Local DNS (AdGuard) resolves `cloud.fortuin.online` to Caddy at `192.168.1.19`.
3. Caddy receives the HTTPS request on port `443` and presents the certificate for `cloud.fortuin.online`.
4. Caddy forwards the request to the Nextcloud app at `192.168.1.21:30027`.
5. Nextcloud responds through Caddy. The browser does not need to know about port `30027`.

**The hostname must resolve to Caddy (`192.168.1.19`), not directly to TrueNAS (`192.168.1.21`).** DNS records do not contain port numbers; Caddy handles the port translation.

## 2. TrueNAS Apps and Nextcloud

### Apps pool

The TrueNAS Apps pool is set to `tank`. TrueNAS manages its hidden `ix-apps` dataset. Do not manually delete or use `ix-apps` as the parent for custom Nextcloud datasets.

### Nextcloud storage layout

For a Host Path storage deployment, the recommended layout is:

```text
MPOOL/
└── appconfig/
   └── nextcloud_database  #PostgreSQL database files   
├── nextcloud_data  # User files and documents
    
```

The `nextcloud` parent dataset is optional; it keeps the three datasets together. The three child datasets are the useful storage separation. If ixVolumes are used instead, TrueNAS creates and manages the app storage under `ix-apps` automatically.

For Host Path storage, follow the permission settings shown by the current TrueNAS Nextcloud wizard. The current TrueNAS guide specifies ACL handling for `html` and `data` and **Automatic Permissions** for `postgres_data`; do not apply the same ACL recipe to all three datasets.

### Nextcloud URL settings for the reverse proxy

In **Apps → Installed → Nextcloud → Edit → Nextcloud Configuration → URLs**, use:

| Setting | Value |
|---|---|
| Protocol | `HTTPS` |
| Host | `cloud.fortuin.online` |
| External Port | `443` |

The Host field is the hostname only: do not include `https://` or a port number.

In **Network Configuration**:

| Setting | Value |
|---|---|
| WebUI Port | `30027` |
| Certificate ID | The certificate selected for the app, if the app is configured to use HTTPS on its own port. In this setup, `truenas_default` was discussed/selected during configuration. |

The WebUI Port and External Port have different jobs:

- `30027` is the local port Caddy connects to on TrueNAS.
- `443` is the normal HTTPS port used by people visiting `cloud.fortuin.online` through Caddy.

The TrueNAS **Web UI** button may still open the direct app address with `:30027`. That is a direct link to the app and is not the correct test of the Caddy hostname. Test `https://cloud.fortuin.online` separately in the browser.

### Trusted Proxies

Nextcloud must trust the reverse proxy so that it can correctly interpret forwarded client information and the original HTTPS request.

Add Caddy's IP address to **Trusted Proxies** if it is not already covered by an existing trusted range:

```text
192.168.1.19
```

Prefer the individual Caddy IP rather than trusting a wider network range unless the wider range is genuinely required. Keep any other entries that are required by the app's own network configuration.

### Certificate note

The certificate configured for the Nextcloud app and the certificate Caddy presents to the browser serve different connections:

- The app certificate applies to the direct TrueNAS-to-Nextcloud/app endpoint on port `30027`.
- Caddy obtains/presents the browser-facing certificate for `cloud.fortuin.online` using the Cloudflare DNS challenge.

These certificates do not conflict simply because both are used in the same setup. Selecting `truenas_default` for the app does **not** replace the certificate used by Caddy for `cloud.fortuin.online`.

## 3. AdGuard DNS rewrite

A wildcard DNS rewrite may already cover `cloud.fortuin.online`. If so, do not create a duplicate rule; verify that the result is `192.168.1.19`.

For local network access, create a DNS rewrite in AdGuard:

| Domain | Rewrite address |
|---|---|
| `cloud.fortuin.online` | `192.168.1.19` |

Do not rewrite `cloud.fortuin.online` directly to `192.168.1.21`, because that bypasses Caddy.

The existing names remain separate:

- `nas.fortuin.online` → Caddy → TrueNAS dashboard (`192.168.1.21:443`)
- `proxmox.fortuin.online` → Caddy → Proxmox (`192.168.1.22:8006`)
- `cloud.fortuin.online` → Caddy → Nextcloud (`192.168.1.21:30027`)


## 4. What is the Caddyfile?

The **Caddyfile** is the text configuration file that tells Caddy what to do with incoming web requests. In this setup, it tells Caddy:

- which hostname a visitor is requesting;
- how to obtain/use HTTPS certificates;
- which internal IP address and port host the real application;
- whether any request or response headers need adjusting.

The Caddyfile used by the Debian Caddy service is:

```text
/etc/caddy/Caddyfile
```

Caddy is the front door for the services. It listens for requests to the domain names and sends each request to the correct internal service. This is why the browser can use normal HTTPS without typing `:8006` or `:30027`.

## 5. Nextcloud Caddy site block

Add this site block to `/etc/caddy/Caddyfile`, leaving your existing site blocks in place:

```caddyfile
cloud.fortuin.online {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    redir /.well-known/carddav /remote.php/dav/ 301
    redir /.well-known/caldav /remote.php/dav/ 301

    reverse_proxy https://192.168.1.21:30027 {
        header_up Host {host}

        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

### What each part means

| Caddyfile line / directive | Explanation |
|---|---|
| `cloud.fortuin.online {` | Starts a site block. This block handles requests whose hostname is `cloud.fortuin.online`. |
| `tls { ... }` | Configures HTTPS for visitors connecting to Caddy. |
| `dns cloudflare {env.CLOUDFLARE_API_TOKEN}` | Uses the Cloudflare DNS provider module and reads the API token from the Caddy service's environment. The token is not written directly into the Caddyfile. Caddy uses DNS validation to obtain/renew the certificate. |
| `redir ...carddav...` | Redirects the standard CardDAV discovery path to Nextcloud's DAV endpoint, for contacts/calendar clients. |
| `redir ...caldav...` | Redirects the standard CalDAV discovery path to Nextcloud's DAV endpoint. |
| `reverse_proxy https://192.168.1.21:30027` | Sends the request to the Nextcloud app on TrueNAS. The `https://` scheme means Caddy uses HTTPS to the app. |
| `header_up Host {host}` | Sends the original requested hostname to the upstream app. This can help the app recognise `cloud.fortuin.online`. |
| `transport http { ... }` | Configures how Caddy communicates with the upstream server. |
| `tls_insecure_skip_verify` | Disables certificate verification for the HTTPS connection from Caddy to the app. This can get a self-signed internal certificate working, but it weakens security because Caddy no longer verifies the app's identity. Prefer trusting the app certificate properly, or use HTTP on the private LAN if the app is configured for HTTP and that is acceptable for your network. Do not confuse this with the browser-facing HTTPS certificate. |
| `}` | Closes the relevant configuration block. |

**Backend protocol check:** The example above assumes the Nextcloud listener on port `30027` is HTTPS, consistent with using an app certificate. If the app's direct endpoint is actually HTTP, change the upstream to `192.168.1.21:30027` (without `https://`) and remove the `transport http` / `tls_insecure_skip_verify` block. The upstream scheme must match what the app is really serving.

## 6. Existing Caddy sites

The following are examples based on the homelab's known upstream addresses. Keep the exact working versions already in your Caddyfile, especially the TrueNAS redirect rewrite that was needed to stop TrueNAS sending the browser back to its IP address.

### Proxmox

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
```

### TrueNAS dashboard

The important parts of the previously working TrueNAS proxy configuration were the original Host header and the `Location` response-header rewrite. This prevents the TrueNAS interface from redirecting the browser to `https://192.168.1.21:443` instead of staying on `nas.fortuin.online`.

```caddyfile
nas.fortuin.online {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    reverse_proxy https://192.168.1.21:443 {
        header_up Host {host}
        header_down Location https://192.168.1.21:443 https://nas.fortuin.online

        transport http {
            tls_insecure_skip_verify
        }
    }
}
```

These examples are not a replacement for checking the current working file. Do not duplicate a hostname in two different site blocks.

## 7. Cloudflare token and Caddy service

The Cloudflare API token is supplied to Caddy as an environment variable:

```text
CLOUDFLARE_API_TOKEN
```

The Caddyfile references it using:

```caddyfile
dns cloudflare {env.CLOUDFLARE_API_TOKEN}
```

The token should not be pasted into the Caddyfile, a GitHub repository, screenshots, or logs. It is supplied through the Caddy systemd service environment/drop-in. The installed Caddy binary must include the Cloudflare DNS provider module; a standard Caddy build without that module will not recognise `dns cloudflare`.

## 8. Validate and reload Caddy

After editing the Caddyfile, validate it before reloading:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

If validation succeeds, reload Caddy:

```bash
sudo systemctl reload caddy
```

Check the service status:

```bash
sudo systemctl status caddy
```

View recent logs if something fails:

```bash
sudo journalctl -u caddy -n 50 --no-pager
```

Do not reload Caddy if validation reports an error. Correct the syntax first.

## 9. Testing checklist

1. In AdGuard, confirm `cloud.fortuin.online` resolves to `192.168.1.19` on the local network.
2. Confirm the Nextcloud app is Running in TrueNAS.
3. Confirm the Nextcloud WebUI Port is `30027`.
4. Confirm the Nextcloud URL Host is `cloud.fortuin.online`, Protocol is HTTPS, and External Port is `443`.
5. Confirm `192.168.1.19` is listed as a trusted proxy in Nextcloud.
6. Confirm the Caddyfile validates and Caddy reloads successfully.
7. Open `https://cloud.fortuin.online` manually in a browser. Do not use the TrueNAS WebUI button as the only proxy test, because that button may open the direct app port.
8. Log in and confirm the Files page loads.
9. Re-test `https://nas.fortuin.online` and `https://proxmox.fortuin.online` to ensure the existing sites still work.

## 10. Troubleshooting notes

### TrueNAS WebUI button opens `cloud.fortuin.online:30027`

That button may be constructing the direct app URL using the WebUI Port. It does not by itself prove that Caddy is broken. Test `https://cloud.fortuin.online` without a port in a separate browser tab.

### Nextcloud redirects to `nas.fortuin.online` or adds `:30027`

Check the Nextcloud **URLs** fields:

- Host must be `cloud.fortuin.online` (hostname only).
- Protocol must be `HTTPS` when accessed through Caddy.
- External Port should be `443` when Caddy is the reverse proxy.
- WebUI Port stays `30027` for Caddy's internal upstream connection.

Also check that the request is reaching the correct Caddy site block and that Nextcloud trusts the `cloud.fortuin.online` domain.

### Browser cannot find the hostname

Check the AdGuard rewrite. It should resolve `cloud.fortuin.online` to `192.168.1.19`, not `192.168.1.21`.

### Caddy reports an upstream connection error

Check that Nextcloud is Running, that the port is still `30027`, and that the upstream protocol in Caddy (`http://` or `https://`) matches the app listener.

### Nextcloud reports an untrusted domain

Check that `cloud.fortuin.online` is the Host in the Nextcloud URL configuration and appears in Trusted Domains. The TrueNAS app normally adds the configured Host to the trusted-domain list.

### TrueNAS dashboard redirects to its IP address

Keep the `header_up Host {host}` and `header_down Location ...` adjustments in the TrueNAS site block. Do not put those rules in the Nextcloud block unless Nextcloud itself is returning an incorrect redirect that needs a specific fix.

## 11. Security and access notes

- Use strong, unique Nextcloud administrator credentials and enable two-factor authentication where possible.
- Keep TrueNAS, Nextcloud, Proxmox, and Caddy updated.
- Do not expose the Nextcloud app port `30027` directly to the internet. Browser access should go through Caddy on HTTPS port `443`.
- A Cloudflare DNS challenge obtains a certificate; it does not by itself make the service reachable from outside the LAN.
- This guide's AdGuard rewrite describes local-network access. Remote access requires a separately planned and secured route (for example, VPN or a properly configured tunnel/reverse-proxy ingress). Do not add router port forwards without deciding that external access is intended.
- `tls_insecure_skip_verify` is a workaround for an untrusted/self-signed upstream certificate. It disables verification of that upstream certificate. A better long-term setup is to configure Caddy to trust the upstream certificate, or to use a suitable private HTTP upstream where appropriate.

## 12. Reference documentation

- [TrueNAS — Nextcloud Deployment](https://apps.truenas.com/resources/deploy-nextcloud/)
- [TrueNAS — Securing Apps](https://apps.truenas.com/getting-started/securing-apps/)
- [Caddy — reverse_proxy directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [Caddy — Automatic HTTPS](https://caddyserver.com/docs/automatic-https)

---

**Last updated:** 2 November 2026  
**Environment:** TrueNAS SCALE 25.10, Debian Caddy host, Caddy v2.11.7 (as previously reported).  
**Note:** IP addresses, domains, and port numbers in this guide are specific to this homelab. Re-check them before reusing the guide on another system.
