# ClientesConfiables · Roadmap del Producto de Datos

El roadmap del proyecto está estructurado estrictamente en **dos fases secuenciales** que, en conjunto, recorren todo el ciclo del dato: desde el diseño y prevención de calidad hasta la orquestación, consumo y monitoreo en producción.

```
Ciclo del Dato:
Diseño ──► Calidad Shift-Left ──► Despliegue ──► Ingesta ──► Transformación ──► Consumo ──► Monitoreo
└─────────────────── Fase 1 ───────────────────┘ └───────────────────────── Fase 2 ─────────────────────────┘
```

---

## Fase 1 · Diseño y Calidad Shift-Left

* **Objetivo:** Definir el modelo lógico y físico de datos, desplegar las tablas e infraestructuras iniciales en Databricks y automatizar la validación de calidad shift-left del archivo de origen en el Pull Request.
* **Qué entrega:**
  1. **Modelo Lógico y Físico:** Especificación formal de entidades, relaciones, tablas (`customers_bronze`, `customers_silver`, `customers_rejected`, `dim_paises`, `customers_gold_by_country` y `pipeline_logs`), columnas, tipos de datos y definición de banderas derivadas (`es_mayor_de_edad`, `es_email_valido`, `es_contactable`).
  2. **Scripts DDL de Despliegue:** Scripts SQL versionados para crear de forma idempotente (`CREATE TABLE IF NOT EXISTS`) los esquemas y tablas en Databricks SQL Warehouse (schema `demo`).
  3. **Dimensión de Países:** Script SQL/Seed para crear y poblar `demo.dim_paises` con códigos ISO, nombres y regiones geográficas de referencia.
  4. **Validación Shift-Left en PR:** Script Python y workflow de GitHub Actions que intercepta cada Pull Request para validar que `data/source/customers.csv` cumpla con encabezados, número de columnas, tipos y reglas estructurales básicas antes de cualquier interacción con Databricks.
  5. **Criterios de Aceptación y Pruebas:** Suite de validación en la carpeta de spec correspondiente comprobable contra el archivo fuente.
* **De qué fase depende:** Ninguna (Fase inicial).

---

## Fase 2 · Pipeline de Punta a Punta

* **Objetivo:** Implementar y orquestar el flujo automatizado completo de ingesta a Bronze, limpieza con segregación de rechazados a Silver, agregación de métricas a Gold y registro de auditoría en cada stage, permitiendo el merge a `main` únicamente con todos los checks en verde.
* **Qué entrega:**
  1. **Ingesta a Bronze (Stage 3):** Script automatizado que ingesta `customers.csv` a `demo.customers_bronze` adjuntando metadatos de auditoría (`run_id`, timestamp de carga, nombre de archivo).
  2. **Motor de Calidad y Limpieza (Stage 4):**
     * Enrutamiento de filas válidas a `demo.customers_silver` con cálculo de banderas (`es_mayor_de_edad`, `es_email_valido`, `es_contactable`) y deduplicación por `customer_id` (conservando el primer registro válido procesado).
     * Enrutamiento de registros anómalos a `demo.customers_rejected` detallando el motivo exacto del rechazo (`INVALID_EMAIL_FORMAT`, `INVALID_AGE_VALUE`, etc.).
     * Control de umbral de calidad: fallo automático del stage y cancelación de propagación hacia Gold si la tasa de rechazos supera el límite definido en la spec.
  3. **Métricas de Negocio en Gold (Stage 5):** Transformación SQL que cruza Silver con `dim_paises` para generar `demo.customers_gold_by_country` con el KPI de contactabilidad por país y región, libre de PII.
  4. **Módulo de Monitoreo y Auditoría (Stage 6):** Registro automatizado de métricas de ejecución (registros leídos, cargados, rechazados, duración y estado) en `demo.pipeline_logs`.
  5. **Pipeline Integral en GitHub Actions:** Configuración del workflow completo conectando los stages 1 al 6, protegido por los 3 secrets del repositorio y condicionado a la aprobación de checks para el merge a `main`.
  6. **Documentación Final en README:** Guía operativa de scripts por stage y validación de ejecución exitosa de punta a punta.
* **De qué fase depende:** Depende de la **Fase 1** (requiere el modelo de datos desplegado, las tablas creadas en Databricks y el contrato shift-left aprobado).
