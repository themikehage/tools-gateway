# tools-gateway — Diseño (genérico)

**Fecha:** 2026-10-13
**Estado:** Aprobado (pendiente de revisión del spec)

> Este repositorio es un **gateway genérico**. No contiene URLs, dominios, nombres de
> servicios ni identidades de clientes reales: todo eso es **configuración** que se
> monta en runtime y vive fuera del repo.

## 1. Problema

Se acumulan APIs internas (REST/HTTP) consumidas por agentes de IA. Hoy cada una se
expone por su cuenta, **sin autenticación**, con formatos y URLs distintas. Eso obliga
a cada consumidor a conocer cada servicio y deja los recursos abiertos a cualquiera
que descubra la URL.

## 2. Objetivos y no-objetivos

**Objetivos**

- **Punto de acceso único** para N APIs internas.
- **Autenticación por cliente** con tokens (registry), **scopes** y **rate limit**.
- **Desacoplado y extensible**: añadir un servicio o un cliente = una entrada de
  config; sin cambios de código.
- Proxy **transparente**: no altera el comportamiento de las APIs.

**No-objetivos (por ahora)**

- Normalizar/reescribir las respuestas de las APIs (pasan transparentes; el envelope
  JSON de §9 es solo para errores propios del gateway).
- Interfaz MCP / tools nativas.
- Dashboard de observabilidad (solo logs).
- Retirar la exposición directa de los servicios (fase 2, §11).

## 3. Decisiones

1. Gateway HTTP unificado (no MCP, no token por servicio suelto).
2. Consumidores: cualquier agente/cliente con token.
3. Se despliega como contenedor (Docker) y se publica con un reverse proxy/TLS al frente.
4. Auth: tokens **por cliente** (registry), no token compartido.
5. En fase 1 los servicios destino se dejan como están; cerrar exposición en fase 2.
6. Rutas: proxy transparente por **prefijo de path**.
7. Scopes: cada cliente puede restringirse a un subconjunto de rutas.

## 4. Arquitectura

```
cliente ──TLS──> proxy/TLS ───> tools-gateway (contenedor Bun)
                                     │
             auth → scope → rate limit → match prefijo → proxy
                                     │
                    ┌────────────────┼────────────────┐
                 upstream-1      upstream-2       upstream-N
                (red interna, declarados en config)
```

Componentes en un único proceso, sin dependencias externas:

1. **Config declarativa** (YAML montado por volumen).
2. **Auth** — valida `Authorization: Bearer <token>`.
3. **Autorización por scope** — allowlist de prefijos por cliente.
4. **Rate limiter** — token bucket en memoria por cliente.
5. **Proxy** — transparente; respeará método/body/streaming; limpia hop-by-hop.
6. **Logger** — una línea JSON a stdout por request.
7. **Health** — `GET /healthz`.

## 5. Configuración

`config.yaml` (montado como volumen; el repo solo lleva `config.example.yaml`):

```yaml
routes:
  service-a: http://upstream-a:3000
  service-b: http://upstream-b:8080

clients:
  - id: client-1
    token_hash: "<sha256 hex>"
    allow: ["*"]
    rate: { rps: 10, burst: 20 }
  - id: client-2
    token_hash: "<sha256 hex>"
    allow: ["service-a"]
    rate: { rps: 5, burst: 10 }
```

- `routes`: mapa `prefijo → upstream`. La clave es el **primer segmento del path**,
  comparado exacto y luego **eliminado** al construir la URL del upstream
  (`/service-a/foo` → `http://upstream-a:3000/foo`); no es match por substring.
- `clients`: `id`, `token_hash` (**solo hash**), `allow` (prefijos o `["*"]`),
  `rate` (`rps`, `burst`).
- Los hashes se generan con un pequeño CLI del repo (`bun run token:new`).

## 6. Flujo de una petición

```
GET /service-a/foo  (Bearer <token>)
 1. Match de prefijo  → upstream service-a
 2. Auth              → busca el token por hash; 401 si falta o no existe
 3. Scope             → ¿el cliente puede "service-a"? 403 si no
 4. Rate limit        → token bucket del cliente; 429 + Retry-After si excede
 5. Reescritura       → /service-a/foo → http://upstream-a:3000/foo
 6. Proxy             → reenvía método/body/headers (menos hop-by-hop y Authorization),
                        añade X-Forwarded-For/-Host y X-Gateway-Client
 7. Log + respuesta   → streamea el upstream tal cual (status/headers/body)
```

Se **elimina `Authorization`** hacia el upstream para no filtrar el token aunque un
backend lo registre.

## 7. Auth y scopes

- Identidad **por cliente**: un token por cliente, guardado como **hash** (nunca el
  valor). Comparación en tiempo constante.
- Los tokens en claro se generan una vez y se entregan fuera del repo.
- **Rotación/revocación**: añadir/quitar la entrada y recargar; el resto no se afecta.
- **Scopes**: `allow: ["*"]` o lista explícita. Fuera de scope → `403`.

## 8. Rate limiting

- Token bucket **en memoria**, por `client.id`, con `rps`/`burst` configurables.
- Exceso → `429` con `Retry-After`.
- Suficiente para una sola instancia. Escalar a >1 réplica requiere store compartido
  (fuera de alcance).

## 9. Errores

- Errores **propios del gateway** con envoltorio JSON:
  ```
  401 {"error":"unauthorized"}
  403 {"error":"forbidden","scope":"service-a"}
  404 {"error":"unknown_route"}
  429 {"error":"rate_limited","retry_after":2}
  502 {"error":"bad_gateway"}
  504 {"error":"gateway_timeout"}
  ```
- Errores del **upstream** pasan transparentes.

## 10. Observabilidad y despliegue

- **Logs**: una línea JSON por request a stdout
  (`{ts, client, method, path, status, ms, upstream, bytes}`). Nunca el token.
- **Deploy**: imagen Docker; el operador define dominio/TLS, healthcheck
  `GET /healthz` y volumen para `config.yaml`.

## 11. Fase 2 (diferida)

Una vez los clientes migren al gateway, **retirar la exposición directa** de los
servicios para que el gateway sea la única puerta.

## 12. Testing

- **Unit**: auth (válido/ inválido/ ausente), scope (permitido/denegado), rate limit,
  match de prefijo, strip de cabeceras.
- **Integración**: arranque local con un upstream falso y `config.yaml` de prueba.
- **E2E**: `curl` contra el gateway con un token de prueba por prefijo.

## 13. Seguridad (transversal)

- TLS termina en el proxy externo; el gateway habla HTTP interno.
- Tokens solo como hash; comparación timing-safe; nunca se loguean.
- Se quita `Authorization` antes de reenviar.
- Upstreams por red interna.
- Rechazo de cuerpos desmesurados (límite configurable).

## 14. Fuera de alcance

- MCP, envoltorio JSON unificado, métricas, multi-réplica, store de rate limit
  compartido y autogestión de tokens por los propios clientes.
