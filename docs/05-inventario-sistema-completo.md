# AgroInteligencia VE — Inventario Completo del Sistema (v2)

## PRINCIPIO: Cada servicio usa la base de datos que su proyecto recomienda

No compartimos bases de datos entre servicios. Cada OSS project tiene su propia DB recomendada. Forzar todo en un solo PostgreSQL genera conflictos de migraciones, extensiones, y versiones.

Los 4 servicios que estaban en el mismo PostgreSQL y por que eso es problema:

| Servicio | Que necesita | Conflicto si comparten PostgreSQL |
|----------|-------------|----------------------------------|
| **NestJS** (app) | PostGIS, tablas de negocio, migraciones con TypeORM/Drizzle | TypeORM corre migraciones automaticas que pueden romper tablas de otros |
| **Agno** (IA) | Tablas `agno_sessions`, `agno_memories`, `agno_metrics`, pgvector para knowledge | Agno maneja su propio schema con Alembic. Conflicto con migraciones de NestJS |
| **Workers geo** (satelital) | PostGIS, postgis_raster, tablas de indices de vegetacion | Requiere extensiones pesadas (postgis_raster) que pueden afectar performance de queries de Agno |
| **Prefect** (workflows) | pg_trgm, tablas de flow_runs/task_runs, migraciones con Alembic | Prefect tiene su propio Alembic. Dos Alembic en la misma DB = desastre |

---

## ARQUITECTURA DE DATOS: Cada servicio con su DB recomendada

```
┌─────────────────────────────────────────────────────────────────┐
│                    BASES DE DATOS                                │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ PostgreSQL   │  │ SQLite       │  │ LanceDB      │          │
│  │ + PostGIS    │  │ (archivo)    │  │ (directorio) │          │
│  │              │  │              │  │              │          │
│  │ NestJS app   │  │ Agno         │  │ Agno RAG     │          │
│  │ Workers geo  │  │ sesiones     │  │ knowledge    │          │
│  │              │  │ memorias     │  │ embeddings   │          │
│  │ (2 databases │  │ metricas     │  │ vectoriales  │          │
│  │  separadas)  │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │ SQLite       │  │ Redis 7      │                             │
│  │ (archivo)    │  │              │                             │
│  │              │  │ db 0: BullMQ │                             │
│  │ Prefect      │  │ db 1: NestJS │                             │
│  │ server       │  │ db 2: cache  │                             │
│  └──────────────┘  └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

### 1. PostgreSQL 16 + PostGIS — Solo para datos geoespaciales y de negocio

**Quien lo usa**: NestJS (app) y Workers satelitales (geo)

**Por que PostgreSQL aqui**: PostGIS es la unica opcion seria para queries geoespaciales (ST_Contains, ST_Distance, ST_Intersects sobre poligonos de parcelas). No hay alternativa.

**Separacion interna** (2 databases en la misma instancia):

```
PostgreSQL 16 + PostGIS
  |
  |-- database: app_db
  |     |-- extension: PostGIS
  |     |-- owner: NestJS
  |     |-- migraciones: TypeORM / Drizzle
  |     |-- tablas:
  |     |     users              -- productores, tecnicos, admins
  |     |     farms              -- fincas (metadata)
  |     |     parcels            -- parcelas con geometria (POLYGON)
  |     |     cooperatives       -- cooperativas
  |     |     subscriptions      -- suscripciones y billing
  |     |     alerts_log         -- historial de alertas enviadas
  |     |     crop_cycles        -- ciclos de cultivo por parcela
  |     |     invoices           -- facturacion
  |
  |-- database: geo_db
        |-- extension: PostGIS, postgis_raster
        |-- owner: Prefect Workers (Python)
        |-- migraciones: Alembic o SQL manual
        |-- tablas:
              ndvi_history       -- serie temporal NDVI por parcela
              ndwi_history       -- serie temporal NDWI
              evi_history        -- serie temporal EVI
              weather_cache      -- cache de datos meteorologicos
              soil_data          -- datos de suelo (SoilGrids)
              satellite_metadata -- metadata de imagenes procesadas
              crop_phenology     -- fases fenologicas detectadas
```

**Por que 2 databases y no 1**: NestJS usa TypeORM/Drizzle para migraciones. Workers usan Alembic o SQL directo. Dos sistemas de migracion en la misma database = conflicto garantizado. Databases separadas, cada una con su propio usuario PostgreSQL y permisos.

**Comunicacion**: NestJS necesita datos geo (NDVI de una parcela). Lo hace via API interna HTTP al worker, o con un usuario PostgreSQL read-only que puede consultar `geo_db` pero no escribir.

---

### 2. SQLite — Agno (sesiones, memorias, metricas)

**Quien lo usa**: Agno AgentOS

**Por que SQLite y no PostgreSQL**:
- Agno recomienda SQLite para deployments single-server. Es su ejemplo principal en docs y cookbooks.
- SQLite es un archivo. No hay servidor, no hay conexiones, no hay conflictos de migraciones con otros servicios.
- Agno maneja su propio schema (`agno_sessions`, `agno_memories`, `agno_metrics`). Con SQLite, ese schema esta completamente aislado en un archivo `.db`.
- Performance: Para sesiones de chat y memorias de agentes, SQLite es mas rapido que PostgreSQL (sin overhead de red, sin connection pool).
- Backup: copiar un archivo. `cp agno.db agno.db.backup`. No hay `pg_dump`.

**Tablas que Agno crea automaticamente**:
```
agno.db (SQLite file)
  |-- agno_sessions    -- sesiones de chat por usuario
  |-- agno_memories    -- memorias aprendidas de cada usuario
  |-- agno_metrics     -- metricas de uso de agentes
```

**Configuracion**:
```python
from agno.db.sqlite import SqliteDb

db = SqliteDb(
    db_file="/data/agno/agno.db",
    session_table="agno_sessions",
    memory_table="agno_memories",
    metrics_table="agno_metrics",
)
```

**Volumen Docker**: `/data/agno/` montado como volumen persistente.

---

### 3. LanceDB — Agno RAG (knowledge base vectorial)

**Quien lo usa**: Agno (knowledge/embeddings para RAG agronomico)

**Por que LanceDB y no pgvector**:
- LanceDB es embedded (archivo en disco). No hay servidor. No compite con PostgreSQL por conexiones o RAM.
- Agno tiene soporte nativo para LanceDB como vector store. Es el ejemplo recomendado junto con SQLite para setups locales.
- LanceDB usa formato columnar (Lance), optimizado para busqueda vectorial. Mas eficiente que pgvector para datasets de 5,000-100,000 documentos.
- Zero-copy reads: no carga todo en RAM. Lee directamente del disco con mmap.
- No necesita extension de PostgreSQL (pgvector requiere compilar extension, puede conflictuar con PostGIS).
- Backup: copiar el directorio. `cp -r /data/lancedb/ /backup/lancedb/`.

**Estructura**:
```
/data/lancedb/  (directorio)
  |-- agronomia_ve/     -- knowledge base principal (INIA, LUZ, UCV, FONAIAP)
  |-- plagas_ve/        -- base de plagas venezolanas
  |-- cultivos_ve/      -- fichas tecnicas de cultivos
  |-- fitosanitarios/   -- productos fitosanitarios disponibles en VE
```

**Configuracion**:
```python
from agno.knowledge.knowledge import Knowledge
from agno.vectordb.lancedb import LanceDb
from agno.knowledge.embedder.ollama import OllamaEmbedder

knowledge = Knowledge(
    vector_db=LanceDb(
        table_name="agronomia_ve",
        uri="/data/lancedb",
        embedder=OllamaEmbedder(id="nomic-embed-text", dimensions=768),
    ),
)
```

**Embeddings**: Modelo local `nomic-embed-text` via Ollama. Sin costo de API. Sin dependencia externa.

---

### 4. SQLite — Prefect Server (estado de workflows)

**Quien lo usa**: Prefect Server

**Por que SQLite y no PostgreSQL**:
- Prefect recomienda SQLite para deployments single-server: "Recommended for lightweight, single-server deployments. SQLite requires essentially no setup."
- AgroInteligencia VE es un solo VPS. No hay multi-server. SQLite es la opcion correcta.
- Prefect maneja su propio schema con Alembic. Con SQLite, ese schema esta completamente aislado.
- Si en el futuro se necesita multi-server, se migra a PostgreSQL dedicado. Pero para MVP y Fase 2, SQLite es suficiente.

**Archivo**: `/data/prefect/prefect.db` (creado automaticamente por Prefect)

**Volumen Docker**: `/data/prefect/` montado como volumen persistente.

---

### 5. Redis 7 — Cache, queues, sesiones web

**Quien lo usa**: BullMQ, NestJS, cache general

**Separacion por database logica**:
```
Redis 7
  |-- db 0: BullMQ queues     -- jobs rapidos (notificaciones, alertas)
  |-- db 1: NestJS sessions    -- sesiones web, cache de API
  |-- db 2: cache general      -- metadata de tiles, weather data temporal
```

---

### 6. MinIO — Object Storage

**Quien lo usa**: Workers satelitales, NestJS, Agno

**Buckets**:
```
MinIO
  |-- satellite-raw        -- GeoTIFFs crudos de Copernicus
  |-- satellite-processed  -- Tiles NDVI/NDWI procesados (PNG/WebP)
  |-- reports              -- PDFs de reportes semanales
  |-- knowledge-docs       -- PDFs fuente para RAG (INIA, LUZ, etc.)
  |-- user-uploads         -- Fotos de cultivos via WhatsApp
  |-- audio-lessons        -- Audios de micro-lecciones
```

---

## RESUMEN: Que base de datos usa cada servicio

| Servicio | DB de estado/sesiones | DB vectorial (RAG) | DB geoespacial | DB de workflows |
|----------|----------------------|--------------------|--------------------|-----------------|
| **Agno AgentOS** | SQLite (`agno.db`) | LanceDB (`/data/lancedb/`) | - | - |
| **NestJS API** | Redis db 1 (sesiones web) | - | PostgreSQL (`app_db`) | - |
| **Prefect Server** | SQLite (`prefect.db`) | - | - | SQLite (`prefect.db`) |
| **Prefect Worker** | - | - | PostgreSQL (`geo_db`) | - |
| **Nuxt 3 PWA** | - (consume NestJS API) | - | - | - |
| **LLM (Ollama/API)** | - (stateless) | - | - | - |

**Cero bases de datos compartidas entre servicios.** Cada servicio es dueno exclusivo de su almacenamiento.

---

## DOCKER-COMPOSE ACTUALIZADO

```yaml
services:
  # --- Bases de datos ---
  postgres:
    image: postgis/postgis:16-3.4
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-databases.sql:/docker-entrypoint-initdb.d/init.sql
    # init.sql crea: app_db (NestJS) y geo_db (workers)
    # con usuarios separados y permisos aislados

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
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
    volumes:
      - agno_data:/data/agno       # SQLite: agno.db
      - lancedb_data:/data/lancedb  # LanceDB: knowledge bases
    environment:
      - AGNO_DB_FILE=/data/agno/agno.db
      - LANCEDB_URI=/data/lancedb

  # --- Servicios Web ---
  api:
    build: ./services/api
    depends_on: [postgres, redis]
    environment:
      - DATABASE_URL=postgresql://app_user:${APP_DB_PASS}@postgres/app_db
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/1

  web:
    build: ./services/web
    depends_on: [api]

  # --- Orquestacion ---
  prefect-server:
    image: prefecthq/prefect:3-latest
    command: prefect server start --host 0.0.0.0
    volumes:
      - prefect_data:/data/prefect  # SQLite: prefect.db
    environment:
      - PREFECT_SERVER_DATABASE_CONNECTION_URL=sqlite+aiosqlite:////data/prefect/prefect.db

  prefect-worker:
    build: ./services/worker-geo
    depends_on: [prefect-server, postgres, minio]
    environment:
      - DATABASE_URL=postgresql://geo_user:${GEO_DB_PASS}@postgres/geo_db
      - PREFECT_API_URL=http://prefect-server:4200/api
      - MINIO_ENDPOINT=minio:9000

  # --- Reverse Proxy ---
  caddy:
    image: caddy:2-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

volumes:
  postgres_data:     # PostgreSQL (app_db + geo_db)
  redis_data:        # Redis
  minio_data:        # MinIO buckets
  agno_data:         # Agno SQLite
  lancedb_data:      # LanceDB knowledge bases
  prefect_data:      # Prefect SQLite
```

---

## RESUMEN DE RECURSOS (actualizado)

| Servicio | RAM | Almacenamiento |
|----------|-----|---------------|
| PostgreSQL 16 (app_db + geo_db) | 300MB | 5-50GB (crece con datos geo) |
| Redis 7 | 100MB | <1GB |
| MinIO | 200MB | 50-500GB (GeoTIFFs) |
| Agno AgentOS + SQLite + LanceDB | 400MB | 1-5GB (knowledge base) |
| NestJS API | 250MB | - |
| Nuxt 3 SSR | 200MB | - |
| Prefect Server + SQLite | 150MB | <1GB |
| Prefect Worker (rasterio/GDAL) | 1GB (picos) | - |
| Caddy | 30MB | - |
| **TOTAL** | **~2.6GB** | **60-560GB** |

**VPS MVP**: CX32 (4 vCPU, 8GB RAM, 80GB SSD) — EUR 8/mes. Sobra RAM.
