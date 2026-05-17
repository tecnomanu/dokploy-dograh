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

## Deploy en Dokploy

1. **Crear servicio Compose** apuntando a este repo (branch `main`, compose
   path `./docker-compose.yaml`, autodeploy ON).
2. **Env vars obligatorias:**
   ```
   REGISTRY=ghcr.io/dograh-hq
   COMPOSE_PROFILES=remote
   ENVIRONMENT=production
   PUBLIC_HOST=tu.dominio.com
   PUBLIC_BASE_URL=https://tu.dominio.com
   BACKEND_API_ENDPOINT=https://tu.dominio.com
   MINIO_PUBLIC_ENDPOINT=https://tu.dominio.com
   SERVER_IP=<IP del VPS>
   OSS_JWT_SECRET=<openssl rand -hex 32>
   ENABLE_TELEMETRY=true
   ```
3. **Domain:** servicio `nginx`, container port `80`, HTTPS + Let's Encrypt.
4. **DNS:** A record `tu.dominio.com` → IP del VPS, **gris/DNS-only** si
   estás detrás de Cloudflare.

## Actualizar

Las imágenes `api`/`ui` se actualizan solas vía registry. Si Dograh cambia
el compose o las templates upstream, se mergea a mano contra este repo
(probablemente bajo en frecuencia).

## Upstream
- Dograh original: https://github.com/dograh-hq/dograh (BSD 2-Clause)
- Imágenes oficiales: https://ghcr.io/dograh-hq
