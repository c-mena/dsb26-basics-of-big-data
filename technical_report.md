# Informe Técnico — Retail Analytics Pipeline

## Resumen

Este informe documenta el desarrollo de un pipeline de Big Data para la empresa ficticia RetailMax, implementado con Apache Spark en modo local. El proyecto procesó el [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), que contiene más de 99,000 pedidos y 112,000 ítems vendidos en un marketplace brasileño. El pipeline abarca desde la exploración con RDDs hasta la construcción de modelos de Machine Learning escalables con MLlib, generando métricas de negocio e insights accionables para el área de marketing.

---

## 1. Las 5V de Big Data aplicadas a RetailMax

El ecosistema Big Data se caracteriza por cinco dimensiones que determinan la complejidad del procesamiento de datos:

| Dimensión | Descripción | Aplicación en RetailMax |
|---|---|---|
| **Volumen** | Cantidad masiva de datos | Millones de transacciones diarias, historiales de navegación y reseñas acumuladas. |
| **Velocidad** | Rapidez de generación y procesamiento | Eventos de compra y navegación en tiempo real que requieren respuestas inmediatas. |
| **Variedad** | Diversidad de formatos y fuentes | Datos estructurados (transacciones, pagos), semi-estructurados (logs) y no estructurados (reseñas). |
| **Veracidad** | Confiabilidad y calidad de los datos | Posibles inconsistencias: duplicados, reseñas falsas, datos faltantes. |
| **Valor** | Capacidad de extraer información accionable | Segmentación de clientes, predicción de satisfacción, optimización de inventario. |

En nuestro caso práctico con el dataset Olist, trabajamos con datos estructurados (CSVs de pedidos, ítems, clientes, pagos y reseñas) que incluyen 8 tablas relacionales con más de 500,000 registros combinados.

---

## 2. Configuración de Spark

Se configuró Apache Spark en **modo local** (`local[*]`), aprovechando todos los núcleos disponibles para paralelismo. Las decisiones técnicas clave fueron:

- **SparkSession**: Motor unificado para RDDs, DataFrames y SQL.
- **Driver memory**: 4 GB para manejar el dataset completo en memoria.
- **Shuffle partitions**: 8, optimizado para ejecución local evitando el overhead de las 200 particiones por defecto.
- **Hadoop en Windows**: Se requirió la instalación de `winutils.exe` para habilitar la escritura de archivos Parquet en el sistema de archivos local.

La validación inicial confirmó la lectura correcta de los CSVs tanto como RDDs (textFile) como DataFrames (spark.read.csv), con esquemas inferidos automáticamente por Spark.

---

## 3. Análisis con RDDs — Linaje y DAG

### Transformaciones aplicadas

Se implementaron las transformaciones requeridas sobre los datos de transacciones:

- **`map`**: Extracción del mes de compra y creación de Pair RDDs `(order_id, price)`.
- **`filter`**: Selección de pedidos entregados (96,478 de 99,441 totales — 97%).
- **`flatMap`**: Descomposición de estados de pedido en tokens individuales.
- **`distinct`**: Identificación de 8 estados de pedido únicos.
- **`sortBy`**: Ordenamiento por monto total descendente.
- **`reduceByKey`**: Agregación de precios por pedido.

### Acciones y estadísticas

Las acciones ejecutadas sobre el RDD de precios arrojaron:

| Métrica | Valor |
|---|---|
| Total de ítems vendidos | 112,650 |
| Suma total de ventas | R$ 13,591,643.70 |
| Precio medio | R$ 120.65 |
| Desviación estándar | R$ 183.63 |
| Precio mínimo | R$ 0.85 |
| Precio máximo | R$ 6,735.00 |

### Linaje y DAG

El linaje del RDD `total_by_order_rdd` mostró la cadena: `HadoopRDD` → `MapPartitionsRDD` (textFile) → `PythonRDD` (filter + map) → `PairwiseRDD` → `ShuffledRDD` (reduceByKey). Spark agrupa las transformaciones *narrow* (map, filter) en un solo *stage* y genera un *shuffle* solo para operaciones *wide* como `reduceByKey`, optimizando la ejecución.

---

## 4. Análisis con Spark SQL — Métricas de negocio

### Ventas por categoría

Las 5 categorías con mayores ingresos fueron:

| Categoría | Ítems vendidos | Ingresos (R$) | Precio promedio (R$) |
|---|---|---|---|
| health_beauty | 9,670 | 1,258,681 | 130.16 |
| watches_gifts | 5,991 | 1,205,006 | 201.14 |
| bed_bath_table | 11,115 | 1,036,989 | 93.30 |
| sports_leisure | 8,641 | 988,049 | 114.34 |
| computers_accessories | 7,827 | 911,954 | 116.51 |

Es notable que **watches_gifts** genera el segundo mayor ingreso a pesar de no ser la categoría con más unidades vendidas, gracias a su alto precio promedio (R$ 201).

### Distribución geográfica

São Paulo (SP) domina ampliamente el volumen de pedidos, seguido por Río de Janeiro (RJ) y Minas Gerais (MG). Esta concentración en el sudeste brasileño es un patrón conocido en el comercio electrónico de Brasil.

### Evolución temporal

Se observa una tendencia de crecimiento sostenido desde los 750 pedidos mensuales en enero 2017 hasta un pico de más de 7,000 pedidos en noviembre 2017 (posiblemente influenciado por Black Friday), con volúmenes estables entre 6,000-7,000 pedidos mensuales durante 2018.

### Optimización con cache

Se utilizó `cache()` sobre el DataFrame de ítems (el más consultado), almacenándolo en memoria para evitar releer el CSV en consultas posteriores. El plan de ejecución confirmó el uso de `InMemoryTableScan` en lugar de `FileScan csv`.

### Persistencia en Parquet

Se generó un DataFrame enriquecido con 115,723 registros que consolida pedidos, clientes, productos, reseñas y pagos, guardado en formato Parquet para lectura eficiente en la etapa de ML.

---

## 5. Pipeline MLlib — Resultados

### 5.1 Regresión Logística (Clasificación de satisfacción)

Se construyó un pipeline MLlib para clasificar la satisfacción del cliente:
- **Label**: `review_score >= 4` → Satisfecho (1), caso contrario → Insatisfecho (0).
- **Features**: price, freight_value, payment_value, category_index, payment_index.
- **Pipeline**: StringIndexer (2 columnas categóricas) → VectorAssembler → LogisticRegression.

**Resultados:**

| Métrica | Valor |
|---|---|
| AUC-ROC | 0.5676 |
| Accuracy | 0.7700 |
| F1-Score | 0.6705 |

**Interpretación:** El modelo muestra un accuracy de 77%, pero este valor es engañoso debido al **desbalance de clases** (77% satisfechos vs 23% insatisfechos). El AUC-ROC de 0.57 indica que el poder discriminativo del modelo es bajo, cercano al azar (0.5). La matriz de confusión confirma que el modelo predice casi todos los casos como "satisfecho", clasificando correctamente solo 10 de 5,177 casos insatisfechos.

Esto sugiere que las variables transaccionales (precio, flete, tipo de pago) no son buenos predictores de la satisfacción del cliente por sí solas. Variables como tiempo de entrega, diferencia entre fecha estimada y real, y el contenido de las reseñas podrían mejorar significativamente el modelo.

### 5.2 K-Means (Segmentación de clientes)

Se evaluaron valores de K de 2 a 8, utilizando Silhouette Score como métrica de evaluación:

| K | Silhouette Score |
|---|---|
| 2 | **0.9582** |
| 3 | 0.8998 |
| 4 | 0.8638 |
| 5 | 0.7938 |
| 6 | 0.7047 |
| 7 | 0.7051 |
| 8 | 0.6829 |

Se seleccionó **K=2** por presentar el mayor Silhouette Score (0.96), indicando una separación muy clara entre los dos segmentos.

**Perfiles de los segmentos:**

| Cluster | Clientes | Gasto promedio (R$) | Ticket promedio (R$) | Review promedio | Flete promedio (R$) |
|---|---|---|---|---|---|
| **0 — Regulares** | 89,115 (97.4%) | 120.35 | 103.32 | 4.16 | 19.44 |
| **1 — Alto valor** | 2,385 (2.6%) | 1,195.34 | 962.62 | 4.07 | 48.95 |

**Interpretación:**
- **Cluster 0 (Clientes regulares)**: La gran mayoría de clientes con compras de ticket medio (~R$103) y buena satisfacción (4.16/5).
- **Cluster 1 (Clientes alto valor)**: Un grupo reducido pero económicamente significativo, con tickets ~9x superiores al promedio y costos de flete ~2.5x mayores, lo que sugiere productos más grandes o pesados.

---

## 6. Insights para marketing

### Basados en la segmentación K-Means

1. **Retención de clientes alto valor (Cluster 1)**: Representan solo el 2.6% de la base pero generan tickets 9 veces superiores. Se recomienda implementar programas de fidelización, acceso anticipado a ofertas y beneficios exclusivos.

2. **Activación de clientes regulares (Cluster 0)**: La estrategia debe enfocarse en incrementar el ticket promedio mediante cross-selling y bundles de productos. Su alta satisfacción (4.16/5) indica buena base para campañas de up-sell.

3. **Optimización logística**: Los clientes alto valor pagan fletes significativamente mayores (R$49 vs R$19). Ofrecer envío gratuito o con descuento a este segmento podría fortalecer su lealtad.

### Basados en la clasificación de satisfacción

4. **Limitaciones del modelo actual**: Las variables transaccionales son insuficientes para predecir satisfacción. Se recomienda incorporar variables de experiencia de entrega (tiempo real vs estimado) y análisis de texto de reseñas (NLP) en futuras iteraciones.

5. **Prioridad en categorías clave**: Las categorías `health_beauty` y `watches_gifts` generan los mayores ingresos y deberían recibir atención preferente en campañas de marketing y gestión de inventario.

---

## Conclusiones

1. **Apache Spark demostró ser una herramienta eficaz** para procesar y analizar el dataset de e-commerce, incluso en modo local. Las operaciones sobre RDDs permiten un control granular de las transformaciones, mientras que DataFrames y Spark SQL ofrecen una interfaz de alto nivel más productiva para consultas analíticas.

2. **El modelo de Regresión Logística evidenció las limitaciones** de utilizar exclusivamente variables transaccionales para predecir satisfacción del cliente. Con un AUC de 0.57, se confirma que la satisfacción depende de factores más complejos que el precio o la categoría del producto.

3. **La segmentación K-Means reveló una estructura clara** en la base de clientes, separando un pequeño grupo de alto valor (2.6%) del grueso de clientes regulares. Este hallazgo es directamente accionable para estrategias de marketing diferenciadas.

4. **El pipeline completo** (ingesta → RDDs → DataFrames → SQL → Parquet → MLlib) demuestra la integración progresiva de las herramientas de Spark en un flujo analítico coherente, cumpliendo con los objetivos del módulo de Fundamentos de Big Data.
