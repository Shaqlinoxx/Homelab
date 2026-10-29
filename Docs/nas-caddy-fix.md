# Fixing TrueNAS SCALE HTTPS with Caddy

## 1. Project Overview

Configured Caddy to securely access TrueNAS SCALE 25 through the custom domain `nas.fortuin.online`, using a Let's Encrypt certificate and Cloudflare DNS validation.

### Environment
- **Caddy server:** `192.168.1.19`
- **TrueNAS SCALE:** `192.168.1.21:443`
- **NAS domain:** `nas.fortuin.online`
- **Proxmox domain:** `proxmox.fortuin.online`

## 2. Errors Encountered

### Error 1: Browser displayed "Not secure"

**Cause:** TrueNAS redirected requests to its private IP address instead of keeping the browser on the domain.

The response showed:

```text
HTTP/2 302
location: https://192.168.1.21:443/ui/
```

**Solution:** Configured Caddy to preserve the hostname and rewrite the redirect response.

### Error 2: Cloudflare API token was empty

Caddy reported:

```text
API token '' appears invalid
```

**Cause:** The Caddy service was not receiving the `CLOUDFLARE_API_TOKEN` environment variable correctly.

**Solution:** Checked the Caddy systemd service configuration and corrected the environment variable.

### Error 3: Caddyfile formatting warning

Caddy reported that the configuration was not formatted.

**Solution:** Validated the configuration and used Caddy's formatter where necessary.

## 3. Final Caddy Configuration

```caddyfile
nas.fortuin.online {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }

    reverse_proxy https://192.168.1.21:443 {
        header_up Host {host}

        transport http {
            tls_insecure_skip_verify
        }

        header_down Location https://192.168.1.21:443 https://nas.fortuin.online
    }
}
```

### Important Configuration Changes

- `tls`: Uses Cloudflare DNS validation to obtain a trusted Let's Encrypt certificate.
- `header_up Host {host}`: Preserves the requested hostname when forwarding requests to TrueNAS.
- `header_down Location`: Rewrites redirects containing the NAS IP address to the NAS domain.
- `tls_insecure_skip_verify`: Allows Caddy to connect to the NAS despite its upstream certificate not being trusted.

## 4. Validation and Testing

Validate the Caddyfile:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Reload Caddy after successful validation:

```bash
sudo systemctl reload caddy
```

Test the NAS domain:

```bash
curl -I https://nas.fortuin.online
```

## 5. Final Result

- Successfully accessed TrueNAS through `https://nas.fortuin.online`.
- Confirmed that Caddy presented a valid Let's Encrypt certificate.
- Fixed the redirect that exposed the NAS private IP address.
- Resolved the Cloudflare API token configuration issue.
- Preserved the existing working Proxmox configuration.

## 6. Security Notes

- Never upload Cloudflare API tokens to GitHub.
- Keep a backup of the working Caddyfile.
- Consider configuring a trusted certificate for the NAS upstream to replace `tls_insecure_skip_verify` in the future.
