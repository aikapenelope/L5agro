# AgroInteligencia VE — Documento Final: Stack, Capacidades y Arquitectura

## QUE ES AGROINTELIGENCIA VE

Plataforma SaaS de inteligencia artificial para el productor agricola venezolano. Convierte datos satelitales gratuitos, meteorologicos abiertos y registros del productor en decisiones accionables entregadas por WhatsApp. Sin hardware propio, sin internet de alta velocidad.

---

## STACK FINAL DEFINITIVO

### Servicios (9 contenedores Docker)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                         │
│                            |                                             │
│                     ┌──────v──────┐                                      │
│                     │   Caddy     │  Reverse proxy + TLS automatico      │
│                     └──┬───┬───┬──┘                                      │
│                        |   |   |                                         │
│          ┌─────────────┘   |   └─────────────┐                          │
│          v                 v                  v                          │
│  ┌───────────────┐ ┌──────────────┐ ┌────────────────┐                  │
│  │ Nuxt 3 PWA    │ │ NestJS API   │ │ Agno AgentOS   │                  │
│  │ (TypeScript)  │ │ (TypeScript) │ │ (Python)       │                  │
│  │               │ │              │ │                │                  │
│  │ Dashboard web │ │ Auth, CRUD   │ │ WhatsApp bot   │                  │
│  │ Mapas         │ │ Gateway datos│ │ Agentes IA     │                  │
│  │ PWA offline   │ │ Billing      │ │ RAG agronomico │                  │
│  └───────────────┘ └──────┬───────┘ └───────┬────────┘                  │
│                           |                  |                          │
│                    ┌──────v──────┐    ┌──────v──────┐                   │
│                    │ PostgreSQL  │    │ SQLite      │                   │
│                    │ + PostGIS   │    │ (agno.db)   │                   │
│                    │             │    ├─────────────┤                   │
│                    │ app_db      │    │ LanceDB     │                   │
│                    │ geo_db      │    │ (vectores)  │                   │
│                    └─────────────┘    └─────────────┘                   │
│                                                                          │
│  ┌───────────────┐ ┌──────────────┐ ┌────────────────┐                  │
│  │ Prefect       │ │ Prefect      │ │ Redis 7        │                  │
│  │ Server        │ │ Worker       │ │                │                  │
│  │               │ │              │ │ db0: BullMQ    │                  │
│  │ Orquestacion  │ │ Satelital    │ │ db1: Sesiones  │                  │
│  │ de workflows  │ │ Meteorologia │ │ db2: Cache     │                  │
│  │ (SQLite)      │ │ Reportes PDF │ │                │                  │
│  └───────────────┘ └──────────────┘ └────────────────┘                  │
│                                                                          │
│  ┌────────────────┐                                                      │
│  │ MinIO          │  Object storage S3-compatible                        │
│  │ 6 buckets      │  GeoTIFFs, reportes, docs RAG, uploads              │
│  └────────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Bases de datos — Cada servicio con la suya

| Servicio | Base de datos | Tipo | Contenido |
|----------|--------------|------|-----------|
| Agno (sesiones/memoria) | SQLite `agno.db` | Archivo | Sesiones de chat, memorias por usuario, metricas |
| Agno (RAG knowledge) | LanceDB `/data/lancedb/` | Directorio | Embeddings de agronomia venezolana (INIA, plagas, cultivos) |
| NestJS (negocio) | PostgreSQL `app_db` + PostGIS | Servidor | Usuarios, parcelas (geometrias), cooperativas, suscripciones |
| Prefect Worker (geodatos) | PostgreSQL `geo_db` + PostGIS | Servidor | NDVI/NDWI historico, cache meteo, datos de suelo |
| Prefect Server (workflows) | SQLite `prefect.db` | Archivo | Estado de flows, historial de ejecuciones |
| NestJS (sesiones web) | Redis db 1 | In-memory | Sesiones de dashboard, cache de API |
| BullMQ (queues) | Redis db 0 | In-memory | Jobs rapidos: alertas, notificaciones |

Cero bases de datos compartidas entre servicios.

### Dependencias externas

| Servicio | Costo | Uso |
|----------|-------|-----|
| Copernicus STAC API | Gratis | Sentinel-2/1 imagenes satelitales |
| Open-Meteo API | EUR 300/mes (comercial) | Meteorologia, ETo, pronostico |
| CHIRPS | Gratis | Precipitacion historica |
| Meta WhatsApp Business API | ~$0.05/conversacion | Canal principal de comunicacion |
| Groq (MVP) / vLLM (prod) | ~$5/mes para 50 usuarios | Inferencia LLM |
| SoilGrids (ISRIC) | Gratis | Datos de suelo (descarga unica) |

### Recursos VPS

| Fase | VPS Hetzner | RAM | Costo |
|------|-------------|-----|-------|
| MVP (50 usuarios, LLM externo) | CX32 (4 vCPU, 8GB) | ~3GB usados | EUR 8/mes |
| Fase 2 (500 usuarios, Ollama local) | CCX33 (8 vCPU, 32GB) | ~12GB usados | EUR 55/mes |
| Produccion (1000+ usuarios, vLLM) | CCX43 + GPU cloud | ~20GB+ | EUR 110+/mes |

---

## TODO LO QUE PUEDE HACER LA PLATAFORMA

### 16 capacidades organizadas por como las experimenta el usuario

#### VIA WHATSAPP (canal principal — 90% de la interaccion)

**1. Chatbot agronomico inteligente**
- El productor escribe: "por que se estan amarillando mis matas de maiz?"
- Agno consulta: RAG agronomico (LanceDB) + datos de SU parcela (NDVI via NestJS) + clima actual (Open-Meteo)
- Responde con diagnostico contextualizado: "Tu parcela en Portuguesa muestra NDVI de 0.45 (bajo para maiz en V8). Con las lluvias de los ultimos 5 dias y tu suelo franco-arcilloso, es probable deficiencia de nitrogeno por lixiviacion. Recomendacion: aplicar 50kg/ha de urea al voleo."

**2. Alertas automaticas de clima**
- Prefect Worker consulta Open-Meteo cada 6 horas
- Si detecta lluvia >50mm en proximas 48h para la zona de una parcela → encola alerta en BullMQ
- NestJS consume la alerta → pide a Agno generar mensaje personalizado
- Agno envia por WhatsApp: "Alerta: se esperan lluvias fuertes (65mm) entre jueves y viernes en tu zona de Acarigua. Si tienes maiz en floracion, considera posponer la aplicacion de fertilizante."

**3. Alertas de estres hidrico**
- Prefect Worker calcula ETo Penman-Monteith + compara con NDWI satelital
- Si deficit hidrico >30% respecto al optimo del cultivo → alerta
- WhatsApp: "Tu parcela de arroz en lote 3 muestra deficit hidrico. Recomendacion: regar 25mm en las proximas 48 horas. Proximo riego optimo: lunes."

**4. Alertas de plagas preventivas**
- Prefect Worker correlaciona temperatura + humedad + fase fenologica con modelos de ciclo de plagas
- 10-14 dias antes de condiciones optimas de infestacion → alerta
- WhatsApp: "Condiciones favorables para sogata del arroz en tu zona en los proximos 10 dias (temperatura 28-32C + humedad >80%). Revisa trampas y considera aplicacion preventiva de imidacloprid."

**5. Recomendaciones de riego semanales**
- Cada lunes, Prefect flow genera recomendacion de riego por parcela
- Calcula: ETo de la semana - lluvia efectiva = requerimiento neto
- WhatsApp: "Programa de riego semana 15-21 abril: Lote 1 (maiz V10): 30mm martes + 25mm viernes. Lote 2 (arroz): mantener lamina, no regar (lluvia esperada miercoles)."

**6. Reporte semanal automatico**
- Cada domingo, Prefect flow genera PDF con WeasyPrint
- Contenido: mapa NDVI de cada parcela, comparacion vs semana anterior, alertas atendidas, clima de la semana, recomendaciones
- WhatsApp: envia PDF adjunto al productor y al gerente de cooperativa

**7. Onboarding por WhatsApp**
- Productor nuevo envia "Hola" al numero de AgroInteligencia
- Agno guia el registro: "Bienvenido. Envia tu ubicacion GPS para registrar tu primera parcela."
- Productor envia ubicacion → Agno crea parcela en app_db via NestJS API
- "Que cultivo tienes en esta parcela?" → "Maiz" → registrado
- Desde ese momento, el sistema empieza a monitorear esa parcela

**8. Consulta de estado de parcela**
- "Como esta mi parcela?" → Agno consulta NDVI actual + clima + alertas activas
- Responde con resumen: "Tu maiz en Lote 1 esta saludable (NDVI 0.78, arriba del promedio para tu zona). No hay alertas activas. Proxima imagen satelital: miercoles."

**9. Micro-lecciones de extension agricola (AgroEscuela)**
- El sistema detecta fase fenologica del cultivo (via NDVI temporal)
- Envia automaticamente leccion relevante: "Tu maiz esta entrando en floracion. Tip: este es el momento critico para riego. Un deficit de agua ahora reduce el rendimiento hasta 50%."
- Fase 2: notas de voz generadas con Piper TTS

**10. Reporte de precios crowdsourced**
- "Cuanto esta el maiz en Acarigua?" → Agno consulta datos reportados por otros productores
- "El maiz blanco se reporto a $0.35/kg en Acarigua hace 2 dias. En Barinas esta a $0.32/kg."
- El productor puede reportar: "Vendi maiz a 0.38 en Turén" → se registra en app_db

#### VIA DASHBOARD WEB (cooperativas y productores con acceso web)

**11. Mapa interactivo de parcelas**
- Nuxt 3 + MapLibre GL JS renderiza parcelas con overlay de NDVI
- Colores: verde (saludable) → amarillo (estres leve) → rojo (estres severo)
- Click en parcela → detalle: NDVI actual, historico 12 meses, clima, alertas
- Tiles servidos desde MinIO (satellite-processed)

**12. Dashboard de cooperativa**
- Vista agregada de todas las parcelas de los miembros
- Metricas: hectareas monitoreadas, alertas emitidas, agua ahorrada estimada
- Ranking de parcelas por salud (NDVI)
- Reporte mensual descargable en PDF

**13. Gestion de parcelas y cultivos**
- CRUD de parcelas: dibujar poligono en mapa o subir GeoJSON
- Registrar ciclo de cultivo: fecha siembra, variedad, insumos aplicados
- Historial de cada parcela: todos los indices, alertas, recomendaciones

**14. Estimacion de rendimiento**
- Grafico de NDVI temporal vs baseline historico de la zona
- "Tu parcela esta 15% por debajo del promedio para maiz en Portuguesa a esta fecha"
- Fase 2: modelo ML calibrado con datos acumulados

#### VIA PREFECT (procesos batch automaticos — invisibles al usuario)

**15. Pipeline satelital cada 5 dias**
```
Buscar Sentinel-2 → Filtrar nubes → Fallback Sentinel-1 SAR →
Descargar → Recortar por parcela → Calcular NDVI/NDWI/EVI →
Detectar anomalias → Almacenar en MinIO + geo_db →
Si anomalia → generar alerta → BullMQ → WhatsApp
```

**16. Pipeline de ingestion RAG**
```
Listar docs nuevos en MinIO → Extraer texto (PyMuPDF) →
Chunking semantico → Embeddings (nomic-embed-text via Ollama) →
Indexar en LanceDB → Disponible para chatbot
```

---

## COMO FUNCIONA: FLUJO COMPLETO

### Flujo 1: Productor hace pregunta por WhatsApp

```
1. Productor envia "por que mi maiz esta amarillo?" via WhatsApp
2. Meta Webhook → Caddy → Agno AgentOS (puerto 8000)
3. Agno identifica user_id por numero de telefono
4. Agno ejecuta el agente agronomico:
   a. Tool: obtener_ndvi_parcela(user_id) → HTTP GET → NestJS → geo_db
      Resultado: "NDVI 0.45, bajo para maiz V8"
   b. Tool: obtener_clima(user_id) → HTTP GET → NestJS → geo_db
      Resultado: "Lluvia acumulada 85mm ultimos 7 dias"
   c. Knowledge search: LanceDB → "deficiencia nitrogeno en maiz"
      Resultado: chunks relevantes de guias INIA
   d. LLM (Groq API): combina todo → genera respuesta contextualizada
5. Agno envia respuesta via Meta WhatsApp API
6. Agno guarda sesion en SQLite, actualiza memoria del usuario
```

**Latencia total**: ~3-5 segundos (dominado por LLM inference)

### Flujo 2: Alerta automatica de clima

```
1. Prefect schedule: cada 6 horas ejecuta flow "monitoreo-clima"
2. Prefect Worker consulta Open-Meteo para todas las parcelas activas
3. Detecta: lluvia >50mm pronosticada para zona de parcela #47
4. Worker escribe alerta en geo_db + encola job en BullMQ (Redis db 0)
5. NestJS consume job de BullMQ
6. NestJS llama a Agno: "genera mensaje de alerta para usuario X, parcela Y, evento Z"
7. Agno genera mensaje personalizado con LLM
8. Agno envia via WhatsApp
```

**Latencia**: Minutos (batch, no interactivo)

### Flujo 3: Procesamiento satelital

```
1. Prefect schedule: cada 5 dias ejecuta flow "pipeline-satelital"
2. Prefect Worker:
   a. Lista parcelas activas desde NestJS API
   b. Busca imagenes Sentinel-2 en Copernicus STAC API
   c. Filtra por nubosidad (<30%)
   d. Si no hay imagen limpia → busca Sentinel-1 SAR
   e. Descarga GeoTIFF → MinIO (satellite-raw)
   f. Recorta por poligono de parcela (rasterio + PostGIS)
   g. Calcula NDVI, NDWI, EVI, SAVI (numpy)
   h. Compara con historico → detecta anomalias
   i. Genera tiles PNG → MinIO (satellite-processed)
   j. Escribe indices en geo_db (ndvi_history, etc.)
   k. Si anomalia detectada → encola alerta en BullMQ
3. Dashboard web muestra nuevos datos automaticamente
```

### Flujo 4: Registro de nuevo productor

```
1. Productor envia "Hola" a WhatsApp de AgroInteligencia
2. Agno detecta user_id nuevo (no existe en SQLite)
3. Agno inicia flujo de onboarding:
   - "Bienvenido! Como te llamas?"
   - "Envia la ubicacion GPS de tu parcela"
   - Productor envia ubicacion → Agno extrae lat/lon
   - Agno llama tool: crear_parcela(nombre, lat, lon) → NestJS API → app_db
   - "Que cultivo tienes?" → "Maiz" → registrado
   - "Listo! Ya estoy monitoreando tu parcela. Te enviare alertas y recomendaciones."
4. Prefect incluye la nueva parcela en el proximo ciclo satelital
```

### Flujo 5: Cooperativa consulta dashboard

```
1. Gerente de cooperativa abre app.agrointeligencia.ve
2. Caddy → Nuxt 3 (SSR)
3. Login → NestJS API → app_db (auth)
4. Dashboard carga:
   a. Mapa de parcelas: NestJS → app_db (geometrias) + geo_db (NDVI)
   b. Tiles NDVI: Nuxt → MinIO (satellite-processed)
   c. Alertas activas: NestJS → app_db (alerts_log)
   d. Metricas: NestJS → agrega datos de app_db + geo_db
5. Gerente descarga reporte PDF: NestJS → MinIO (reports bucket)
```

---

## FASES DE IMPLEMENTACION

### Fase 1 — MVP (3 meses) — 50 productores piloto

**Modulos activos**: Monitoreo satelital (NDVI) + Riego + Alertas clima + Chatbot basico + Onboarding WhatsApp

**Lo que se construye**:
- Agno con 3 agentes: Riego, Clima, General
- Prefect con 2 flows: pipeline satelital, monitoreo clima
- NestJS con CRUD basico: usuarios, parcelas, cooperativas
- Nuxt 3 con mapa de parcelas + NDVI
- RAG con ~500 documentos INIA curados
- WhatsApp como canal unico

**Costo mensual**: ~$50 USD (VPS EUR 8 + Groq ~$5 + WhatsApp ~$10 + Open-Meteo ~$30 amortizado)

### Fase 2 — Expansion (6 meses) — 500 productores

**Agrega**: Prediccion rendimiento + IPM plagas + Agentes autonomos + RAG completo + AgroEscuela + Gestion cooperativa + Reportes PDF

### Fase 3 — Monetizacion (12 meses) — 1000+ productores

**Agrega**: Trazabilidad/exportacion + AgroFinanzas + Inteligencia de mercado + Microseguro parametrico + Creditos de carbono MRV

---

## RESUMEN EN UNA FRASE

AgroInteligencia VE es un WhatsApp bot inteligente respaldado por satelites, que le dice al productor venezolano que hacer con su cultivo, cuando hacerlo, y por que — sin que el productor necesite internet rapido, smartphone caro, ni conocimiento tecnico.
