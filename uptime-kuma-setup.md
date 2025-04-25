# Uptime Kuma + Traefik + Authentik Setup Documentation

---

## VPS Setup Overview
- **OS**: Ubuntu VPS
- **Reverse Proxy**: Traefik v2.10 (Dockerized)
- **Authentication Layer**: Authentik (SSO via ForwardAuth)
- **Monitoring App**: Uptime Kuma (Dockerized)
- **Container Network**: `shared_network` (external)

---

## 1. Traefik Setup

### Static Config (`traefik/config/traefik.yml`):
```yaml
providers:
  docker:
    exposedByDefault: false
  file:
    directory: "/etc/traefik/conf/dynamic"
    watch: true

api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"

certificatesResolvers:
  myresolver:
    acme:
      email: exelbert2010@gmail.com
      storage: /letsencrypt/acme.json
      tlsChallenge: true

metrics:
  prometheus:
    addEntryPointsLabels: true
    addRoutersLabels: true
    addServicesLabels: true
```

### Docker Compose (`traefik/docker-compose.yml`):
```yaml
services:
  traefik:
    container_name: traefik
    image: traefik:v2.10
    command:
      - "--configFile=/etc/traefik/conf/traefik.yml"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.exeltan.com`)"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=myresolver"
      - "traefik.http.routers.traefik.middlewares=authentik@file"
    ports:
      - "80:80"
      - "443:443"
      - "8082:8082" # Prometheus metrics
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"
      - "./config:/etc/traefik/conf"
    networks:
      - shared_network

networks:
  shared_network:
    external: true
```

###  Dynamic Config:
#### `config/dynamic/routes.yml`
```yaml
http:
  routers:
    authentik:
      rule: "Host(`authentik.exeltan.com`)"
      entryPoints:
        - websecure
      service: authentik
      tls:
        certResolver: myresolver

  services:
    authentik:
      loadBalancer:
        servers:
          - url: "http://authentik:9000"
```

#### `config/dynamic/headers.yml`
```yaml
http:
  middlewares:
    authentik:
      forwardAuth:
        address: http://authentik:9000/outpost.goauthentik.io/auth/traefik
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-groups
          - X-authentik-entitlements
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
          - X-authentik-jwt
          - X-authentik-meta-jwks
          - X-authentik-meta-outpost
          - X-authentik-meta-provider
          - X-authentik-meta-app
          - X-authentik-meta-version
```

---

## 2. Authentik Setup

- Applications → Create Traefik & Kuma applications
- Providers → Create a ForwardAuth provider
- Outposts → Register your Traefik outpost using worker:

```bash
docker exec -it authentik-worker ak outpost discover --token <REGISTRATION_TOKEN>
```

- Ensure each Application is mapped to a Provider and has the correct **redirect URI**.

---

## 3. Uptime Kuma Setup

### Docker Compose:
```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.kuma.rule=Host(`status.exeltan.com`)"
      - "traefik.http.routers.kuma.entrypoints=websecure"
      - "traefik.http.routers.kuma.tls.certresolver=myresolver"
      - "traefik.http.routers.kuma.service=kuma"
      - "traefik.http.routers.kuma.middlewares=authentik@file"
      - "traefik.http.services.kuma.loadbalancer.server.port=3001"
    networks:
      - shared_network

networks:
  shared_network:
    external: true
```

### Monitoring Internal Docker Services:
- Make sure all services you want to monitor share the `shared_network`.
- Add monitors using internal service names:
  - Example: `http://spring-api:8080/health`
  - Example: `tcp://postgres:5432`

---

