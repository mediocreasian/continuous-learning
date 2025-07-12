# 🛠️ Keycloak Demo Deployment — DevOps Debug Log

## ✅ Objective
Deploy a Spring Boot app (secured with Keycloak) behind Traefik using GitHub Actions CI/CD, GitHub Container Registry (GHCR), and Cloudflare DNS, hosted on a NetCup VPS.

---

## 🚀 Deployment Pipeline Setup

### ✅ 1. CI/CD Workflow: Push to GHCR

**Tool:** GitHub Actions  
**Command:**  
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=ghcr.io/<org>/<image>
```

- Switched from Dockerfile to `spring-boot:build-image` (Paketo Buildpacks).
- Added `chmod +x ./mvnw` step to fix executable issue.
- Fixed missing `.mvn/wrapper/maven-wrapper.jar` due to `.gitignore` mistake.

**Final Action:**
- Builds Spring Boot image with `spring-boot:build-image`
- Pushes `:latest` and `:<sha>` to GHCR
- Sends Discord notifications for success/failure

---

### ✅ 2. Deployment to VPS via SSH + Docker Compose

**Tool:** Reusable GitHub workflow  
**Script:**
- SSH into server
- Authenticate to GHCR
- Pull image
- Run `docker compose up -d` using path passed from workflow

**Key Inputs:**
```yaml
with:
  IMAGE_NAME: "keycloak-demo"
  COMPOSE_PATH: "/home/exeltan/keycloak-demo/docker-compose.yml"
```

---

## 🐳 Containerization Setup

### ✅ Application Runtime

Used `spring-boot:build-image`, avoiding Dockerfile entirely.

**Ports:**
- App listens on `SERVER_PORT=8001`
- Set via env var + `server.port=${SERVER_PORT:8080}` in `application.properties`

---

## 🌐 Domain + Traefik Routing

### ✅ Domain: `demo.exeltan.com`

**DNS:**  
- Configured A record in Cloudflare pointing to VPS IP.
- Ensured **DNS-only (grey cloud)** to allow Let's Encrypt HTTP challenge.

---

### ✅ Traefik Setup

- Used static config (`traefik.yml`) with `web` and `websecure` entrypoints.
- Enabled Let’s Encrypt via `certificatesResolvers.letsencrypt.acme`.
- TLS via HTTP-01 challenge on port `80`.

**Key Labels on App:**
```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.keycloak-demo.rule=Host(`demo.exeltan.com`)
  - traefik.http.routers.keycloak-demo.entrypoints=websecure
  - traefik.http.routers.keycloak-demo.tls=true
  - traefik.http.routers.keycloak-demo.tls.certresolver=letsencrypt
  - traefik.http.services.keycloak-demo.loadbalancer.server.port=8001
```

---

## 🧨 Issues Encountered + Resolutions

### ❌ Maven Wrapper Failure (`Permission denied`)

- **Fix:** Added `chmod +x ./mvnw` to workflow.

---

### ❌ Maven Wrapper Missing `.jar` File

- **Cause:** `.gitignore` excluded `.mvn/wrapper/maven-wrapper.jar`
- **Fix:** Unignored + committed wrapper files

---

### ❌ Cannot resolve host: `demo.exeltan.com`

- **Cause:** NetCup’s internal DNS (`46.38.x.x`) failed to resolve newly created Cloudflare subdomain.
- **Workaround:**
  - Used `--resolve` in curl
  - Verified DNS via `dig demo.exeltan.com @1.1.1.1`
  - Manually edited `/etc/resolv.conf` temporarily (not persistent)

---

### ❌ Self-signed TLS Certificate

- **Cause:** Traefik used fallback cert because Let's Encrypt didn't issue a valid one.
- **Fix:**
  - Verified Traefik was exposed on ports `80` and `443`
  - Ensured Cloudflare was set to DNS-only
  - Used proper ACME config and `tls.certresolver=letsencrypt` label

---

## 📌 Next Steps (Post-NetCup Issue)

- Monitor DNS once NetCup resolves the upstream resolver issue
- Revert any temporary `--resolve` or `/etc/hosts` changes
- Optionally switch to DNS-01 challenge with Cloudflare proxy + API token later