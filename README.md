# Proyecto Ames de Ingeniería de Datos I

Este repositorio es el template del proyecto de la Unidad 4 de **Ingeniería de Datos I**. Creamos un repositorio a partir de este template y lo utilizamos para organizar el pipeline, la salida, la validación y la documentación del proyecto.

El proyecto consiste en construir en **Azure Data Factory** un proceso ETL que integra información de un archivo CSV, una base PostgreSQL en Neon y colecciones de MongoDB Atlas. El resultado fue el archivo `salida.csv`, con una fila por propiedad y los 81 campos establecidos en el enunciado.

## Información del equipo

- **Nombre del equipo:** `Equipo9`
- **Estado del proyecto:** `Finalizado`

| Integrante | Usuario de GitHub |
|---|---|
| `Jenifer Marcela Vivas` | `@MarcelaVivas13` |
| `Andres Olave` | `@Andrestast` |
| `Santiago Portilla` | `@Sportilla16` |
| `Sebastian Gonzalez` | `@sebasgonzalezm` |

## Objetivo

Construir un pipeline ETL en Azure Data Factory que investigue e integre por `PID` las fuentes entregadas, aplique las transformaciones definidas en el enunciado y produzca una salida validable con una fila por propiedad.

## Fuentes y tecnologías

| Componente | Uso en el proyecto |
|---|---|
| `sources/AmesProperty.csv` | Fuente de datos en formato CSV |
| PostgreSQL en Neon | Tablas relacionales del proyecto |
| MongoDB Atlas | Colecciones `garage`, `pool`, `bsmt` y `misc` |
| Azure Data Factory | Orquestación e implementación del ETL |
| Jupyter y Python | Validación de `salida.csv` |

## Salida

El archivo final fue guardado en `out/salida.csv` y cumple con las reglas del enunciado. Entre las comprobaciones principales se encuentran:

- una fila por propiedad;
- 81 campos en el orden solicitado;
- unicidad y cobertura de `PID`;
- uso consistente de camel case, snake case o kebab case, sin espacios;
- conservación de `PID`, `MS` y `QC` en mayúsculas;
- uso de `NA` y `0` como únicos marcadores permitidos para valores ausentes, según corresponda;
- aplicación y validación de las transformaciones obligatorias.

## Estructura del repositorio

| Ruta | Contenido esperado |
|---|---|
| `sources/` | Fuentes originales, colecciones y scripts entregados |
| `pipelines/` | Exportaciones y documentación del pipeline de ADF |
| `notebooks/` | Notebook de validación y guía de uso |
| `out/` | `salida.csv` y descripción de la salida final |
| `docs/` | Guías para el reporte, la presentación y el uso del repositorio |
| `report.md` | Reporte del proyecto |
| `README.md` | Presentación general y acceso a los entregables |

## Entregables

- [Reporte del proyecto](report.md)
- [Documentación del pipeline](pipelines/pipelines.md)
- Archivo final `out/salida.csv`
- [Descripción de la salida](out/out.md)
- [Notebook de validación](notebooks/validar_salida_ames.ipynb)
- Evidencias utilizadas en el reporte y la presentación

## Licencia

Consulte el archivo [LICENSE](LICENSE).
