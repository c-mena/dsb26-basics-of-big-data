# Retail Analytics Pipeline

Este repositorio aloja el proyecto del **Módulo 9: Fundamentos del Big Data**, perteneciente al **"Bootcamp Fundamentos de Ciencias de Datos 2026"** de **Talento Digital para Chile - SENCE**.

El proyecto aborda un caso de negocio para la empresa ficticia RetailMax, que maneja millones de transacciones diarias en su plataforma e-commerce. El objetivo es diseñar e implementar un pipeline integral de Big Data que cubra desde la ingesta y procesamiento de datos masivos con Apache Spark, hasta la construcción de modelos de Machine Learning escalables con MLlib, permitiendo clasificar usuarios y generar información accionable para el área de marketing.

Se utiliza el **[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)** como proxy de los datos transaccionales de RetailMax.

## Estructura del Proyecto

- [`README.md`](README.md): Presentación y documentación del proyecto.
- [`main.ipynb`](main.ipynb) ([▶ Ver en nbviewer](https://nbviewer.org/github/c-mena/dsb26-basics-of-big-data/blob/main/main.ipynb)): Notebook interactivo principal que contiene todo el desarrollo del caso.
- [`technical_report.md`](technical_report.md): Informe técnico con el resumen ejecutivo de los hallazgos, análisis, reflexiones y conclusiones.
- [`technical_report.pdf`](technical_report.pdf): Versión PDF del informe técnico.
- [`data/brazilian-ecommerce.zip`](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce): Dataset comprimido (No incluido en el repositorio. Se descarga automáticamente al ejecutar `main.ipynb`).
- `data/brazilian-ecommerce/`: Archivos CSV extraídos del dataset (No incluidos en el repositorio).
- [`data/enriched_orders.parquet`](data/enriched_orders.parquet): DataFrame enriquecido con datos consolidados de pedidos, generado en la Lección 4.
- [`data/customer_segments.parquet`](data/customer_segments.parquet): Resultados de la segmentación de clientes (K-Means), generado en la Lección 5.

## Requisitos y Tecnologías

- Para ejecutar este proyecto y reproducir los experimentos, es necesario disponer de un entorno Python (>=3.10, < 3.11) con las siguientes bibliotecas:
  - `pyspark`: Motor de procesamiento distribuido Apache Spark, utilizado para manipulación de RDDs, DataFrames, Spark SQL y MLlib.
  - `numpy`: Para operaciones numéricas auxiliares.
  - `pandas`: Para conversión de DataFrames de Spark a Pandas y manipulación tabular en visualizaciones.
  - `matplotlib`: Para la generación de gráficos estáticos.
  - `seaborn`: Para paletas de color y visualizaciones estadísticas avanzadas.
  - `plotly`: Para visualizaciones interactivas complementarias.
  - `fastparquet`: Backend de lectura/escritura de archivos Parquet.
  - `pyarrow`: Motor alternativo de serialización Parquet, compatible con Spark.
  - `scipy`: Para funciones estadísticas de soporte.
  - `statsmodels`: Para análisis estadístico complementario.
  - `ipykernel`: Para la ejecución del cuaderno en entorno Jupyter / VS Code.
- `Java JDK (11 o 17)`: Entorno de ejecución de Java necesario para el funcionamiento de Apache Spark. Se recomienda la versión 17.
- `Hadoop (Winutils)`: Requisito necesario para la correcta ejecución de Spark en entornos Windows, específicamente para la gestión de archivos Parquet.

## Instalación y Despliegue Local

### 1. Clonar el repositorio

```bash
git clone https://github.com/c-mena/dsb26-basics-of-big-data
cd dsb26-basics-of-big-data
```

### 2. Crear e inicializar el entorno virtual con `uv`

Para asegurar la máxima reproducibilidad y eficiencia en la gestión de entornos, se recomienda usar [uv](https://docs.astral.sh/uv/). Esto ayuda aislar y no contaminar tu versión global de Python de forma extremadamente veloz.

```bash
uv venv
```

### 3. Activar el entorno virtual

```bash
# Windows
.venv\Scripts\activate

# macOS / Linux
source .venv/bin/activate
```

### 4. Instalar dependencias

Recomendamos instalar y sincronizar las bibliotecas requeridas utilizando el archivo `pyproject.toml` incluido en el proyecto, garantizando operar con las dependencias exactas y probadas:

```bash
uv sync
```

#### 4.1 Instalación de Java JDK

Apache Spark requiere Java para funcionar. Se recomienda instalar el JDK 17 (Temurin):

1. Descarga el instalador desde [Adoptium - Temurin 17](https://adoptium.net/es/temurin/releases?version=17).
2. Sigue las instrucciones de instalación.
3. Asegúrate de configurar la variable de entorno `JAVA_HOME` apuntando al directorio raíz de tu instalación de Java.

#### 4.2 Configuración de Hadoop (Solo Windows)

Debido a que Spark utiliza ciertos componentes de Hadoop para operaciones de archivos locales (como la escritura de Parquet), en Windows es necesario contar con `winutils.exe` y `hadoop.dll`. El proyecto está configurado para buscar estos archivos en una carpeta local llamada `hadoop/bin`.

Para configurar esto manualmente, ejecuta los siguientes comandos en tu terminal (PowerShell):

```powershell
# Crear la estructura de carpetas
mkdir -Force hadoop/bin

# Descargar winutils.exe y hadoop.dll
curl.exe -L -o hadoop/bin/winutils.exe https://github.com/cdarlint/winutils/raw/master/hadoop-3.3.6/bin/winutils.exe
curl.exe -L -o hadoop/bin/hadoop.dll https://github.com/cdarlint/winutils/raw/master/hadoop-3.3.6/bin/hadoop.dll
```

### 5. Ejecutar el proyecto

La alternativa recomendada para inicializar y observar el proyecto es emplear **Visual Studio Code (VS Code)** con su extensión de **Jupyter** instalada. Sólo debes abrir la carpeta del repositorio en VS Code, seleccionar tu entorno virtual `.venv` como tu Kernel de ejecución, y abrir el archivo `main.ipynb` para ejecutar el código y visualizar sus correspondientes anotaciones en simultáneo.

_(Otra alternativa válida, desde la consola y con tu entorno activo, es ejecutar el comando `jupyter notebook` para levantar el servidor y entrar a `main.ipynb` directamente mediante tu navegador web)._

---