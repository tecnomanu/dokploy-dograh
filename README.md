# dokploy-dograh

Installer minimal para deployar [Dograh](https://github.com/dograh-hq/dograh)
(plataforma open-source de voice AI agents) en un VPS con Dokploy + Traefik.

## Qué es y qué no es

- ✅ **Es** el scaffold mínimo (compose + scripts/templates) para que Dokploy
  levante Dograh detrás de Traefik (TLS de Let's Encrypt al frente).
- ❌ **No es** un fork del código de Dograh. Las imágenes `api` y `ui` se
  pullean del registry oficial `ghcr.io/dograh-hq` — los updates de la app
  vienen automáticos por release.

## Patches respecto al compose oficial

1. Servicio `nginx`: `ports: 80:80 / 443:443` → `expose: ["80"]` para no
   chocar con Traefik (que ya bindea 80/443 del host).
2. Template `deploy/templates/nginx.remote.conf.template`: solo HTTP en :80
   (Traefik termina TLS). El bloque `listen 443 ssl` se sacó.
3. Servicio `cloudflared` eliminado (no aplica con Dokploy + LE).
4. Dependencia `cloudflared` removida del `depends_on` del `api`.
5. `certs/local.crt`+`local.key` dummy (autofirmados) para que pase el chequeo
   del `run_dograh_init.sh`. Nginx no los usa (no hay bloque SSL).

## Deploy en Dokploy — paso a paso

### 1. Crear servicio Compose
- Tipo: **Compose**
- Provider: **Git** → `https://github.com/tecnomanu/dokploy-dograh`
- Branch: `main`
- Compose Path: `./docker-compose.yaml`
- Autodeploy: ON

### 2. Cargar TODAS estas env vars (Environment tab)

Copiá tal cual y reemplazá los placeholders en `< >`:

```env
# Imágenes y profile (sin esto el nginx no arranca)
REGISTRY=ghcr.io/dograh-hq
COMPOSE_PROFILES=remote
ENVIRONMENT=production
ENABLE_TELEMETRY=true

# Tu dominio público (ojo: los 3 endpoints abajo deben ser IDÉNTICOS a PUBLIC_BASE_URL)
PUBLIC_HOST=<tu.dominio.com>
PUBLIC_BASE_URL=https://<tu.dominio.com>
BACKEND_API_ENDPOINT=https://<tu.dominio.com>
MINIO_PUBLIC_ENDPOINT=https://<tu.dominio.com>

# Server
SERVER_IP=<IP_PUBLICA_VPS>
FASTAPI_WORKERS=1

# TURN (obligatorio — TURN_HOST DEBE ser igual a PUBLIC_HOST)
TURN_HOST=<tu.dominio.com>
TURN_SECRET=<correr: openssl rand -hex 32>

# JWT del backend
OSS_JWT_SECRET=<correr: openssl rand -hex 32>
```

### 3. Constraints que el validador enforce (si no se cumplen, `dograh-init` falla con exit 1)

- `TURN_HOST` **debe ser igual** a `PUBLIC_HOST`
- `BACKEND_API_ENDPOINT` **debe ser idéntico** a `PUBLIC_BASE_URL` (incluyendo `https://`)
- `MINIO_PUBLIC_ENDPOINT` **debe ser idéntico** a `PUBLIC_BASE_URL`
- `SERVER_IP` debe ser una IPv4 válida
- `FASTAPI_WORKERS` entero ≥ 1

### 4. Domain
- Service Name: `nginx`
- Container Port: `80`
- HTTPS: ON, Let's Encrypt

### 5. DNS
- A record `<tu.dominio.com>` → `<IP_VPS>`
- **Gris/DNS-only** si está detrás de Cloudflare (si no, LE falla el challenge HTTP-01)

### 6. Deploy
Tarda ~2-3 min la primera vez (pull de imágenes). Verificá los logs:
- `dograh_init` debe terminar **Exited (0)** con mensaje
  `✓ dograh-init rendered remote nginx and coturn config`.
  Si exit 1 → log te dice qué env var falta o no matchea.
- `nginx_https`, `ui`, `api`, `coturn` deben quedar **running**.
- Entrá a `https://<tu.dominio.com>` — UI de Dograh.

## Generar secrets rápido

```bash
openssl rand -hex 32   # OSS_JWT_SECRET y TURN_SECRET (uno cada uno)
```

## Actualizar

Las imágenes `api`/`ui` se actualizan solas vía registry (autodeploy). Si Dograh
cambia el compose o las templates upstream, sincronizás a mano contra este repo
(probablemente bajo en frecuencia).

## Upstream
- Dograh original: https://github.com/dograh-hq/dograh (BSD 2-Clause)
- Imágenes oficiales: https://ghcr.io/dograh-hq
