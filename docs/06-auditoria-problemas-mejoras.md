# AgroInteligencia VE — Auditoria del Stack: Problemas, Gaps y Mejoras

El stack actual cubre los 11 modulos propuestos. Este documento identifica 7 problemas concretos que hay que resolver para produccion con 20-50 usuarios multi-tenant, y 6 mejoras recomendadas.

---

## PROBLEMA 1 (CRITICO): Agno WhatsApp + Teams tiene bug de aislamiento multi-tenant

**Hallazgo**: Issue [#4825](https://github.com/agno-agi/agno/issues/4825) en el repo de Agno — cuando se usa `Team()` con la interfaz WhatsApp, todos los usuarios comparten la misma sesion. Mensajes del usuario A se mezclan con memorias del usuario B. Fix en PR #4827, pero revela que el multitenancy de Agno via WhatsApp es reciente y fragil.

**Impacto**: Con 20-50 productores, si el productor Juan pregunta "como esta mi maiz" y el sistema responde con datos de la parcela de Pedro, se pierde toda credibilidad.

**Mitigacion**:
- Usar `Agent()` en lugar de `Team()` para WhatsApp. El Agent aislado por phone_number funciona correctamente segun docs.
- Si se necesita multi-agente, orquestar internamente: el Agent principal llama a los agentes especializados como tools, no como Team members.
- Pinear la version de Agno al release que incluye el fix (#4827+).
- Testear aislamiento con 3+ numeros de WhatsApp antes de produccion.

---

## PROBLEMA 2: El RAG es compartido, no multi-tenant — y eso esta bien, pero falta contexto por finca

LanceDB tiene una sola knowledge base (`agronomia_ve`) que todos los usuarios consultan. Para conocimiento agronomico general (INIA, plagas, cultivos) esto es correcto — es el mismo para todos.

**Lo que falta**: Datos especificos por finca. Cuando el productor pregunta "por que se estan amarillando mis matas", el agente necesita contexto de SU parcela: su NDVI actual, su historial de riego, su tipo de suelo. Eso no esta en LanceDB — esta en `geo_db` y `app_db`.

**Solucion — RAG de dos capas**:

1. **Knowledge base global** (LanceDB): Agronomia venezolana, plagas, cultivos. Igual para todos. Ya esta resuelto.
2. **Contexto por usuario** (tools de Agno): El agente usa tools que consultan NestJS API con el `user_id` del productor para obtener datos de SU parcela. No es RAG — es tool calling con filtro por tenant.

```python
@tool
def obtener_ndvi_parcela(user_id: str, parcela_id: str) -> str:
    """Obtiene el NDVI actual y historico de la parcela del productor"""
    response = httpx.get(
        f"http://api:3000/internal/parcels/{parcela_id}/ndvi",
        headers={"X-User-Id": user_id}
    )
    return response.text

@tool
def obtener_clima_parcela(user_id: str, parcela_id: str) -> str:
    """Obtiene pronostico meteorologico para la ubicacion de la parcela"""
    response = httpx.get(
        f"http://api:3000/internal/parcels/{parcela_id}/weather",
        headers={"X-User-Id": user_id}
    )
    return response.text

@tool
def obtener_perfil_suelo(user_id: str, parcela_id: str) -> str:
    """Obtiene datos de suelo (pH, textura, materia organica) de la parcela"""
    response = httpx.get(
        f"http://api:3000/internal/parcels/{parcela_id}/soil",
        headers={"X-User-Id": user_id}
    )
    return response.text
```

LanceDB soporta `knowledge_filters` por metadata (ej: `{"user_id": "juan123"}`), pero para datos de parcela que cambian cada 5 dias, no tiene sentido vectorizarlos. Tool call directo es mas eficiente.

---

## PROBLEMA 3: Falta pipeline de ingestion para el RAG

Tenemos LanceDB como vector store y documentos fuente en MinIO (`knowledge-docs`). Pero no hay nada que conecte los dos.

**Lo que falta**:
1. **Loader de documentos**: Leer PDFs del INIA, HTMLs scrapeados desde MinIO
2. **Chunking**: Partir documentos en fragmentos semanticos (no por paginas)
3. **Embedding**: Generar vectores con `nomic-embed-text` via Ollama
4. **Indexing**: Insertar en LanceDB
5. **Actualizacion incremental**: Cuando se agrega un documento nuevo, re-indexar solo ese

**Solucion**: Flow de Prefect dedicado a ingestion RAG:

```python
from prefect import flow, task

@task(retries=2)
def listar_nuevos_docs(bucket: str = "knowledge-docs") -> list[str]:
    """Lista documentos en MinIO que no estan indexados en LanceDB"""
    # ... compara MinIO listing vs LanceDB metadata ...

@task
def procesar_documento(doc_path: str) -> list[dict]:
    """Lee PDF, aplica chunking semantico, genera embeddings"""
    # Agno Knowledge.insert() hace esto internamente
    # Pero para control fino, usar:
    # 1. PyMuPDF para extraer texto
    # 2. Semantic chunking (por similitud de embeddings entre parrafos)
    # 3. OllamaEmbedder para generar vectores

@task
def indexar_en_lancedb(chunks: list[dict]):
    """Inserta chunks con embeddings en LanceDB"""

@flow(name="rag-ingestion", log_prints=True)
def ingestar_documentos():
    nuevos = listar_nuevos_docs()
    for doc in nuevos:
        chunks = procesar_documento(doc)
        indexar_en_lancedb(chunks)
    print(f"Indexados {len(nuevos)} documentos nuevos")
```

Este flow corre como Prefect deployment con schedule (diario o on-demand). Agno solo lee de LanceDB. Prefect escribe.

**Seed data**: Al primer deploy, este flow debe correr con un set inicial de ~100-500 documentos curados del INIA para que el chatbot tenga conocimiento base desde el dia 1.

---

## PROBLEMA 4: NestJS necesita ser gateway de datos para Agno

Agno necesita datos de parcela (NDVI, clima, suelo) para responder preguntas contextualizadas. Esos datos estan en `geo_db` (PostgreSQL del worker). Pero Agno no tiene acceso directo a `geo_db` (principio de separacion).

**Solucion**: NestJS API como gateway unico de datos.

```
Agno tool call
    |
    v
HTTP GET http://api:3000/internal/parcels/{id}/ndvi
    |
    v
NestJS API
    |
    +--- app_db (owner) -----> users, parcels, farms
    |
    +--- geo_db (read-only) -> ndvi_history, weather_cache, soil_data
```

NestJS tiene:
- Usuario `app_user` con full access a `app_db` (su database)
- Usuario `geo_reader` con SELECT-only en `geo_db` (database del worker)

Endpoints internos (no expuestos a internet, solo red Docker):
- `GET /internal/parcels/:id/ndvi` — NDVI actual + historico
- `GET /internal/parcels/:id/weather` — Pronostico 14 dias
- `GET /internal/parcels/:id/soil` — Perfil de suelo
- `GET /internal/parcels/:id/alerts` — Alertas activas
- `GET /internal/users/:id/parcels` — Lista de parcelas del usuario

---

## PROBLEMA 5: No hay rate limiting para LLM

Con 50 usuarios en WhatsApp, si 10 mandan mensajes simultaneos, son 10 llamadas al LLM. Groq free tier: 30 req/min. Groq paid: 100 req/min. Ollama: secuencial.

**Solucion por fase**:
- **MVP (Groq free)**: Guardrail de Agno que limita a 1 request por usuario cada 10 segundos. Suficiente para 50 usuarios.
- **Fase 2 (Groq paid o Ollama)**: BullMQ como buffer. Mensajes WhatsApp se encolan, se procesan con concurrencia controlada (max 5 simultaneos).
- **Fase 3 (vLLM)**: Sin limite practico. vLLM maneja 128+ concurrentes.

---

## PROBLEMA 6: No hay generacion de reportes PDF

Los modulos mencionan "reportes semanales en PDF" para cooperativas. No hay nada en el stack que genere PDFs.

**Solucion**: WeasyPrint (Python) en un Prefect flow semanal.

```python
@flow(name="reporte-semanal")
def generar_reportes_cooperativas():
    cooperativas = obtener_cooperativas_activas()
    for coop in cooperativas:
        datos = recopilar_datos_semana(coop.id)  # NDVI, alertas, clima
        html = renderizar_template_jinja2(datos)  # Template HTML con graficos
        pdf = weasyprint.HTML(string=html).write_pdf()
        subir_a_minio(pdf, f"reports/{coop.id}/{fecha}.pdf")
        notificar_whatsapp(coop.gerente_phone, pdf_url)
```

WeasyPrint: ~50MB de dependencias. Corre dentro del Prefect Worker. No necesita servicio adicional.

---

## PROBLEMA 7: Falta buffer para webhooks de WhatsApp

Si Agno esta caido o reiniciando cuando llega un webhook de WhatsApp, Meta reintenta 3 veces y descarta el mensaje. El productor no recibe respuesta y no sabe por que.

**Solucion**: Caddy recibe el webhook y lo pasa a un endpoint de NestJS que lo encola en BullMQ (Redis db 0). Un consumer en Agno (o un bridge) procesa la queue. Si Agno esta caido, el mensaje espera en Redis hasta que vuelva.

```
Meta WhatsApp → Caddy → NestJS /webhook/whatsapp → BullMQ (Redis)
                                                        |
                                                        v
                                                  Agno consumer
```

Alternativa mas simple: Configurar Caddy con retry/buffering. Pero BullMQ da visibilidad (cuantos mensajes pendientes, cuantos fallidos).

---

## MEJORAS RECOMENDADAS

| Mejora | Que agrega | Fase | Esfuerzo |
|--------|-----------|------|----------|
| **Health checks Docker** | `healthcheck` para cada servicio. Caddy no rutea a servicios caidos. Restart automatico. | 1 | Bajo |
| **Backup automatizado** | Cron: `cp agno.db`, `cp -r lancedb/`, `pg_dump app_db`, `pg_dump geo_db` → MinIO o storage externo | 1 | Bajo |
| **Seed data RAG** | Script Prefect que descarga guias INIA publicas y las indexa en LanceDB al primer deploy | 1 | Medio |
| **Metricas por tenant** | Cuantos mensajes, consultas NDVI, tokens LLM consume cada productor. Para billing y deteccion de abuso. | 1 | Medio |
| **Webhook buffer** | BullMQ queue entre Caddy y Agno para no perder mensajes WhatsApp | 1 | Medio |
| **Logging centralizado** | Loki + Grafana para logs de todos los servicios | 2 | Medio |
| **TTS para AgroEscuela** | Piper TTS local para generar notas de voz en espanol | 2 | Medio |
| **Alertas de sistema** | Notificacion (email/Telegram) si un servicio cae, si Prefect flow falla, si disco >80% | 2 | Bajo |

---

## CHECKLIST: Esta listo para 20-50 usuarios?

| Requisito | Estado | Que falta |
|-----------|--------|-----------|
| Multi-tenant WhatsApp | Parcial | Pinear Agno post-fix, usar Agent no Team, testear |
| RAG agronomico | Parcial | Pipeline de ingestion (Prefect flow) + seed data |
| Datos por parcela en chatbot | No | Tools de Agno + endpoints internos NestJS |
| Procesamiento satelital | Si | Prefect + rasterio/GDAL |
| Alertas automaticas | Parcial | Falta bridge Prefect → Agno para disparar alertas |
| Dashboard web | Si | NestJS + Nuxt 3 |
| Reportes PDF | No | WeasyPrint en Prefect flow |
| Rate limiting LLM | No | Guardrail de Agno o BullMQ buffer |
| Backups | No | Script cron + MinIO |
| Monitoreo | Parcial | Agno tracing si, pero falta health checks y alertas |
