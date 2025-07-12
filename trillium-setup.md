# 📝 Trilium Notes Deployment with Traefik + Authentik SSO

This setup runs [Trilium Notes](https://github.com/zadam/trilium) behind a Traefik reverse proxy with Single Sign-On (SSO) powered by Authentik. The internal Trilium password prompt is disabled to allow Authentik to handle authentication.

---

## Stack Overview

- **Trilium Notes** — A hierarchical note-taking app with rich editing
- **Traefik** — Reverse proxy with HTTPS and middleware support
- **Authentik** — Identity provider for SSO (OIDC/SAML)
- **Cloudflare** — DNS management and domain resolution

---

## 🛠 Directory Structure

```bash
trilium/
├── docker-compose.yml
├── trilium-data/
│   └── config.ini  # Custom configuration
```

---

## 🚀 Deployment Instructions

### 1. Prerequisites

- Working **Docker + Docker Compose** installation
- **Traefik** running and configured with:
  - ACME resolver (e.g. Let's Encrypt)
  - `authentik@file` or similar forward auth middleware
- Cloudflare DNS record:  
  **A record** → `notes.exeltan.com` → your VPS IP (Proxy = **DNS only**)

---

### 2.  `docker-compose.yml`

```yaml
services:
  trilium:
    image: zadam/trilium:latest
    container_name: trilium
    restart: unless-stopped
    environment:
      - TRILIUM_DATA_DIR=/home/node/trilium-data
    volumes:
      - ./trilium-data:/home/node/trilium-data
    networks:
      - shared_network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.trilium-router.rule=Host(`notes.exeltan.com`)"
      - "traefik.http.routers.trilium-router.entrypoints=websecure"
      - "traefik.http.routers.trilium-router.tls=true"
      - "traefik.http.routers.trilium-router.tls.certresolver=myresolver"
      - "traefik.http.routers.trilium-router.middlewares=authentik@file"
      - "traefik.http.services.trilium-service.loadbalancer.server.port=8080"

volumes:
  trilium-data:

networks:
  shared_network:
    external: true
```

---

### 3.  `trilium-data/config.ini`

This disables Trilium's built-in login, handing auth to Authentik instead:

```ini
[Network]
port=8080

[general]
disableLogin=true
```

> The `[Network]` section with `port` is required to prevent startup errors.

---

### 4. Deploy and Test

```bash
docker compose up -d
```

- Visit `https://notes.exeltan.com`
- You will be redirected to Authentik to log in
- Upon success, you’ll enter Trilium without a password prompt

---

## Maintenance

- **Backups**: Trilium stores everything in `trilium-data/`. Back up this folder regularly.
- **Logs**: See logs via `docker logs trilium`
- **Updates**: Pull the latest image with:
  ```bash
  docker compose pull
  docker compose up -d
  ```

---

## Security Notes

- Ensure Authentik protects access to `notes.exeltan.com`
- Trilium login is **fully disabled** via config — SSO is required
- Keep Cloudflare proxy disabled to allow Let's Encrypt cert generation

---

## Resources

- Trilium Docs: https://github.com/zadam/trilium
- Authentik Docs: https://goauthentik.io/docs/
- Traefik ForwardAuth + Authentik: https://goauthentik.io/docs/providers/proxy/
