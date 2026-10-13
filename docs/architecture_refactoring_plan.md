# Plan de Refactorización y Automatización: MediFinderBot (2026)

## 1. Diagnóstico del Cuello de Botella (¿Por qué tardaba 2 a 3 días?)

Tras analizar en detalle el código en `medifinder-ingestor` (`services/etl.py` y `models/database.py`), se identificó la causa exacta de la lentitud extrema:

```
[Archivo TXT / Excel] 
        │
        ▼ (Iteración fila por fila)
┌─────────────────────────────────────────────────────────────────┐
│ Por CADA registro (ej. 200,000 filas):                          │
│  1. SELECT region_id FROM regions ...                           │
│  2. SELECT center_id FROM medical_centers ...                   │
│  3. UPDATE / INSERT medical_centers ...                         │
│  4. SELECT type_id FROM product_types ...                       │
│  5. SELECT product_id FROM products ...                         │
│  6. UPDATE / INSERT products ...                                │
│  7. INSERT INTO inventory ... ON CONFLICT DO UPDATE             │
│                                                                 │
│  * Cada consulta abre cursor, ejecuta y hace conn.commit()      │
│  * Todo viajando a través de un SSH Tunnel (latencia 150-250ms) │
└─────────────────────────────────────────────────────────────────┘
```

### Cálculo del impacto:
* **Consultas por fila**: Entre 5 y 8 peticiones SQL secuenciales por cada registro.
* **Autocommit individual**: `with self.get_cursor()` hace commit de la transacción en cada micro-operación.
* **Latencia de red del túnel SSH**: Cada round-trip añade ~150-250ms.
* **Resultado**: $1.2\text{ a }1.6\text{ segundos por registro} \times 200,000\text{ registros} \approx 240,000\text{ a }320,000\text{ segundos} \approx \mathbf{66\text{ a }90\text{ horas (2.8 a 3.7 días)}}$.

---

## 2. Nueva Arquitectura del Pipeline ETL (De 3 días a < 60 segundos)

El paradigma debe cambiar de **procesamiento transaccional fila por fila** a **procesamiento por lotes basado en conjuntos (Set-Based Bulk Ingestion)**.

```
┌────────────────────────────────────────────────────────┐
│  Descarga Automatizada (Playwright / HTTP Session)     │
│  Archivos Excel / CSV descargados localmente o a /tmp  │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  Carga y Normalización en Memoria (Polars / DuckDB)    │
│  - Limpieza y tipado en microsegundos                  │
│  - Extracción de dimensiones únicas (Regiones, etc.)   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  Sincronización de Dimensiones (In-Memory Cache)       │
│  - Pre-cargar IDs existentes en memoria en 1 SELECT    │
│  - Bulk insert de solo nuevas entidades                │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  PostgreSQL COPY a Tabla Temporal Staging              │
│  - Volcado de 200k filas vía streaming en ~3-5 seg     │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│  Single SQL MERGE / UPSERT                             │
│  - Una sola consulta SQL hace el INSERT/UPDATE masivo  │
│  - Tiempo total estimado: 30 a 60 segundos             │
└────────────────────────────────────────────────────────┘
```

### Componentes clave de la optimización:
1. **In-Memory Dimension Lookup**: Pre-cargar en memoria los catálogos (hay apenas unas 25 regiones, cientos de centros y miles de productos). En lugar de hacer miles de `SELECT`, la resolución del ID se hace en memoria en $O(1)$.
2. **PostgreSQL `COPY`**: Método más veloz de PostgreSQL para transferir datos hacia una tabla de staging temporal (`temp_staging_inventory`).
3. **Set-Based Upsert**:
   ```sql
   INSERT INTO inventory (center_id, product_id, current_stock, ...)
   SELECT center_id, product_id, current_stock, ...
   FROM temp_staging_inventory
   ON CONFLICT (center_id, product_id, report_date) 
   DO UPDATE SET current_stock = EXCLUDED.current_stock, updated_at = CURRENT_TIMESTAMP;
   ```

---

## 3. Automatización de la Descarga de Excel con Login (2026)

Para descargar los reportes de las webs del gobierno (ej. SISMED / DIGEMID / MINSA):

### Alternativa A: Playwright (Recomendada)
* **Por qué**: Sitios gubernamentales a menudo usan ASP.NET, JavaScript, iframes o botones que disparan eventos dinámicos.
* **Ventajas**:
  - Manejo nativo de eventos de descarga:
    ```python
    with page.expect_download() as download_info:
        page.locator("#btnExportarExcel").click()
    download = download_info.value
    download.save_as("data/inventory_latest.xlsx")
    ```
  - Persistencia de sesión (`storage_state="auth.json"`): No necesita escribir usuario y contraseña en cada ejecución si la cookie sigue vigente.
  - Ejecutable en modo headless sin interfaz gráfica en cualquier contenedor o runner.

### Alternativa B: Sesión HTTP Ligera (`httpx` o `curl_cffi`)
* Si el login es un formulario HTTP POST estándar con cookies y CSRF tokens, una llamada directa vía `httpx.Client()` es instantánea, sin el peso del navegador.

---

## 4. Orquestación y Ejecución Programada

Para que todo funcione sin intervención manual y sin costo de servidores 24/7:

| Opción | Infraestructura | Costo | Ventajas |
| :--- | :--- | :--- | :--- |
| **Opción 1: GitHub Actions (Recomendada)** | Workflow con cron (`schedule`) | **Gratis** | No requiere configurar servidores. Los secretos (credenciales web y DB) se guardan en GitHub Secrets. Corre Playwright + Python en contenedores limpios. |
| **Opción 2: GCP Cloud Run Job + Cloud Scheduler** | Contenedor en Cloud Run ejecutado por trigger | **Prácticamente gratis** (< $0.10/mes) | Se integra con la infraestructura existente de Google Cloud mencionada en el proyecto. |

---

## 5. Hoja de Ruta de Implementación Propuesta

### Fase 1: Scraper & Downloader Automatizado (`medifinder-scraper`)
- [ ] Desarrollar script con Playwright / httpx con soporte de login y descarga automática de los Excel gubernamentales.
- [ ] Implementar manejo de rotación de credenciales y reintentos ante caídas del portal.

### Fase 2: Rediseño del Motor ETL (`medifinder-ingestor`)
- [ ] Reemplazar lectura línea por línea con Polars / DuckDB para parsing ultrarrápido de Excel/TXT.
- [ ] Implementar caché en memoria de dimensiones (`regions`, `medical_centers`, `products`).
- [ ] Reemplazar el bucle de inserción individual por `COPY` hacia tabla temporal y un único `UPSERT` masivo.

### Fase 3: Integración del Pipeline y Pruebas Locales
- [ ] Conectar la descarga del Excel con la ejecución del ETL optimizado en un solo comando: `python pipeline.py`.
- [ ] Medir tiempos de ejecución y validar integridad de datos contra la base de datos.

### Fase 4: Automatización CI/CD y Programación
- [ ] Crear el workflow de GitHub Actions (`.github/workflows/daily_ingest.yml`) o Cloud Run Job con schedule diario/semanal.
- [ ] Configurar alertas vía Webhook (Telegram / Discord / Correo) ante fallos de scraping o actualización.
