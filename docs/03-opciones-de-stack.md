# AgroInteligencia VE — 3 Opciones de Stack Tecnico

Contexto: VPS individual en Hetzner Cloud. El stack debe soportar procesamiento satelital, IA/LLM, RAG vectorial, WhatsApp API, y una web app — todo en un servidor o cluster pequeno.

---

## OPCION A — Python-Centric (FastAPI + Celery + GDAL)

### Filosofia
Todo en Python. El ecosistema geoespacial y de ML es nativo en Python. Maximiza la reutilizacion de librerias cientificas sin bridges entre lenguajes.

### Componentes

| Capa | Tecnologia | Justificacion |
|------|-----------|---------------|
| **API Backend** | FastAPI | Async nativo, OpenAPI auto-docs, tipado fuerte con Pydantic. El estandar de facto para APIs Python en 2026. |
| **Task Queue / Workers** | Celery + Redis | Jobs asincrono para procesamiento satelital pesado (descarga, NDVI, tiles). Maduro, probado, simple. |
| **Satelital** | rasterio + GDAL + numpy | Stack geoespacial de referencia. rasterio para I/O de GeoTIFF, numpy para calculo de indices (NDVI, NDWI, EVI). |
| **Acceso Satelital** | sentinelsat / odc-stac + Copernicus STAC API | Busqueda y descarga de Sentinel-2/Sentinel-1 via STAC. |
| **Meteorologia** | openmeteo-requests + httpx | Cliente para Open-Meteo API (ETo, temperatura, precipitacion, humedad). |
| **Base de Datos** | PostgreSQL 16 + PostGIS + pgvector | Relacional + geoespacial + vectores para RAG. Una sola DB para todo. |
| **RAG / Embeddings** | LangChain + pgvector | Embeddings con modelo local (all-MiniLM-L6-v2 o similar). Busqueda vectorial en PostgreSQL. |
| **LLM Inference** | Ollama (dev) / vLLM (prod) | Ollama para desarrollo rapido. vLLM cuando haya concurrencia real (>10 usuarios simultaneos). Modelo: Llama 3.1 8B o Mistral 7B. |
| **Orquestacion IA** | LangGraph | Grafos de estado para agentes (clima, plagas, riego, rendimiento). Mas control que CrewAI. |
| **WhatsApp** | whatsapp-business-api via 360dialog o Twilio | 360dialog es mas barato para LATAM. Twilio mas documentado. |
| **Frontend Web** | Nuxt 3 (Vue) o Next.js (React) | PWA con offline support via Service Workers. Dashboard para cooperativas. |
| **Cache** | Redis | Cache de tiles satelitales procesados, sesiones, rate limiting. |
| **Object Storage** | MinIO (self-hosted) | Almacenamiento de GeoTIFFs, PDFs generados, assets. Compatible S3. |
| **Reverse Proxy** | Caddy o Traefik | TLS automatico, routing. |

### Pros
- **Ecosistema geoespacial nativo**: rasterio, GDAL, shapely, geopandas, xarray — todo es Python. Sin bridges.
- **Ecosistema ML/AI nativo**: scikit-learn, PyTorch, LangChain, LangGraph — todo Python.
- **Contratacion**: Python es el lenguaje mas comun en data science y agritech. Facil encontrar talento.
- **Prototipado rapido**: De Jupyter notebook a produccion con minimo refactoring.
- **Comunidad**: Cada problema geoespacial + ML tiene solucion documentada en Python.

### Contras
- **Concurrencia limitada**: Python GIL limita paralelismo real. Celery mitiga pero agrega complejidad operacional.
- **Consumo de memoria**: GDAL + numpy + LLM en el mismo servidor requiere gestion cuidadosa de RAM.
- **Dos runtimes**: Backend Python + Frontend JS/TS. Dos ecosistemas de dependencias.
- **Celery operacional**: Redis como broker + Celery workers requieren monitoreo (Flower). Puede ser fragil bajo carga.

### Costo estimado VPS
- **Minimo viable**: CX41 (8 vCPU, 16GB RAM, 160GB SSD) — ~EUR 16/mes. Suficiente para MVP sin LLM local.
- **Con LLM local**: CCX33 (8 vCPU dedicados, 32GB RAM) — ~EUR 55/mes. Necesario para Llama 3.1 8B quantizado.
- **Produccion**: CCX43 (16 vCPU, 64GB RAM) — ~EUR 110/mes. LLM + procesamiento satelital concurrente.

---

## OPCION B — TypeScript Full-Stack (NestJS + BullMQ + GDAL bindings)

### Filosofia
Un solo lenguaje (TypeScript) para backend, frontend, y orquestacion. Reduce context-switching del equipo. Usa bindings nativos para las partes geoespaciales.

### Componentes

| Capa | Tecnologia | Justificacion |
|------|-----------|---------------|
| **API Backend** | NestJS | Framework enterprise-grade para Node.js. Modular, inyeccion de dependencias, decoradores. |
| **Task Queue / Workers** | BullMQ + Redis | Queue nativa de Node.js. Mas simple que Celery, dashboard integrado (Bull Board). |
| **Satelital** | gdal-async (bindings GDAL) + sharp + geotiff.js | geotiff.js para lectura de COGs en JS. gdal-async para operaciones pesadas. |
| **Acceso Satelital** | stac-js + fetch | Cliente STAC nativo en JS para Copernicus. |
| **Meteorologia** | openmeteo (npm) + fetch | Cliente Open-Meteo para Node.js. |
| **Base de Datos** | PostgreSQL 16 + PostGIS + pgvector | Misma DB que Opcion A. TypeORM o Drizzle como ORM con soporte geoespacial. |
| **RAG / Embeddings** | LangChain.js + pgvector | LangChain tiene port completo a TypeScript. Embeddings con Transformers.js o API externa. |
| **LLM Inference** | Ollama (dev) / vLLM (prod) | Mismo que Opcion A. El LLM server es independiente del lenguaje del backend. |
| **Orquestacion IA** | LangGraph.js | Port oficial de LangGraph a TypeScript. Funcional pero menos maduro que la version Python. |
| **WhatsApp** | whatsapp-web.js o Twilio SDK (Node) | Twilio tiene SDK nativo de Node.js de primera clase. |
| **Frontend Web** | Nuxt 3 (Vue) o Next.js (React) | Comparte tipos TypeScript con el backend. Monorepo natural. |
| **Cache** | Redis | Mismo uso que Opcion A. |
| **Object Storage** | MinIO | Mismo que Opcion A. AWS SDK v3 para Node.js es excelente. |
| **Reverse Proxy** | Caddy o Traefik | Mismo que Opcion A. |

### Pros
- **Un solo lenguaje**: TypeScript end-to-end. Tipos compartidos entre backend y frontend. Monorepo con turborepo/nx.
- **Concurrencia I/O**: Node.js event loop maneja miles de conexiones concurrentes (WhatsApp webhooks, API requests) mejor que Python.
- **BullMQ mas simple que Celery**: Menos piezas moviles, dashboard integrado, retry/backoff nativo.
- **Ecosistema frontend maduro**: React/Vue/Nuxt son ciudadanos de primera clase.
- **Alineado con tu experiencia**: Tu infra actual (platform-infra) es TypeScript. Consistencia de stack.

### Contras
- **Ecosistema geoespacial inmaduro**: geotiff.js y gdal-async existen pero son 10x menos documentados que rasterio/GDAL en Python. Bugs edge-case en procesamiento raster.
- **Ecosistema ML limitado**: No hay equivalente de scikit-learn, PyTorch en JS. Para modelos custom de rendimiento/plagas, necesitas Python de todas formas.
- **LangGraph.js menos maduro**: La version TypeScript de LangGraph tiene menos ejemplos, menos community support, y features llegan 2-3 meses despues que en Python.
- **Procesamiento numerico**: No hay numpy en JS. Alternativas (ndarray, scijs) son ordenes de magnitud menos capaces.
- **Talento AI/ML**: Los data scientists y ML engineers trabajan en Python. Contratar talento TS para geoespacial + ML es dificil.

### Costo estimado VPS
- Similar a Opcion A. Node.js usa menos RAM base pero GDAL bindings consumen lo mismo.

---

## OPCION C — Hibrido Python + TypeScript (Recomendada)

### Filosofia
Cada lenguaje donde brilla. Python para el cerebro (IA, satelital, ML). TypeScript para el cuerpo (API, WhatsApp, web, orquestacion de requests). Comunicacion via queue (Redis/BullMQ) y API interna.

### Arquitectura

```
                    +------------------+
                    |   Nuxt 3 / PWA   |  <-- Frontend (TypeScript)
                    +--------+---------+
                             |
                    +--------v---------+
                    |  NestJS API       |  <-- API Gateway (TypeScript)
                    |  - Auth/Sessions  |
                    |  - WhatsApp hooks |
                    |  - REST/GraphQL   |
                    +--------+---------+
                             |
                    +--------v---------+
                    |  Redis (BullMQ)   |  <-- Message Queue
                    +--+-----+-----+---+
                       |     |     |
              +--------+  +--+--+  +--------+
              v           v     v           v
     +--------+--+ +------+----+ +----------+-+
     | Worker:    | | Worker:   | | Worker:    |
     | Satelital  | | AI/LLM    | | Meteo      |
     | (Python)   | | (Python)  | | (Python)   |
     | rasterio   | | LangGraph | | Open-Meteo |
     | GDAL       | | vLLM      | | CHIRPS     |
     | numpy      | | pgvector  | | ERA5       |
     +------------+ +-----------+ +------------+
                             |
                    +--------v---------+
                    |  PostgreSQL 16    |
                    |  + PostGIS        |
                    |  + pgvector       |
                    +------------------+
                    |  MinIO (S3)       |
                    +------------------+
```

### Componentes

| Capa | Tecnologia | Lenguaje | Justificacion |
|------|-----------|----------|---------------|
| **API Gateway** | NestJS | TypeScript | Maneja requests HTTP, webhooks WhatsApp, auth, rate limiting. Event loop de Node.js ideal para I/O concurrente. |
| **Frontend** | Nuxt 3 (Vue 3) | TypeScript | PWA con offline support. SSR para SEO del dashboard. Comparte tipos con NestJS. |
| **Task Queue** | BullMQ + Redis | TypeScript (producer) / Python (consumer) | NestJS encola jobs. Python workers los consumen. BullMQ soporta workers en cualquier lenguaje via protocolo Redis. |
| **Worker Satelital** | Python: rasterio + GDAL + numpy + xarray | Python | Descarga Sentinel-2/1, calcula NDVI/NDWI/EVI/SAVI, genera tiles, detecta nubes. |
| **Worker AI/LLM** | Python: LangGraph + LangChain + vLLM | Python | Orquestacion de agentes, RAG, chatbot. LangGraph en Python es la version mas madura y documentada. |
| **Worker Meteo** | Python: httpx + pandas | Python | Consulta Open-Meteo, CHIRPS, ERA5. Calcula ETo Penman-Monteith. |
| **LLM Server** | vLLM (prod) / Ollama (dev) | Python | Servidor de inferencia separado. Llama 3.1 8B quantizado (GPTQ/AWQ). |
| **Base de Datos** | PostgreSQL 16 + PostGIS + pgvector | SQL | Una DB para todo: datos relacionales, geometrias de parcelas, embeddings vectoriales. |
| **WhatsApp** | Twilio WhatsApp Business API | TypeScript (NestJS) | Webhooks recibidos en NestJS, procesados y encolados a workers Python. |
| **Object Storage** | MinIO | - | GeoTIFFs cacheados, PDFs de reportes, assets. |
| **Cache** | Redis 7 | - | Cache de tiles, sesiones, queue de BullMQ. |
| **Monitoreo** | Prometheus + Grafana | - | Metricas de workers, latencia API, uso de GPU/RAM. |
| **Reverse Proxy** | Caddy | - | TLS automatico, routing a NestJS + Nuxt. |

### Patron de comunicacion

1. **Request WhatsApp llega** → NestJS recibe webhook → valida → encola job en BullMQ
2. **Worker Python toma el job** → consulta LLM via vLLM → consulta pgvector para RAG → genera respuesta
3. **Worker devuelve resultado a Redis** → NestJS lo lee → envia respuesta via Twilio WhatsApp API
4. **Procesamiento satelital** → Cron job (node-cron en NestJS) encola descarga cada 5 dias → Worker Python descarga, procesa, almacena en MinIO + PostGIS
5. **Dashboard web** → Nuxt 3 consulta NestJS API → NestJS consulta PostgreSQL/PostGIS → renderiza mapas con MapLibre GL JS

### Pros
- **Lo mejor de ambos mundos**: Python para ciencia (geoespacial, ML, LLM). TypeScript para ingenieria web (API, frontend, webhooks).
- **Ecosistema geoespacial completo**: rasterio, GDAL, numpy, xarray, scikit-learn — sin compromisos.
- **Ecosistema web completo**: NestJS + Nuxt 3 + tipos compartidos — sin compromisos.
- **Escalabilidad natural**: Workers Python escalan independientemente del API. Si el procesamiento satelital es lento, agregas otro worker sin tocar el API.
- **LangGraph maduro**: La version Python de LangGraph tiene 10x mas ejemplos, documentacion, y community support.
- **Contratacion flexible**: Frontend/API devs en TypeScript. Data/ML engineers en Python. Cada quien en su zona de expertise.
- **Alineado con tu stack existente**: Tu infra (platform-infra) es TypeScript. Tu Data Plane ya tiene PostgreSQL+pgvector, Redis, MinIO. Reutilizas conocimiento.

### Contras
- **Dos runtimes**: Necesitas gestionar dependencias Python (venv/poetry) y Node.js (npm/pnpm). Docker mitiga esto.
- **Complejidad de deploy**: Mas contenedores (NestJS, Nuxt, workers Python, vLLM, PostgreSQL, Redis, MinIO). Docker Compose o Coolify lo simplifica.
- **Debugging cross-language**: Un bug que cruza la frontera TS→Redis→Python requiere tracing distribuido (OpenTelemetry).

### Costo estimado VPS
- **MVP sin LLM local**: CX41 (8 vCPU, 16GB RAM) — ~EUR 16/mes. LLM via API externa (Groq, Together.ai).
- **MVP con LLM local**: CCX33 (8 vCPU dedicados, 32GB RAM) — ~EUR 55/mes.
- **Produccion**: CCX43 (16 vCPU, 64GB RAM) + GPU cloud para LLM si es necesario — ~EUR 110/mes + GPU on-demand.

---

## COMPARATIVA FINAL

| Criterio | Opcion A (Python) | Opcion B (TypeScript) | Opcion C (Hibrido) |
|----------|-------------------|----------------------|-------------------|
| Geoespacial/Satelital | ★★★★★ | ★★☆☆☆ | ★★★★★ |
| ML / AI / LLM | ★★★★★ | ★★☆☆☆ | ★★★★★ |
| API / Webhooks / I/O | ★★★☆☆ | ★★★★★ | ★★★★★ |
| Frontend / PWA | ★★★☆☆ | ★★★★★ | ★★★★★ |
| Simplicidad operacional | ★★★★☆ | ★★★★☆ | ★★★☆☆ |
| Contratacion talento | ★★★★☆ | ★★★☆☆ | ★★★★★ |
| Escalabilidad | ★★★☆☆ | ★★★★☆ | ★★★★★ |
| Time-to-MVP | ★★★★☆ | ★★★☆☆ | ★★★★☆ |
| Madurez ecosistema agro | ★★★★★ | ★★☆☆☆ | ★★★★★ |

---

## RECOMENDACION: OPCION C (Hibrido)

La Opcion C es la recomendada por las siguientes razones:

1. **No hay compromiso en lo critico**: El procesamiento satelital y la IA son el core del producto. Hacerlos en un lenguaje suboptimo (JS para geoespacial) es deuda tecnica desde el dia 1.

2. **La API y WhatsApp son I/O-bound**: Node.js/NestJS maneja miles de webhooks concurrentes con un solo proceso. Python (FastAPI) puede hacerlo pero requiere mas tuning (uvicorn workers, async discipline).

3. **Escalabilidad por diseno**: Los workers Python son stateless y escalables horizontalmente. Si manana necesitas procesar 10x mas parcelas, agregas workers sin tocar el API.

4. **El LLM server es independiente**: vLLM/Ollama expone una API OpenAI-compatible. Da igual si el consumer es TypeScript o Python. Esto permite mover el LLM a un servidor dedicado (o GPU cloud) sin cambiar codigo.

5. **Docker Compose lo unifica**: En un VPS Hetzner, todo corre en Docker Compose. Un `docker-compose.yml` con 6-7 servicios es manejable y reproducible.

6. **Alineado con el mercado**: Los mejores agritech SaaS (EOSDA, Farmonaut, Kilimo) usan Python para procesamiento + framework web separado para la interfaz. Es el patron probado de la industria.

### Stack minimo para MVP (Fase 1)

```yaml
# docker-compose.yml conceptual
services:
  api:        # NestJS - API Gateway + WhatsApp webhooks
  web:        # Nuxt 3 - PWA Dashboard
  worker-sat: # Python - Procesamiento satelital
  worker-ai:  # Python - LangGraph + RAG
  llm:        # Ollama - LLM inference (dev)
  db:         # PostgreSQL 16 + PostGIS + pgvector
  redis:      # Redis 7 - Cache + BullMQ
  minio:      # MinIO - Object storage
  caddy:      # Reverse proxy + TLS
```

### Alternativa LLM sin GPU local

Si el VPS no tiene GPU, usar API externa para LLM:
- **Groq**: Llama 3.1 8B a ~$0.05/1M tokens. Latencia <500ms. Gratis hasta 30 req/min.
- **Together.ai**: Llama 3.1 8B a ~$0.10/1M tokens. Mas modelos disponibles.
- **OpenRouter**: Agregador de multiples providers. Fallback automatico.

Esto elimina la necesidad de RAM/GPU para LLM y reduce el VPS minimo a CX41 (8 vCPU, 16GB, ~EUR 16/mes).
