# ClientesConfiables · Stack Tecnológico y Arquitectura de Datos

## 1. Principios de Arquitectura

1. **Infraestructura y Modelado como Código:** Ninguna tabla, vista o esquema se crea de forma manual en la interfaz de Databricks. Toda la estructura está versionada en scripts DDL y se ejecuta a través del pipeline de CI/CD.
2. **Independencia de Herramientas (Vendor-Agnostic):** La arquitectura prioriza las **prácticas de ingeniería de datos y DataOps** sobre herramientas particulares. Databricks y GitHub Actions constituyen la implementación de referencia actual, pero cada componente posee alternativas directas sin alterar la especificación.
3. **Calidad Shift-Left:** La prevención de errores ocurre en el Pull Request antes de realizar cualquier llamada o cómputo en la plataforma de datos.
4. **Gobierno y Privacidad por Diseño:** El dato personal (PII como email) se aísla, protegiendo las capas analíticas y garantizando trazabilidad total por lote de ejecución.

---

## 2. Tecnologías del Stack y Alternativas Equivalentes

| Práctica / Capa | Herramienta en el Proyecto | Alternativa Open Source | Alternativa Comercial | Justificación de la Práctica |
|---|---|---|---|---|
| **Motor de Cómputo / Data Warehouse** | Databricks SQL Warehouse (schema `demo`) | DuckDB / PostgreSQL / Trino | Snowflake / Google BigQuery / AWS Redshift | Cómputo analítico SQL desacoplado del almacenamiento. |
| **Formato de Almacenamiento** | Delta Lake | Apache Iceberg / Apache Hudi | Snowflake Micro-partitions | Soporte ACID, viajes en el tiempo (time travel) y schemas estrictos. |
| **Orquestación y CI/CD** | GitHub Actions | Jenkins / GitLab CI / Apache Airflow | Dagster Cloud / Astronomer / Azure DevOps | Automatización por stages, ramas protegidas y ejecución shift-left. |
| **Lenguajes de Desarrollo** | SQL (DDL, DML, métricas) y Python (validación, conectores) | SQL estándar / Python (Pandas/Polars) | SQL / Python | SQL para lógica declarativa de datos; Python para validaciones y scripts de orquestación. |
| **Gestión de Secretos** | GitHub Secrets (`DATABRICKS_HOST`, `DATABRICKS_TOKEN`, `DATABRICKS_HTTP_PATH`) | HashiCorp Vault / SOPS | AWS Secrets Manager / Azure Key Vault | Eliminación de credenciales hardcodeadas en repositorios y logs. |
| **Validación de Calidad** | Scripts Python con reglas personalizadas | Great Expectations / Soda Core / dbt tests | Monte Carlo / Datadog Data Observability | Reglas sintácticas, de esquema y de negocio con bloqueo automático. |

---

## 3. Arquitectura de Datos: Patrón Medallion

```
[ data/source/customers.csv ]
              │
              ▼  (Stage 1: Shift-Left PR Validation)
      [ Validación Local ]
              │
              ▼  (Stage 3: Ingesta Bronze)
    [ demo.customers_bronze ] (Dato crudo + metadata de carga)
              │
              ▼  (Stage 4: Calidad y Deduplicación)
       ┌──────┴───────────────────────────┐
       ▼                                  ▼
[ demo.customers_silver ]      [ demo.customers_rejected ]
(Limpios, deduplicados,        (Filas inválidas + motivo de rechazo
 banderas de contactabilidad)   + run_id)
       │
       ├──────────────────────────────────┐
       │ (Stage 5: Enriquecimiento)       │ (Join)
       ▼                                  ▼
[ demo.customers_gold_by_country ] ◄── [ demo.dim_paises ]
(Agregaciones, KPI por país/región,     (Código, nombre, región)
 CERO datos personales/email)
```

### Capas del Modelo Físico y Tablas:
1. **Bronze (`demo.customers_bronze`):**
   * **Objetivo:** Ingesta fiel del archivo crudo sin transformaciones destructivas.
   * **Columnas:** `customer_id` (STRING), `name` (STRING), `age` (STRING), `country` (STRING), `email` (STRING), `ingestion_timestamp` (TIMESTAMP), `run_id` (STRING), `source_file` (STRING).
2. **Silver (`demo.customers_silver`):**
   * **Objetivo:** Datos validados, tipados, limpios y deduplicados (regla: primer registro procesado en caso de duplicidad de `customer_id`).
   * **Columnas:** `customer_id` (STRING), `name` (STRING), `age` (INT), `country_code` (STRING), `email` (STRING), `es_mayor_de_edad` (BOOLEAN), `es_email_valido` (BOOLEAN), `es_contactable` (BOOLEAN), `processed_at` (TIMESTAMP), `run_id` (STRING).
   * **Reglas:** Los menores de edad (`age < 18`) permanecen en Silver con `es_mayor_de_edad = false` y `es_contactable = false`. El email se conserva únicamente aquí para fines operativos de CRM bajo estricto control de acceso.
3. **Rechazados (`demo.customers_rejected`):**
   * **Objetivo:** Captura y aislamiento de filas que violan reglas de calidad críticas.
   * **Columnas:** `record_id` (STRING), `raw_record` (STRING/JSON), `rejection_reason` (STRING), `run_id` (STRING), `rejected_at` (TIMESTAMP).
   * **Motivos de rechazo:** `INVALID_EMAIL_FORMAT`, `INVALID_AGE_VALUE`, `NULL_CUSTOMER_ID`, `NULL_COUNTRY`.
4. **Referencia de Países (`demo.dim_paises`):**
   * **Objetivo:** Dimensión estática desplegada y versionada desde código en Fase 1.
   * **Columnas:** `country_code` (STRING, PK), `country_name` (STRING), `region` (STRING).
   * **Datos base:** PE (Perú, Sudamérica), CL (Chile, Sudamérica), CO (Colombia, Sudamérica), MX (México, Norteamérica), AR (Argentina, Sudamérica).
5. **Gold (`demo.customers_gold_by_country`):**
   * **Objetivo:** Agregación analítica por país y región para toma de decisiones (Andrés - BI).
   * **Columnas:** `country_code` (STRING), `country_name` (STRING), `region` (STRING), `total_customers` (INT), `contactable_customers` (INT), `contactability_rate` (DECIMAL(5,2)), `calculated_at` (TIMESTAMP), `run_id` (STRING).
   * **Restricción de gobierno:** Ningún dato personal (nombre ni email) está presente.
6. **Monitoreo y Auditoría (`demo.pipeline_logs`):**
   * **Objetivo:** Trazabilidad operativa de cada stage.
   * **Columnas:** `run_id` (STRING), `stage_name` (STRING), `status` (STRING: SUCCESS/FAILED), `records_read` (INT), `records_loaded` (INT), `records_rejected` (INT), `execution_time_seconds` (DOUBLE), `executed_at` (TIMESTAMP), `details` (STRING).

---

## 4. Pipeline de CI/CD: Stages y Reglas de Ejecución

El pipeline se ejecuta de forma secuencial y automatizada en **GitHub Actions**:

```
[PR / Push] 
     │
     ▼
[Stage 1: Calidad Shift-Left] ──(Falla)──► [Bloquea PR / Detiene ejecución]
     │ (Éxito)
     ▼
[Stage 2: Despliegue del Modelo] (Crea/actualiza esquemas y tablas idempotentemente)
     │ (Éxito)
     ▼
[Stage 3: Ingesta a Bronze] (Carga del lote crudo con trazabilidad)
     │ (Éxito)
     ▼
[Stage 4: Calidad de Datos] ──(Rechazos > Umbral)──► [Falla Stage / Cancela Gold]
     │ (Rechazos dentro de umbral)
     ▼
[Stage 5: Gold - Métricas de Negocio] (Calcula KPIs por país/región)
     │ (Éxito)
     ▼
[Stage 6: Monitoreo y Auditoría] (Registra bitácora final en demo.pipeline_logs)
```

### Detalle de cada Stage:
1. **Stage 1 · Calidad Shift-Left:**
   * Se ejecuta en cada Pull Request antes de interactuar con Databricks.
   * Valida en Python la existencia del archivo, columnas exactas requeridas, encabezados, tipos básicos e integridad sintáctica general.
   * Si falla, el PR queda bloqueado inmediatamente.
2. **Stage 2 · Despliegue del Modelo:**
   * Ejecuta scripts SQL DDL idempotentes (`CREATE TABLE IF NOT EXISTS` / migraciones) en el esquema `demo` de Databricks.
   * Asegura que el catálogo de países (`dim_paises`) y las tablas Medallion existan.
3. **Stage 3 · Ingesta a Bronze:**
   * Carga el archivo `customers.csv` a la tabla `demo.customers_bronze` adjuntando `run_id`, timestamp y nombre de archivo.
4. **Stage 4 · Calidad de Datos (Silver + Rechazados):**
   * Aplica validaciones de negocio: formato de correo (presencia de `@` y dominio válido), formato numérico de edad, campos no nulos.
   * Deduplica por `customer_id` conservando el primer registro procesado.
   * Registros válidos se insertan en `demo.customers_silver` calculando `es_mayor_de_edad`, `es_email_valido` y `es_contactable`.
   * Registros inválidos se insertan en `demo.customers_rejected` con su motivo.
   * **Política de Umbral de Fallo:** Si los registros rechazados superan el umbral establecido por la spec (o si ocurren fallos críticos en el lote), el stage **falla** y **se cancela la carga hacia Gold**, protegiendo los datos históricos de BI y alertando al equipo.
5. **Stage 5 · Gold (Métricas para el Negocio):**
   * Cruza `customers_silver` con `dim_paises` para generar métricas agregadas por país y región.
   * Calcula el KPI de contactabilidad y consolida la tabla lista para consumo de BI.
6. **Stage 6 · Monitoreo:**
   * Cada stage reporta sus métricas a `demo.pipeline_logs` para auditar salud del dato, volúmenes procesados y tiempos de respuesta.

---

## 5. Estrategia de Versionamiento y Ramas

* **Rama `main` Protegida:** No se permiten commits directos (`push` directo rechazado).
* **Flujo de Trabajo (Branching Model):**
  * Toda feature o fase se desarrolla en su rama dedicada (ej. `fase-1-diseno-shift-left`, `fase-2-pipeline-end-to-end`).
  * Los cambios se integran a `main` exclusivamente mediante **Pull Request**.
  * Regla de Merge: El merge solo se habilita cuando todos los checks y stages del pipeline en GitHub Actions pasen en **verde**.

---

## 6. Configuración de Secretos y Operabilidad

El repositorio está diseñado para ejecutarse de punta a punta configurando únicamente tres secretos en **GitHub Settings → Secrets and variables → Actions**:
1. `DATABRICKS_HOST`: Dirección del workspace (sin prefijo `https://`).
2. `DATABRICKS_HTTP_PATH`: Endpoint HTTP del SQL Warehouse (`/sql/1.0/warehouses/<id>`).
3. `DATABRICKS_TOKEN`: Token de autenticación personal (PAT).

Todos los scripts desarrollados para cada stage deben quedar documentados en el `README.md` indicando su función, dependencias y en qué stage se ejecutan.
