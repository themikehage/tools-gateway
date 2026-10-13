# tools-gateway — Plan de implementación

**Fecha:** 2026-10-13
**Spec:** `docs/superpowers/specs/2026-10-13-tools-gateway-design.md`

Plan genérico (sin datos concretos). Los valores reales (rutas, clientes, dominio)
vienen de `config.yaml`, que no está en el repo.

## Stack

Bun + Hono + TypeScript. Tests con `bun test`. Imagen Docker multi-stage.

## Estructura de ficheros objetivo

```
src/
  index.ts        # arranque, wiring de middlewares, /healthz
  config.ts       # carga + validación (Zod) del YAML
  types.ts        # tipos de config y contexto
  auth.ts         # Bearer → hash → lookup timing-safe
  scopes.ts       # autorización por prefijo
  ratelimit.ts    # token bucket en memoria
  proxy.ts        # match de prefijo + reenvío transparente
  errors.ts       # envelope JSON de errores propios
  log.ts          # logger JSON a stdout
scripts/
  token-new.ts    # genera token + hash
tests/
  *.test.ts
config.example.yaml
Dockerfile
```

## Tareas

### T1 — Scaffolding
- `bun init`, deps: `hono`, `zod`, `yaml`.
- `tsconfig.json`, scripts (`dev`, `start`, `test`, `token:new`).
- `Dockerfile` multi-stage (builder + runner `oven/bun:1-debian`), healthcheck.
- **Verificación:** `bun run start` levanta y responde `/healthz`.

### T2 — Config
- Cargar `config.yaml` de ruta por env (`CONFIG_PATH`, default `/app/config.yaml`).
- Validar con Zod: `routes` (record string→url), `clients` (`id`, `token_hash`
  hex, `allow` array, `rate.rps`/`burst`).
- Fallar al arrancar con mensaje claro si falta o es inválido.
- **Verificación:** tests de config válida/inválida.

### T3 — CLI de tokens
- `bun run token:new` → imprime token en claro (una vez) + `sha256` para la config.
- **Verificación:** hash generado coincide al hashear el token.

### T4 — Auth
- Parsear `Authorization: Bearer <t>`; hashear; buscar cliente por comparación
  timing-safe; adjuntar `client` al contexto. Ausente/inválido → `401`.
- **Verificación:** unit (válido, inválido, ausente, formato raro).

### T5 — Scopes
- `allow` con `["*"]` o lista de prefijos. Si el prefijo no está permitido → `403`.
- **Verificación:** unit permitido/denegado.

### T6 — Rate limit
- Token bucket en memoria por `client.id` (`rps`/`burst`); exceso → `429` +
  `Retry-After`.
- **Verificación:** unit dentro/fuera del bucket.

### T7 — Proxy transparente
- Match por **primer segmento** exacto; strip del prefijo; construir URL.
- Reenviar método, body y headers; eliminar hop-by-hop y `Authorization`; añadir
  `X-Forwarded-For`/`-Host` y `X-Gateway-Client`; streaming de respuesta.
- Límite de tamaño de cuerpo (configurable).
- **Verificación:** integración con upstream falso (echo) — status, headers y body
  idénticos; `Authorization` no llega; prefijo bien recortado.

### T8 — Errores
- Envelope JSON solo para errores propios (`401/403/404/429/502/504`); upstream
  pasa transparente.
- **Verificación:** unit + e2e.

### T9 — Logging
- Línea JSON por request (`ts, client, method, path, status, ms, upstream, bytes`);
  nunca el token.
- **Verificación:** inspección de stdout en integración.

### T10 — Health
- `GET /healthz` → `200 {"status":"ok"}` sin auth.
- **Verificación:** e2e.

### T11 — Tests
- Unit (auth, scope, rate limit, prefix, header-strip, config).
- Integración: arranque con upstream falso + `config.yaml` de prueba.
- E2E: `curl` contra el gateway con token de prueba.

### T12 — Despliegue
- App Coolify en el host, Dockerfile, volumen para `config.yaml`, healthcheck
  `/healthz`, dominio + TLS.
- Alta DNS del subdominio (paso externo).
- **Verificación:** `curl` con token por cada prefijo; sin token → 401; fuera de
  scope → 403.

## Criterios de aceptación

- Añadir una ruta/cliente no toca código (solo `config.yaml` + reload/redeploy).
- Sin token → 401; token fuera de scope → 403; exceso → 429.
- Las 6 integraciones responden igual a través del gateway que directo.
- `/healthz` verde y logs JSON por request sin secretos.

## Pendiente de confirmar (concreto, fuera del repo)

- Upstreams internos reales (contenedor:puerto) de cada servicio.
- Paths/verbos exactos de cada API.
- DNS del dominio y compatibilidad con el borde (Cloudflare Access).
