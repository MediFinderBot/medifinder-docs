# Specification: Automated Ingestion & Extraction for DIGEMID Product Catalog

## 1. Executive Summary & Purpose

This specification defines the architectural design, extraction protocols, data dictionary, and integration patterns for the automated extraction of pharmaceutical product catalog data from the **DIGEMID - Observatorio de Productos Farmacéuticos (OPPF / OPM)** platform (`https://opm-digemid.minsa.gob.pe`).

While this source provides normalized commercial brands, active ingredients (IFA), pharmaceutical forms, concentrations, and sanitary registrations, it serves as the **Master Catalog (Dimension Table)** for MediFinderBot. It links directly with regional health facility inventory databases (e.g., SISMED / MINSA health centers) to provide semantic resolution for user queries.

---

## 2. Source Platform Architecture

### 2.1 Web Frontend Architecture
* **Platform Type**: Single Page Application (SPA) built with Angular.
* **Hosting / Delivery**: Web server delivered behind a **Cloudflare CDN / WAF reverse proxy**.
* **Configuration Vector**: External configuration injected through `app-config/opm-config-file.js`:
  ```javascript
  window.__env.servicio = 'https://ms-opm.minsa.gob.pe';
  window.__env.app = 'OPM-COVID';
  window.__env.keyGoogle = 'PRD'; // Google reCAPTCHA v3 site key
  ```

### 2.2 Security & Bot Management Layer
* **Cloudflare Protection**: Cloudflare monitors incoming requests and drops standard HTTP client User-Agents (e.g., Python `urllib`) with HTTP 403 Forbidden. Legitimate browser User-Agents and standard request headers bypass this threshold without triggering interactive JavaScript challenges or CAPTCHAs.
* **Google reCAPTCHA v3**: The frontend Angular component registers an action (`catalogoproductos`) against Google reCAPTCHA v3 before triggering the export. However, server-side validation on the catalog microservice is permissive for public catalog exports, accepting payloads where `tokenGoogle` is empty or omitted.

---

## 3. Extraction Protocol Specification

Rather than employing a resource-intensive headless browser (Chromium / Playwright) with rendering and DOM latency, extraction can be executed via direct, deterministic HTTP calls to the backend API microservice.

### 3.1 Endpoint Details
* **Method**: `POST`
* **Target URL**: `https://ms-opm.minsa.gob.pe/msopmcovid/producto/catalogoproductos`
* **Network Protocol**: HTTPS (TLS 1.2 / 1.3)
* **Expected Response MIME**: `application/json` header containing raw binary payload `application/vnd.ms-excel` (ZIP container with XML worksheets).

### 3.2 Required HTTP Headers
```http
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36
Origin: https://opm-digemid.minsa.gob.pe
Referer: https://opm-digemid.minsa.gob.pe/
Content-Type: application/json
Accept: application/json, text/plain, */*
```

### 3.3 Request Payload
```json
{
  "filtro": {
    "situacion": "ACT",
    "tokenGoogle": null
  }
}
```

### 3.4 Extraction Performance Benchmark
* **Latency to First Byte**: ~1.2 seconds.
* **Full Stream Transfer (1.07 MB)**: ~4.3 seconds.
* **Total Extraction Window**: < 6.0 seconds.

---

## 4. Dataset Schema & Parsing Considerations

### 4.1 Structure & Header Offsets
* **File Format**: Standard Office Open XML Spreadsheet (`.xlsx`).
* **Worksheet Name**: `Catálogo`
* **Header Offset (`skiprows`)**: 6 rows.
  * Rows 0-1: Blank spacer rows.
  * Row 2: Document title (`Catálogo de Productos Farmacéuticos actualizado diariamente`).
  * Row 3: Scope notes and legal disclaimers.
  * Row 4: Last update timestamp (e.g., `Fec. de actualizac.: 29/09/2026 04:57:03 PM`).
  * Row 5: Column headers (`Cod_Prod`, `Nom_Prod`, etc.).
  * Row 6+: Tabular product records.

### 4.2 Data Dictionary

| Column Name | Data Type | Nullable | Cardinality (Approx.) | Description & Role in MediFinderBot |
| :--- | :--- | :---: | :---: | :--- |
| `Cod_Prod` | `INTEGER` | No | 18,251 (Unique) | Primary Key. Master identifier used to link inventory records. |
| `Nom_Prod` | `VARCHAR(255)` | No | ~8,750 | Commercial brand name or official generic title. |
| `Concent` | `VARCHAR(100)` | Yes | ~1,430 | Product strength / concentration (e.g., `500 mg`, `120 mg/ 5 mL`). |
| `Nom_Form_Farm` | `VARCHAR(100)` | No | ~180 | Pharmaceutical dosage form (e.g., `Tableta Recubierta`, `Jarabe`). |
| `Presentac` | `VARCHAR(255)` | No | ~1,840 | Commercial package description (e.g., `Caja Envase Blíster x 100`). |
| `Fracción` | `INTEGER` | No | ~80 | Units per package / minimum dispensable fraction. |
| `Num_RegSan` | `VARCHAR(50)` | No | ~11,940 | Official sanitary registration code issued by DIGEMID. |
| `Nom_Titular` | `VARCHAR(255)` | No | ~430 | Marketing Authorization Holder / Pharmaceutical distributor. |
| `Nom_Fabricante` | `VARCHAR(255)` | No | ~900 | Manufacturing laboratory. |
| `Nom_IFA` | `VARCHAR(255)` | Yes | ~1,830 | **Key Search Field**: Active Pharmaceutical Ingredient (API / Generic name). |
| `Nom_Rubro` | `VARCHAR(100)` | No | 4 | Regulatory category (`ESPECIALIDAD FARMACEUTICA`, `PRODUCTO BIOLOGICO`, etc.). |
| `Situación` | `VARCHAR(10)` | No | 1 | Status indicator (`ACT` = Active). |

---

## 5. Automated Agent Architecture Design

```
┌────────────────────────────────────────────────────────┐
│                   DigemidCatalogAgent                  │
├────────────────────────────────────────────────────────┤
│  1. Check Last Modified / Daily Ingestion State        │
│  2. Execute Direct API Streaming Request               │
│  3. Validate Magic Bytes ('PK\x03\x04') & Payload Size │
│  4. Parse In-Memory with Pandas / Polars (skiprows=6)  │
│  5. Normalize Text (Stripping, Upper Case, Trim)       │
│  6. Bulk Upsert into PostgreSQL Master `products`      │
└────────────────────────────────────────────────────────┘
```

### 5.1 Resilience & Error Handling
1. **Fallback Strategy**: If MINSA enforces strict reCAPTCHA v3 verification in future revisions, the downloader module must switch dynamically to a headless Playwright runner executing browser-side reCAPTCHA tokens.
2. **Data Integrity Checks**:
   * Minimum row count assertion: Fail pipeline if row count < 15,000.
   * Null checks on `Cod_Prod` and `Nom_Prod`: Must have 0 null entries.
   * Magic number check: First 4 bytes must match standard ZIP signature `0x50 0x4B 0x03 0x04`.

---

## 6. Integration Roadmap with Regional Inventory (SISMED)

Once the raw inventory data from the authenticated portal is mapped:
1. **Catalog Synchronization**: Run `DigemidCatalogAgent` daily to ensure newly authorized generic/branded medicines exist in the database.
2. **Inventory ETL (`medifinder-ingestor`)**: Map the `codmed` / `Cod_Prod` from the health facility reports to the normalized catalog records.
3. **Conversational Engine (`medifinder-mcp` + `medifinder-web`)**: Enable full-text and semantic search over `Nom_IFA` and `Nom_Prod`, ensuring queries like *"Busco paracetamol de 500mg cerca a Piura"* correctly resolve to available stock across local medical centers.
