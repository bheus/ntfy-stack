# ntfy-stack

Self-hosted [ntfy](https://ntfy.sh) push-notification server, deployed on apple-pi via Portainer Git stack.

- **Public URL**: https://ntfy.builtbybrendan.com (Cloudflare Tunnel `brendan-website` → `http://localhost:2586`)
- **Host port**: `2586` → container `:80`
- **Auth**: `deny-all` by default; admin user created post-deploy
- **Web UI**: enabled, login required

## Deploy (Portainer)

1. Add this repo as a Portainer Git stack.
2. Set environment variable in the stack config:
   - `NTFY_CONFIG_PATH=/data/compose/<N>/ntfy/server.yml` (Portainer substitutes `<N>` with the stack ID; or use an absolute host path if you cloned the repo elsewhere).
3. Deploy.

## Post-deploy: bootstrap admin user

```bash
docker exec -it ntfy ntfy user add --role=admin admin
# (you'll be prompted for password)
```

Per-app users with topic ACLs:

```bash
docker exec -it ntfy ntfy user add homelab
docker exec ntfy ntfy access homelab "homelab-alerts" rw
```

## Cloudflare Tunnel

Add to `/etc/cloudflared/config.yml` on apple-pi, **before** the catch-all `http_status:404`:

```yaml
  - hostname: ntfy.builtbybrendan.com
    service: http://localhost:2586
    originRequest:
      httpHostHeader: ntfy.builtbybrendan.com
```

Then route DNS and restart the tunnel:

```bash
docker exec cloudflared-tunnel cloudflared tunnel route dns brendan-website ntfy.builtbybrendan.com
docker restart cloudflared-tunnel   # or redeploy cloudflare-stack in Portainer
```

## Publishing

```bash
# With auth
curl -u homelab:PASSWORD -d "backup complete" https://ntfy.builtbybrendan.com/homelab-alerts

# Subscribe (CLI)
ntfy subscribe -u homelab:PASSWORD https://ntfy.builtbybrendan.com/homelab-alerts
```

## Notes

- `ntfy-cache` named volume holds `user.db`, `cache.db`, and attachments.
- `behind-proxy: true` makes rate limiting use `X-Forwarded-For` from Cloudflare.
- Image pinned to `v2.22.0`; verify and bump deliberately.
