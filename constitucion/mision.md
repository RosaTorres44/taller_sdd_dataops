# ClientesConfiables · Misión del Producto de Datos

## 1. El PARA QUÉ del dato: Problema de negocio

En la actualidad, el equipo de **Marketing / CRM** envía campañas de correo electrónico semanales a la base de clientes exportada del CRM (`customers.csv`). Sin embargo, el equipo enfrenta una **tasa de rebote crítica y un rendimiento deficiente** debido a que:
1. Se desconoce qué registros cuentan con correos válidos y sintácticamente correctos.
2. No existe visibilidad inmediata de si un cliente es **mayor de edad**, lo que genera riesgos regulatorios y de negocio al intentar contactar a menores.
3. El equipo de **Analítica / BI** no dispone de un reporte consolidado por país y región actualizado antes de las 8:00 a. m. de cada lunes, lo que impide monitorear oportunamente la salud de la base.
4. El equipo de **Gobierno de Datos y Seguridad** carece de control sobre la exposición de Datos Personales (PII como emails en reportes) y de una trazabilidad clara sobre qué registros fallan y por qué motivo.

**ClientesConfiables** nace como un producto de datos confiable, gobernado y automatizado que transforma el archivo crudo del CRM en una base limpia, categorizada y agregada, garantizando que el negocio contacte únicamente a clientes elegibles sin manipulación manual.

---

## 2. Audiencia del Producto de Datos

* **Carmen (Marketing / CRM):** Consumidora directa de la lista de clientes contactables (mayores de edad con correo válido). Requiere saber el estado de contactabilidad sin realizar cálculos adicionales.
* **Andrés (Analítica / BI):** Consumidor de métricas agregadas por país y región geográfica. Requiere reportes actualizados y confiables cada lunes antes de las 8:00 a. m.
* **Patricia (Gobierno de Datos y Seguridad):** Supervisora del cumplimiento de políticas: confidencialidad del email (PII fuera de reportes analíticos), no contactar a menores de edad, trazabilidad por ejecución y aislamiento de registros rechazados para corrección en origen.
* **Raúl (Ingeniería de Datos / DataOps):** Responsable de la estabilidad, idempotencia, versionamiento en GitHub y ejecución automatizada por stages sin intervención manual.

---

## 3. Alcance del Proyecto

### Qué queda DENTRO:
* **Validación de calidad Shift-Left:** Verificación sintáctica, estructural y de tipos del archivo fuente en el Pull Request antes de ejecutar procesos en Databricks.
* **Arquitectura Medallion:**
  * **Bronze:** Ingesta del dato crudo con metadatos de auditoría (`run_id`, timestamp).
  * **Silver:** Datos limpios y deduplicados (regla de primer registro válido), con cálculo de banderas derivadas (`es_mayor_de_edad`, `es_email_valido`, `es_contactable`). Los menores de edad se conservan en Silver marcados con `es_mayor_de_edad = false`.
  * **Gold:** Agregación analítica por país (con nombre y región geográfica) y cálculo del indicador de contactabilidad. Libre de PII.
  * **Tabla de Rechazados:** Almacenamiento segregado de filas inválidas (email mal formado, nulos en campos obligatorios, edad no numérica, etc.) con el motivo específico del rechazo y metadatos de ejecución.
  * **Catálogo de Referencia de Países:** Dimensión con códigos ISO, nombres y regiones geográficas.
* **Trazabilidad y Monitoreo:** Registro histórico de cada ejecución (`run_id`, estado, volumen procesado, rechazados y tiempos de ejecución).
* **CI/CD Automatizado:** Pipeline en GitHub Actions gobernado por ramas protegidas y merge a `main` condicionado al éxito total de los stages.

### Qué queda FUERA:
* **Herramientas de entrega o envío de correos (ESP/CRM):** La integración o disparo de campañas en plataformas como HubSpot, Braze, Salesforce o Mailchimp.
* **Corrección manual o edición de datos en origen:** El pipeline no modifica el CRM de origen; aísla las anomalías en la tabla de rechazados para retroalimentar al equipo responsable.
* **Otras entidades o fuentes transaccionales:** No abarca ventas, pedidos, facturación ni navegación web.
* **Exposición de PII en capas analíticas:** El email queda estrictamente fuera de la capa Gold y de las herramientas de visualización masiva.

---

## 4. Cómo se ve el éxito (KPI medible)

### KPI Principal de Negocio:
$$\text{\% de Clientes Contactables} = \left( \frac{\text{Clientes únicos contactables (Silver)}}{\text{Total de clientes únicos válidos procesados}} \right) \times 100$$

* **Definición de "Cliente Contactable":**
  $$\text{es\_contactable} = (\text{es\_email\_valido} = \text{true}) \;\land\; (\text{es\_mayor\_de\_edad} = \text{true})$$
* **Línea Base (Actual):** **30%** de clientes contactables.
* **Meta de Éxito:** Alcanzar como mínimo el **60%** de contactabilidad efectiva en la base depurada.

### KPIs Operativos y de Calidad (DataOps):
1. **Puntualidad:** Pipeline ejecutado y capa Gold disponible cada lunes antes de las 8:00 a. m.
2. **Confiabilidad:** 0 registros con fallas sintácticas de email o tipos de datos erróneos en la capa Silver.
3. **Seguridad:** 0 credenciales expuestas en repositorios o logs; 0 correos expuestos en la capa Gold.
4. **Idempotencia:** 0 duplicados generados ante re-ejecuciones de un mismo lote.
