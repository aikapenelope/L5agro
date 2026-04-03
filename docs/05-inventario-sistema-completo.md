# AgroInteligencia VE — Inventario Completo del Sistema y Separacion de Datos

## PRINCIPIO: Cada servicio es dueno de sus datos

Compartir una base de datos entre servicios (Agno, NestJS, Prefect, workers) genera:
- **Conflictos de schema**: Agno crea tablas `agno_sessions`, `agno_memories`, `agno_metrics`. Si otro servicio migra el schema, puede romper Agno.
- **Acoplamiento oculto**: Un cambio en la estructura de datos de un servicio afecta a otro sin aviso.
- **Contention de conexiones**: Agno, NestJS, y workers compitiendo por el pool de conexiones de una sola instancia PostgreSQL.
- **Backups complicados**: No puedes restaurar los datos de un servicio sin afectar a otro.
- **Escalabilidad bloqueada**: No puedes mover un servicio a otro servidor si comparte DB.

**Solucion**: Multiples bases de datos logicas dentro de una sola instancia PostgreSQL. Cada servicio tiene su propia database. Comparten el mismo servidor PostgreSQL pero estan completamente aisladas.

---

## INVENTARIO COMPLETO: Todo lo que necesita el sistema

### 1. BASES DE DATOS (PostgreSQL 16 — una instancia, multiples databases)

```
PostgreSQL 16 (un solo contenedor/proceso)
  |
  |-- database: agrointeligencia_app        <-- NestJS (datos de negocio)
  |     |-- extension: PostGIS
  |     |-- tablas: users, farms, parcels (geometrias), subscriptions,
  |     |           cooperatives, alerts_log, reports, invoices
  |     |-- proposito: datos de negocio, usuarios, parcelas, suscripciones
  |
  |-- database: agrointeligencia_agno       <-- Agno AgentOS (IA exclusivamente)
  |     |-- extension: pgvector
  |     |-- tablas: agno_sessions, agno_memories, agno_metrics,
  |     |           knowledge_agronomia (embeddings RAG),
  |     |           knowledge_plagas (embeddings RAG)
  |     |-- proposito: sesiones de chat, memoria de agentes, knowledge bases
  |
  |-- database: agrointeligencia_geo        <-- Workers satelitales (geodatos)
  |     |-- extension: PostGIS, postgis_raster
  |     |-- tablas: satellite_tiles, ndvi_history, ndwi_history,
  |     |           weather_cache, soil_data, crop_phenology
  |     |-- proposito: datos geoespaciales procesados, indices de vegetacion,
  |     |              cache meteorologico, datos de suelo
  |
  |-- database: agrointeligencia_prefect    <-- Prefect (orquestacion de workflows)
        |-- extension: pg_trgm (requerida por Prefect)
        |-- tablas: gestionadas automaticamente por Prefect (flow_runs,
        |           task_runs, deployments, work_pools, etc.)
        |-- proposito: estado de workflows, historial de ejecuciones, schedules
```

**Por que no 4 instancias PostgreSQL separadas**: En un VPS, una sola instancia PostgreSQL con 4 databases usa ~200MB RAM. Cuatro instancias usarian ~800MB. La separacion logica (databases) da el aislamiento necesario sin el overhead de multiples procesos.

---

### 2. PREFECT — Orquestacion de Workflows Pesados

**Por que Prefect y no solo BullMQ para todo:**

BullMQ es excelente para jobs simples (encolar → procesar → devolver resultado). Pero los pipelines satelitales y meteorologicos son **workflows multi-paso con dependencias, reintentos inteligentes, y scheduling**:

```
Pipeline Satelital (cada 5 dias):
  1. Buscar imagenes Sentinel-2 disponibles para parcelas activas
  2. Filtrar por nubosidad (<30%)
  3. Si no hay imagen limpia → buscar Sentinel-1 SAR como fallback
  4. Descargar imagen seleccionada
  5. Recortar por parcela (clip to polygon)
  6. Calcular NDVI, NDWI, EVI, SAVI
  7. Comparar con historico (anomalia detection)
  8. Si anomalia detectada → generar alerta
  9. Almacenar tiles procesados en MinIO
  10. Actualizar PostGIS con nuevos indices
  11. Notificar a Agno para que envie alertas via WhatsApp
```

Si el paso 4 falla (Copernicus API caida), Prefect reintenta con backoff exponencial sin re-ejecutar pasos 1-3. BullMQ re-ejecutaria todo el job desde cero.

**Prefect no consume tokens LLM**. Es puro Python con decoradores `@flow` y `@task`. No hay IA involucrada — es orquestacion determinista de codigo.

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=[60, 300, 900])
def descargar_sentinel2(parcela_id: str, fecha: str) -> str:
    """Descarga imagen de Copernicus STAC API"""
    # ... logica de descarga ...
    return ruta_geotiff

@task
def calcular_ndvi(ruta_geotiff: str) -> dict:
    """Calcula NDVI con rasterio/numpy"""
    # ... logica NDVI ...
    return {"ndvi_mean": 0.72, "ndvi_min": 0.31, ...}

@flow(name="pipeline-satelital")
def procesar_parcela(parcela_id: str):
    geotiff = descargar_sentinel2(parcela_id, "2026-04-01")
    ndvi = calcular_ndvi(geotiff)
    # ... mas pasos ...
```

**Prefect self-hosted**: Usa SQLite por defecto (cero config) o PostgreSQL para produccion. En nuestro caso, su propia database `agrointeligencia_prefect`. El servidor Prefect consume ~150MB RAM.

---

### 3. REDIS (una instancia, multiples databases logicas)

```
Redis 7 (un solo contenedor/proceso)
  |
  |-- db 0: BullMQ queues          <-- Jobs rapidos (WhatsApp responses, notificaciones)
  |-- db 1: NestJS sessions/cache  <-- Sesiones de usuario web, cache de API
  |-- db 2: Agno cache             <-- Cache de respuestas de agentes, rate limiting
  |-- db 3: General cache          <-- Cache de tiles satelitales metadata, weather data
```

Redis soporta 16 databases logicas (db 0 a db 15) por defecto. Cada una es aislada. No hay conflicto de keys entre servicios.

---

### 4. MINIO — Object Storage (un solo servicio, multiples buckets)

```
MinIO
  |
  |-- bucket: satellite-raw        <-- GeoTIFFs crudos descargados de Copernicus
  |-- bucket: satellite-processed  <-- Tiles NDVI/NDWI/EVI procesados (PNG/WebP)
  |-- bucket: reports              <-- PDFs de reportes semanales generados
  |-- bucket: knowledge-docs       <-- PDFs/documentos fuente para RAG (INIA, LUZ, etc.)
  |-- bucket: user-uploads         <-- Fotos de cultivos subidas por productores via WhatsApp
  |-- bucket: audio-lessons        <-- Audios de micro-lecciones (AgroEscuela)
```

Cada bucket tiene su propia politica de retencion y acceso. Los workers satelitales solo escriben en `satellite-*`. Agno solo lee de `knowledge-docs`. NestJS solo lee de `reports` y `user-uploads`.

---

### 5. SERVICIOS — Inventario completo

#### 5.1 Agno AgentOS (Python)
- **Funcion**: Nucleo de IA. Chatbot WhatsApp, agentes especializados, RAG agronomico.
- **Base de datos**: `agrointeligencia_agno` (PostgreSQL — sesiones, memorias, knowledge)
- **Redis**: db 2 (cache)
- **MinIO**: Lee de `knowledge-docs`
- **Puerto**: 8000
- **Recursos**: ~512MB-1GB RAM (sin LLM local). Con LLM local: +4-8GB.
- **Dependencias externas**: API de LLM (Groq/Together.ai/OpenAI), WhatsApp Business API (Meta)

#### 5.2 NestJS API (TypeScript)
- **Funcion**: API del dashboard web, autenticacion, gestion de usuarios/parcelas/cooperativas, billing.
- **Base de datos**: `agrointeligencia_app` (PostgreSQL — datos de negocio)
- **Redis**: db 1 (sesiones, cache)
- **MinIO**: Lee de `reports`, `satellite-processed`, `user-uploads`
- **Puerto**: 3000
- **Recursos**: ~200-400MB RAM
- **Dependencias externas**: Ninguna critica. Opcionalmente: Stripe/payment gateway.

#### 5.3 Nuxt 3 PWA (TypeScript)
- **Funcion**: Dashboard web para cooperativas y productores con acceso web.
- **Base de datos**: Ninguna directa. Consume NestJS API.
- **Puerto**: 3001 (dev) / servido por Caddy en produccion (static + SSR)
- **Recursos**: ~150-300MB RAM (SSR mode)
- **Dependencias externas**: MapLibre GL JS (mapas), NestJS API.

#### 5.4 Prefect Server (Python)
- **Funcion**: Orquestacion de workflows batch (satelital, meteorologico, reportes, ETL).
- **Base de datos**: `agrointeligencia_prefect` (PostgreSQL)
- **Puerto**: 4200 (UI) + 4201 (API)
- **Recursos**: ~150-300MB RAM
- **Dependencias externas**: Ninguna.

#### 5.5 Prefect Worker (Python)
- **Funcion**: Ejecuta los flows de Prefect. Contiene la logica de procesamiento satelital y meteorologico.
- **Base de datos**: `agrointeligencia_geo` (PostgreSQL — escribe datos procesados)
- **Redis**: db 3 (cache de metadata)
- **MinIO**: Escribe en `satellite-raw`, `satellite-processed`, `reports`
- **Recursos**: ~512MB-2GB RAM (picos durante procesamiento GDAL/rasterio)
- **Dependencias**: rasterio, GDAL, numpy, xarray, httpx, sentinelsat
- **Dependencias externas**: Copernicus STAC API, Open-Meteo API, CHIRPS

#### 5.6 LLM Server (Ollama o API externa)
- **Funcion**: Inferencia de modelos de lenguaje.
- **Opcion A — API externa (MVP)**: Groq/Together.ai. Sin contenedor local. Agno llama via HTTP.
- **Opcion B — Ollama local (dev/staging)**: Contenedor con Llama 3.1 8B quantizado.
- **Opcion C — vLLM (produccion con GPU)**: Servidor dedicado con GPU.
- **Recursos**: Opcion A: 0 RAM local. Opcion B: 4-8GB RAM. Opcion C: GPU + 16GB+ RAM.
- **Puerto**: 11434 (Ollama) o 8080 (vLLM)

#### 5.7 Redis (servicio compartido)
- **Funcion**: Cache, queues BullMQ, sesiones.
- **Recursos**: ~100-200MB RAM
- **Puerto**: 6379
- **Persistencia**: RDB snapshots cada 5 min. AOF para queues criticas.

#### 5.8 PostgreSQL 16 (servicio compartido)
- **Funcion**: Almacenamiento relacional, geoespacial, y vectorial.
- **Extensiones**: PostGIS, pgvector, pg_trgm, postgis_raster
- **Recursos**: ~300-500MB RAM (shared_buffers configurado)
- **Puerto**: 5432
- **Backups**: pg_dump por database individual. Cron diario.

#### 5.9 MinIO (servicio compartido)
- **Funcion**: Object storage S3-compatible.
- **Recursos**: ~200-400MB RAM
- **Puerto**: 9000 (API) + 9001 (console)
- **Almacenamiento**: Depende del volumen. ~50GB para MVP, ~500GB para produccion.

#### 5.10 Caddy (reverse proxy)
- **Funcion**: TLS automatico, routing a todos los servicios.
- **Recursos**: ~30-50MB RAM
- **Puerto**: 80, 443
- **Rutas**:
  - `app.agrointeligencia.ve` → Nuxt 3 (3001)
  - `api.agrointeligencia.ve` → NestJS (3000)
  - `ai.agrointeligencia.ve` → Agno AgentOS (8000)
  - `prefect.agrointeligencia.ve` → Prefect UI (4200) [acceso interno]

---

### 6. COMUNICACION ENTRE SERVICIOS

```
Productor envia WhatsApp
        |
        v
  Meta Webhook ──> Agno AgentOS (8000)
        |               |
        |               |-- consulta knowledge (agrointeligencia_agno / pgvector)
        |               |-- consulta LLM (Groq API / Ollama 11434)
        |               |-- si necesita datos de parcela:
        |               |      HTTP GET → NestJS API (3000)
        |               |        └── consulta agrointeligencia_app
        |               |-- si necesita datos satelitales:
        |               |      HTTP GET → NestJS API (3000)
        |               |        └── consulta agrointeligencia_geo
        |               |
        |               v
        |         Respuesta WhatsApp ──> Meta API
        |
  Cada 5 dias (Prefect schedule):
        |
        v
  Prefect Server (4200) ──> Prefect Worker
        |                        |
        |                        |-- descarga Sentinel-2 (Copernicus API)
        |                        |-- procesa NDVI/NDWI (rasterio/GDAL)
        |                        |-- almacena en MinIO (satellite-processed)
        |                        |-- escribe en agrointeligencia_geo
        |                        |-- si anomalia detectada:
        |                        |      encola alerta en BullMQ (Redis db 0)
        |                        |
        |                        v
        |                  NestJS consume alerta de BullMQ
        |                        |
        |                        v
        |                  NestJS pide a Agno generar mensaje de alerta
        |                        |
        |                        v
        |                  Agno envia alerta via WhatsApp
        |
  Dashboard web:
        |
        v
  Nuxt 3 (3001) ──> NestJS API (3000)
        |                |
        |                |-- consulta agrointeligencia_app (usuarios, parcelas)
        |                |-- consulta agrointeligencia_geo (NDVI, weather)
        |                |-- sirve tiles de MinIO (satellite-processed)
        |                v
        |          Renderiza mapas con MapLibre GL JS
```

---

### 7. RESUMEN DE RECURSOS (VPS Hetzner)

#### MVP (Fase 1) — Sin LLM local, LLM via API externa

| Servicio | RAM estimada |
|----------|-------------|
| Agno AgentOS | 512MB |
| NestJS API | 300MB |
| Nuxt 3 SSR | 200MB |
| Prefect Server | 200MB |
| Prefect Worker | 1GB (picos) |
| PostgreSQL 16 | 400MB |
| Redis 7 | 150MB |
| MinIO | 300MB |
| Caddy | 50MB |
| **TOTAL** | **~3.1GB** |

**VPS recomendado**: **CX32** (4 vCPU, 8GB RAM, 80GB SSD) — ~EUR 8/mes. Margen de 5GB para picos y OS.

Si se necesita mas procesamiento satelital concurrente: **CX41** (8 vCPU, 16GB RAM) — ~EUR 16/mes.

#### Produccion (Fase 2+) — Con LLM local (Ollama)

Agregar 4-8GB para Ollama con Llama 3.1 8B quantizado.
**VPS recomendado**: **CCX33** (8 vCPU dedicados, 32GB RAM) — ~EUR 55/mes.

---

### 8. DEPENDENCIAS EXTERNAS (APIs de terceros)

| Servicio | Uso | Costo | Criticidad |
|----------|-----|-------|------------|
| **Copernicus STAC API** | Descarga Sentinel-2/1 | Gratis | Alta — sin esto no hay monitoreo satelital |
| **Open-Meteo API** | Datos meteorologicos, ETo | Gratis (no-comercial) / EUR 300/mes (comercial) | Alta — sin esto no hay riego ni alertas clima |
| **CHIRPS** | Datos de precipitacion historica | Gratis (dominio publico) | Media — complementa Open-Meteo |
| **Meta WhatsApp Business API** | Envio/recepcion de mensajes | ~$0.05-0.08/conversacion | Critica — canal principal |
| **Groq / Together.ai** | Inferencia LLM | ~$0.05-0.10/1M tokens | Alta (si no hay LLM local) |
| **SoilGrids (ISRIC)** | Datos de suelo | Gratis | Baja — se descarga una vez |
| **ERA5-Land (Copernicus CDS)** | Datos climaticos historicos | Gratis | Media — para modelos de rendimiento |

---

### 9. SECRETOS Y CREDENCIALES

Gestionados via variables de entorno (`.env`) o Pulumi ESC:

| Secreto | Servicio que lo usa |
|---------|-------------------|
| `POSTGRES_PASSWORD` | Todos los que conectan a PostgreSQL |
| `REDIS_PASSWORD` | Todos los que conectan a Redis |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | Workers, NestJS |
| `WHATSAPP_ACCESS_TOKEN` | Agno |
| `WHATSAPP_PHONE_NUMBER_ID` | Agno |
| `WHATSAPP_APP_SECRET` | Agno |
| `GROQ_API_KEY` o `TOGETHER_API_KEY` | Agno (LLM) |
| `OPENAI_API_KEY` | Agno (embeddings, si se usa OpenAI embedder) |
| `COPERNICUS_CLIENT_ID` / `COPERNICUS_CLIENT_SECRET` | Prefect Worker |
| `JWT_SECRET` | NestJS (auth) |

---

### 10. DOMINIOS Y DNS

| Dominio | Destino | Proposito |
|---------|---------|-----------|
| `app.agrointeligencia.ve` | Nuxt 3 | Dashboard web |
| `api.agrointeligencia.ve` | NestJS | API REST |
| `ai.agrointeligencia.ve` | Agno AgentOS | API de IA + WhatsApp webhook |
| `storage.agrointeligencia.ve` | MinIO | Acceso a assets (tiles, reportes) |

Todos apuntan al mismo VPS. Caddy rutea por hostname.

---

### 11. BACKUPS

| Que | Como | Frecuencia | Retencion |
|-----|------|-----------|-----------|
| `agrointeligencia_app` | `pg_dump` | Diario | 30 dias |
| `agrointeligencia_agno` | `pg_dump` | Diario | 30 dias |
| `agrointeligencia_geo` | `pg_dump` | Semanal (datos grandes) | 14 dias |
| `agrointeligencia_prefect` | `pg_dump` | Semanal | 7 dias |
| MinIO buckets | `mc mirror` a backup externo | Diario (reports, knowledge) | 30 dias |
| Redis | RDB snapshot | Cada 5 min | 3 dias |
| Configuracion (docker-compose, .env) | Git | Cada cambio | Indefinido |

---

### 12. DOCKER-COMPOSE COMPLETO (conceptual)

```yaml
services:
  # --- Bases de datos ---
  postgres:
    image: postgis/postgis:16-3.4
    # Extensiones: PostGIS, pgvector, pg_trgm, postgis_raster
    # Init scripts crean las 4 databases
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-databases.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

  # --- Servicios de IA ---
  agno:
    build: ./services/agno
    # Python: Agno AgentOS + agentes + RAG
    depends_on: [postgres, redis]
    environment:
      - DATABASE_URL=postgresql://agno:pass@postgres/agrointeligencia_agno
      - REDIS_URL=redis://redis:6379/2

  # --- Servicios Web ---
  api:
    build: ./services/api
    # NestJS: Dashboard API + Auth
    depends_on: [postgres, redis]
    environment:
      - DATABASE_URL=postgresql://app:pass@postgres/agrointeligencia_app
      - REDIS_URL=redis://redis:6379/1

  web:
    build: ./services/web
    # Nuxt 3: PWA Dashboard
    depends_on: [api]

  # --- Orquestacion ---
  prefect-server:
    image: prefecthq/prefect:3-latest
    command: prefect server start
    depends_on: [postgres]
    environment:
      - PREFECT_API_DATABASE_CONNECTION_URL=postgresql://prefect:pass@postgres/agrointeligencia_prefect

  prefect-worker:
    build: ./services/worker-geo
    # Python: rasterio, GDAL, numpy + Prefect agent
    depends_on: [prefect-server, postgres, minio]
    environment:
      - DATABASE_URL=postgresql://geo:pass@postgres/agrointeligencia_geo
      - PREFECT_API_URL=http://prefect-server:4200/api

  # --- Reverse Proxy ---
  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

Esto es el sistema completo. Cada servicio tiene su propia base de datos, su propio espacio en Redis, y su propio bucket en MinIO. No hay datos compartidos entre servicios excepto a traves de APIs HTTP.
