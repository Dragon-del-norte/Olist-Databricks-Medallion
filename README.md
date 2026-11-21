# Olist E-Commerce Data Pipeline

Pipeline de datos end-to-end para el dataset público de Olist (e-commerce brasileño, 100K pedidos entre 2016 y 2018), implementado en Databricks con arquitectura Medallion.

## Contexto

Este proyecto resuelve un caso típico de ingeniería de datos: ingestar múltiples fuentes CSV, limpiarlas, unificarlas y generar tablas agregadas listas para consumo analítico. El objetivo no es el análisis en sí, sino construir la infraestructura que lo hace posible de forma reproducible y automatizada.

## Arquitectura

El pipeline sigue el patrón Medallion, separando las capas en esquemas independientes dentro de Unity Catalog:

raw_files (Volume)
|
v
bronze (9 tablas Delta, datos crudos + auditoría)
|
v
silver (5 tablas limpias + 1 tabla integrada)
|
v
gold (4 tablas agregadas para negocio)
|
v
Dashboard + Genie Agent


Las dependencias entre capas se manejan mediante un Job de Databricks con tres tareas secuenciales. Si la capa Bronze falla, Silver y Gold no se ejecutan.

## Estructura del repositorio

```text
.
├── notebooks/
│   ├── 01_bronze_ingestion.ipynb
│   ├── 02_silver_transformation.ipynb
│   └── 03_gold_aggregations.ipynb
├── docs/
│   ├── dashboard.png
│   ├── job_run.png
│   └── genie_agent.png
└── README.md


Decisiones técnicas

¿Por qué separar Bronze de raw files?
Los archivos CSV originales viven en un Volume de Unity Catalog sin transformación alguna. La capa Bronze los convierte en tablas Delta con dos columnas de auditoría (_ingested_at, _source_file) que permiten trazabilidad y reprocesamiento selectivo. Esta separación facilita auditorías y debugging.

¿Por qué usar CREATE OR REPLACE TABLE en lugar de INSERT o MERGE?
El dataset es estático y el pipeline se ejecuta completo en cada corrida. Para un caso de producción con datos incrementales, se usaría Auto Loader con MERGE INTO para procesamiento incremental. Este proyecto prioriza simplicidad y reproducibilidad.

¿Por qué tablas Gold separadas en lugar de vistas?
Las tablas Gold materializan agregaciones que se consultan con frecuencia desde dashboards. En un escenario con muchas consultas concurrentes, materializar reduce el costo de cómputo frente a recalcular en cada lectura.

¿Por qué no hay tests de calidad de datos?
El dataset es público y no tiene problemas de integridad conocidos. En un pipeline de producción, se agregarían validaciones con dbt tests o Databricks expectations. Queda fuera del alcance de este proyecto.

Stack

Databricks Free Edition (Serverless)
Unity Catalog
Delta Lake
SQL y PySpark
Databricks Workflows
AI/BI Dashboards
Genie Agents
Estructura de datos

Bronze

9 tablas Delta con los CSV crudos. Cada tabla incluye _ingested_at y _source_file.

Tablas: olist_customers, olist_geolocation, olist_order_items, olist_order_payments, olist_order_reviews, olist_orders, olist_products, olist_sellers, product_category_name_translation.

Silver

Transformaciones aplicadas: conversión de tipos (fechas a TIMESTAMP, precios a DECIMAL), normalización de texto, y joins para materializar orders_enriched con la información consolidada de pedidos.

Tablas: customers_clean, orders_clean, order_items_clean, products_clean, payments_clean, orders_enriched.


Gold

Tabla	Grano	Uso
ventas_por_mes	Mes	Análisis de tendencia temporal
top_categorias	Categoría de producto	Ranking de rentabilidad
ventas_por_estado	Estado brasileño	Distribución geográfica
analisis_entregas	Mes	KPIs de logística

Orquestación

Job olist_pipeline_medallion en Databricks Workflows:

3 tareas secuenciales: Bronze → Silver → Gold.
Schedule diario a las 06:00 (America/Lima).
Reintentos automáticos: 3 por tarea con intervalos.
Notificaciones por email solo en caso de fallo (a nivel de Job y tarea).
Duración típica: 2 minutos 14 segundos.


Consumo

Dashboard: 4 visualizaciones construidas sobre la capa Gold (ingresos por mes, top categorías, distribución por estado, tiempo de entrega).

Genie Agent: "Sales and Delivery Performance", conectado a las 4 tablas Gold, permite consultas en lenguaje natural en español. Ejemplo:

Usuario: ¿Cuáles fueron los ingresos totales por mes durante 2017?

Agente: [Genera el SQL, lo ejecuta en Databricks y devuelve un gráfico de barras con los ingresos mes a mes, acompañado de un resumen ejecutivo de los insights].

Cómo reproducir

Descargar el dataset desde Kaggle.
Crear el Volume workspace.default.raw_files en Unity Catalog.
Subir los 9 CSV al Volume.
Importar los notebooks de la carpeta notebooks/.
Ejecutar en orden: 01_bronze_ingestion, 02_silver_transformation, 03_gold_aggregations.
Opcional: configurar el Job en Workflows con las 3 tareas en secuencia.
Limitaciones

El pipeline está diseñado para ejecución batch completa. No soporta ingesta incremental.
No incluye tests de calidad de datos automatizados.
La versión gratuita de Databricks impone límites de cómputo diario que pueden afectar ejecuciones en horarios pico.
La capa Gold se recrea completa en cada corrida, sin procesamiento incremental.

Mejoras futuras

Migrar la ingesta Bronze a Auto Loader para procesamiento incremental.
Agregar tests de calidad con Databricks expectations.
Implementar SCD Type 2 en las dimensiones de clientes y productos.
Añadir una capa de métricas de negocio con KPIs versionados.
Autor

Percy Olarte — GitHub

Licencia

Dataset: CC BY-NC-SA 4.0 (Olist).
Código: uso educativo y de portafolio.
