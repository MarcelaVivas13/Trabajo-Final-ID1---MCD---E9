# Reporte del Proyecto Ames: Proceso ETL en Azure Data Factory

## 1. Identificación del proyecto

* **Curso y unidad:** Ingeniería de Datos I - 262, Universidad Icesi, Maestría en Ciencia de Datos — Evaluación Unidad 4
* **Proyecto:** Proceso de Extracción, Transformación y Carga (ETL) sobre los datos del Proyecto Ames
* **Integrantes:** Marcela Vivas, Sebastián Sánchez, Santiago Portilla y Andrés Olave (Equipo 9)
* **Fecha de entrega:** Sábado 28 de noviembre de 2026
* **Repositorio:** `https://github.com/MarcelaVivas13/Trabajo-Final-ID1---MCD---E9/tree/main`

---

## 2. Introducción y objetivo

### Introducción

El proyecto Ames plantea un reto de integración de datos sobre la información de 2.930 propiedades vendidas en Ames (Iowa) que está repartida en tres fuentes que no son relacionables de forma directa. Trabajamos con un archivo plano (`AmesProperty.csv`) que contiene los datos generales del lote y la ubicación, una base relacional en PostgreSQL (`amesDB`, alojada en Neon) conformada por 6 tablas entre entidades y catálogos (`amesdbtemp`, `saleproperty`, `floordetail`, `mssubclass`, `mszoning` y `typequality`), y una base documental en MongoDB Atlas (`test`) con 4 colecciones (`garage`, `pool`, `bsmt` y `misc`) que registran atributos opcionales de las viviendas. Para unificar estas fuentes sin perder inmuebles ni duplicar filas, diseñamos, implementamos y validamos un flujo ETL en Azure Data Factory (ADF) que consolida todo en `salida.csv`.

### Objetivo

Construir, ejecutar y validar un proceso ETL en Azure Data Factory que extraiga, normalice e integre las tres fuentes del proyecto Ames (`AmesProperty.csv`, las seis tablas de PostgreSQL en Neon y las cuatro colecciones de MongoDB Atlas), resolviendo diferencias de granularidad, tipos de datos y valores ausentes para generar `salida.csv` con exactamente una fila por propiedad (`PID`) y los 81 campos requeridos en formato `snake_case`.

---

## 3. Fuentes de datos y exploración

Antes de abrir Azure Data Factory, auditamos cada fuente por separado mediante `pgAdmin`, `MongoDB Compass` y por medio de Microsoft Excel para el csv. Contrastar los volúmenes de referencia contra mediciones reales permitió detectar diferencias de esquema y calidad antes de diseñar los cruces.

### 3.1 Archivo `AmesProperty.csv`

| Aspecto | Descripción / Valor observado | Evidencia |
| :--- | :--- | :--- |
| **Ubicación utilizada por el pipeline** | Contenedor `ames-etl-data` en Azure Blob Storage, ruta `raw/AmesProperty.csv` | Dataset `Ds_CSV_ames` en ADF |
| **Separador, codificación y encabezado** | **Separador:** Punto y coma (`;`)<br>**Codificación:** ASCII / UTF-8 sin BOM<br>**Encabezado:** Primera fila con 19 nombres con espacios | Inspección del archivo y configuración del dataset |
| **Filas y columnas medidas** | **Referencia:** 2.930 filas<br>**Observado:** 2.930 filas × 19 columnas | Conteo en Python |
| **Tipo, unicidad y faltantes de `PID`** | **Unicidad:** 2.930 valores únicos (0 duplicados, 0 vacíos)<br>**Tipo y longitud:** Entero, rango `526301100`–`1007100110` (2.929 de 9 dígitos y uno de 10: `1007100110`) | Auditoría de llave `PID` |
| **Campos que aporta a la salida** | **19 columnas:** `PID`, `Lot Frontage`, `Lot Area`, `Street`, `Alley`, `Lot Shape`, `Land Contour`, `Utilities`, `Lot Config`, `Land Slope`, `Neighborhood`, `Condition 1`, `Condition 2`, `Bldg Type`, `House Style`, `Overall Qual`, `Overall Cond`, `Year Built`, `Year Remod/Add` | Esquema de `Ds_CSV_ames` |
| **Problemas de calidad relevantes** | • `Year Remod/Add`: 1.569 vacíos (53,5 %). Ningún valor presente iguala a `Year Built`, lo que confirma que el vacío significa "sin remodelación".<br>• `Lot Frontage`: 490 vacíos (16,7 %) distribuidos en las 5 categorías de `Lot Config`.<br>• `Alley`: texto literal `"NA"` en 2.732 filas (93,2 %); es una categoría válida ("sin callejón"), no un nulo.<br>• **Códigos mezclados:** `Lot Shape` y `Neighborhood` traen descripción (“Regular”, “North Ames”), no código; `Condition 1` mezcla 2.646 códigos con 284 descripciones; `Condition 2` mezcla 23 códigos con 2.907 descripciones.<br>• **Dominio:** `Greens` (8), `GrnHill` (2) y `Landmrk` (1) no están en la tabla de equivalencias del enunciado porque ya vienen como código.<br>• **Atípicos e inconsistencias:** `Lot Area` máx. `215.245` (`PID 916176125`), `Lot Frontage` máx. `313`, y `PID 907194160` con `Year Remod/Add = 2001` anterior a `Year Built = 2002` (conservados sin alterar). | Perfilación de datos en Microsoft Excel|

![Ubicación del archivo CSV en Blob Storage](anexos/3.1%20ubicación%20csv.jpeg)

### 3.2 PostgreSQL en Neon (`amesDB`)

La base relacional `amesDB` (esquema `public`) tiene 6 tablas: 3 entidades de datos (`amesdbtemp`, `saleproperty`, `floordetail`) y 3 catálogos (`mszoning`, `typequality`, `mssubclass`). Al revisar el catálogo de restricciones (`pg_constraint`), confirmamos llaves primarias en todas las tablas y llaves foráneas desde `floordetail` y `saleproperty` hacia `amesdbtemp`; las relaciones hacia los catálogos no están declaradas como FK en el DDL, pero presentan integridad completa.

| Objeto / Tabla | Propósito | Campos relevantes | Llave | Cardinalidad con `PID` | Filas (Ref. / Obs.) | Hallazgos de calidad y faltantes | Aporte a `salida.csv` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`amesdbtemp`** | Entidad central con atributos estructurales y constructivos de cada vivienda. | `pid`, `MS SubClass`, `MS Zoning`, `Exter Qual`, `Exter Cond`, `Heating QC`, `Kitchen Qual`, `Central Air`, `1st Flr SF`, `2nd Flr SF`, `Low Qual Fin SF`, `Mas Vnr Type`, `Mas Vnr Area`, `electrical` (32 columnas). | PK: `pid` | **1:1** | **Ref:** 2.930<br>**Obs:** 2.930 | Sin duplicados. 23 nulos en `Mas Vnr Type` y `Mas Vnr Area`; 1 nulo en `electrical` (`PID 916386080`). `Central Air` es booleano (2.734 `True`, 196 `False`). | **32 columnas** (incluye `gr_liv_area` calculada). |
| **`saleproperty`** | Registra las condiciones comerciales y fecha de venta de cada propiedad. | `pid`, `Sale Date`, `Sale Type`, `Sale Condition`, `saleprice`. | PK: `id`<br>FK: `pid` | **1:1** | **Ref:** 2.930<br>**Obs:** 2.930 | Sin nulos ni duplicados. `Sale Date` es tipo `DATE`. `Sale Type` trae `"WD "` con espacio final en 2.536 filas. | **5 columnas** (`mo_sold`, `yr_sold`, `sale_type`, `sale_condition`, `sale_price`). |
| **`floordetail`** | Detalla habitaciones y baños desagregados por piso de la vivienda. | `pid`, `Floor`, `bedrooms`, `Full Bath`, `Half Bath`. | PK: `(pid, Floor)` | **1:N** | **Ref:** 4.182<br>**Obs:** 4.182 | Sin nulos ni duplicados sobre `(pid, Floor)`. 1.678 casas tienen 1 piso y 1.252 tienen 2 pisos. | **3 columnas** agregadas (`bedroom`, `full_bath`, `half_bath`). |
| **`mszoning`** | Catálogo de zonificación urbanística (`id` → `code`). | `id`, `code`, `description`. | PK: `id` | **1:N** (catálogo) | **Ref:** 8<br>**Obs:** 8 | Sin nulos ni huérfanos. Códigos `A (agr)`, `C (all)` e `I (all)` conservan paréntesis. | Traduce `MS_zoning` mediante `LEFT JOIN`. |
| **`typequality`** | Catálogo de escalas de calidad y condición (`1 Ex` a `5 Po`). | `id`, `code`, `description`. | PK: `id` | **1:N** (catálogo) | **Ref:** 5<br>**Obs:** 5 | Sin nulos ni huérfanos. | Traduce `exter_qual`, `exter_cond`, `heating_QC` y `kitchen_qual`. |
| **`mssubclass`** | Catálogo descriptivo del tipo de vivienda. | `code`, `description`. | PK: `code` | **1:N** (catálogo) | **Ref:** 16<br>**Obs:** 16 | Sin nulos ni duplicados. | Referencia (la salida conserva el código numérico de `amesdbtemp`). |

#### Comportamiento de `floordetail`
`floordetail` es la única entidad relacional con cardinalidad **1:N** respecto a `PID` (4.182 filas distribuidas entre 1.678 viviendas de un piso y 1.252 de dos pisos). Unirla directamente sin consolidar multiplicaría las 1.252 propiedades de dos niveles. Por ello se agrupa previamente por `pid` sumando `bedrooms`, `Full Bath` y `Half Bath` antes de cruzarla con `amesdbtemp`.

#### Evidencias de PostgreSQL en Neon
* **Figuras. Esquema y conteos en `amesDB`:**
* ![Conteo de registros en amesdbtemp](anexos/3.2%20amesdbtemp%20count.png)
*Figura 3.2a. Distribución por piso en `floordetail` (4.182 filas).*
* ![Conteo de registros en floordetail](anexos/3.2%20floordetail%20count.png)
*Figura 3.2b. Conteo total en `floordetail` antes de agrupar por `pid`.*
* ![Conteo de registros en mssubclass](anexos/3.2%20mssubclass%20count.png)
*Figura 3.2c. Verificación del catálogo `mssubclass` (16 registros).*
* ![Conteo de registros en mszoning](anexos/3.2%20mszoning%20count.png)
*Figura 3.2d. Verificación del catálogo `mszoning` (8 registros).*
* ![Conteo de registros en saleproperty](anexos/3.2%20saleproperty%20count.png)
*Figura 3.2e. Verificación de conteo en `saleproperty` (2.930 filas).*
* ![Conteo de registros en typequality](anexos/3.2%20typequality%20count.png)
*Figura 3.2f. Verificación del catálogo `typequality` (5 registros).*
* **Figura. Comprobación de duplicados y relación 1:N en `floordetail`:**
* *Figura 3.2g. Verificación de conteo en `amesdbtemp` (2.930 filas).*
* ![Conteo de registros en floordetail (vista detalle)](anexos/3.2%20floordetail%20count.jpeg)

### 3.3 MongoDB Atlas (`test`)

La base documental `test` contiene cuatro colecciones (`garage`, `bsmt`, `pool` y `misc`). En todas ellas la llave `PID` es entero largo (`Int64` / `$numberLong`), sin documentos sin `PID`, sin duplicados dentro de cada colección (cardinalidad máxima **0 a 1** respecto a las 2.930 casas) y sin arreglos ni subdocumentos anidados.

| Colección | Campos y tipos observados | Docs (Ref. / Obs.) | `PID` duplicados / sin `PID` | Casas sin doc. | Hallazgos y campos opcionales | Aporte a `salida.csv` |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`garage`** | `PID` (`Int64`), `Garage Type`, `Garage Finish`, `Garage Qual`, `Garage Cond` (`String`), `Garage Yr Blt`, `Garage Cars`, `Garage Area` (`Int64`), `Order` (`Int64`). | **Ref:** 2.773<br>**Obs:** 2.773 | 0 / 0 | 157 | 3 variaciones estructurales: algunos documentos omiten campos (ej. `PID 910201180` solo trae `Garage Type`; 2 documentos omiten `Garage Yr Blt`). Campo extra `Order` excluido. Valor atípico `Garage Yr Blt = 2207` (`PID 916384070`) conservado. | **7 columnas** (`garage_type`, `garage_yr_blt`, `garage_finish`, `garage_cars`, `garage_area`, `garage_qual`, `garage_cond`). |
| **`bsmt`** | `PID` (`Int64`), `Bsmt Qual`, `Bsmt Cond`, `Bsmt Exposure`, `BsmtFin Type 1`, `BsmtFin Type 2` (`String`), `BsmtFin SF 1`, `BsmtFin SF 2`, `Bsmt Unf SF`, `Total Bsmt SF`, `Bsmt Full Bath`, `Bsmt Half Bath` (`Int64`). | **Ref:** 2.851<br>**Obs:** 2.851 | 0 / 0 | 79 | 4 variaciones estructurales: `PID 903230120` solo trae `PID`; `PID 528142130` omite `BsmtFin Type 2`; 3 documentos omiten `Bsmt Exposure`. | **11 columnas** de sótano. |
| **`pool`** | `PID` (`Int64`), `Pool Area` (`Int64`), `Pool QC` (`String`). | **Ref:** 13<br>**Obs:** 13 | 0 / 0 | 2.917 | Los 13 documentos tienen sus 3 campos completos. | **2 columnas** (`pool_area`, `pool_qc`). |
| **`misc`** | `PID` (`Int64`), `Misc Feature` (`String`), `Misc Val` (`Int64`). | **Ref:** 106<br>**Obs:** 106 | 0 / 0 | 2.824 | 3 documentos registran `Misc Val = 0` explícito de origen. | **2 columnas** (`misc_feature`, `misc_val`). |

#### Documento ausente, campo ausente, nulo y cero válido
* **Documento ausente:** La vivienda no cuenta con esa infraestructura (ej. 2.917 casas sin `pool` o 157 sin `garage`). Tras el `Left Outer Join`, sus campos quedan nulos y reciben `NA` o `0`.
* **Campo ausente:** La vivienda tiene documento, pero omite una llave opcional (ej. 3 casas con sótano sin `Bsmt Exposure`). Al extraer hacia CSV en `raw/`, esa celda queda vacía (`null`) y recibe `NA` o `0`.
* **Valor nulo (`null`):** No se observaron `null` explícitos almacenados en los documentos BSON de origen.
* **Cero válido (`0`):** En `bsmt` (áreas parciales/baños) y en 3 documentos de `misc` (`Misc Val = 0`) el `0` viene de origen y se conserva intacto; en cambio, `Garage Cars`, `Garage Area`, `Pool Area` y `Total Bsmt SF` nunca valen `0` en la fuente, por lo que en la salida indican ausencia.

#### Evidencias de MongoDB Atlas
* **Figura 3. Estructura de documentos y conteos en `test`:** `[Insertar captura de documentos ejemplo en MongoDB Compass y conteo/unicidad de PID, sin credenciales]`.

### 3.4 Síntesis de la exploración

| Fuente y objeto | Llave | Cardinalidad con `PID` | Tamaño obs. | Problema de calidad | Aporte |
| :--- | :--- | :--- | ---: | :--- | :--- |
| CSV `AmesProperty.csv` | `PID` | 1:1 | 2.930 | Vacíos en `Year Remod/Add` (1.569) y `Lot Frontage` (490); descripciones mezcladas | 19 columnas |
| PG `amesdbtemp` | `pid` | 1:1 | 2.930 | Calidades y zona como `id`; 23 nulos en `Mas Vnr` y 1 en `electrical` | 32 columnas |
| PG `saleproperty` | `pid` | 1:1 | 2.930 | Fecha completa; `"WD "` con espacio final | 5 columnas |
| PG `floordetail` | `(pid, Floor)` | 1:N | 4.182 | 1 o 2 filas por propiedad | 3 agregadas |
| PG catálogos | `id` / `code` | catálogo | 8, 5, 16 | Relaciones no declaradas en DDL (sin huérfanos) | Códigos |
| Mongo `garage`, `bsmt`, `pool`, `misc` | `PID` | 0..1 | 2.773, 2.851, 13, 106 | Documentos y campos ausentes; formato `$numberLong`; campo `Order` | 22 columnas |

#### Hallazgos que definieron el diseño del ETL
1. **Agregación previa de `floordetail`:** La cardinalidad 1:N obligó a agrupar por `pid` dentro de la consulta SQL antes de cruzar con `amesdbtemp`.
2. **Desacoplamiento de MongoDB Atlas:** Como Mapping Data Flow no lee colecciones de MongoDB directamente y los enteros vienen envueltos en `$numberLong`, se creó un pipeline previo (`DS_Mongo`) para extraer las 4 colecciones a CSV en `raw/` con mapeo explícito.
3. **Homologación de `PID` como `long`:** Al venir como texto en CSV, entero en PostgreSQL e `Int64` en MongoDB (incluyendo un `PID` de 10 dígitos), se fijó `PID as long` en los 6 orígenes del Data Flow.
4. **Joins por izquierda:** La cobertura parcial de MongoDB (desde 13 hasta 2.851 documentos) exigió `Left Outer Join` desde el flujo base para conservar las 2.930 propiedades.

---

## 4. Diseño de la integración

El universo de propiedades lo define `AmesProperty.csv` con sus **2.930 `PID` únicos**. 

* **Reparto de procesamiento:** En PostgreSQL, la consulta SQL del origen `Postgress` resuelve los cruces con `mszoning`, `typequality` (4 alias) y `saleproperty`, la subconsulta agregada de `floordetail`, el cálculo de `Gr Liv Area`, la extracción de mes/año de venta, la conversión de `Central Air` y el `COALESCE` de nulos relacionales, entregando 2.930 filas con 40 atributos. En ADF, `df_ames` ejecuta las 5 uniones por `csv@PID` (`Left Outer Join`), imputa `NA`/`0`, normaliza códigos y ordena las 81 columnas.
* **Control de no multiplicación ni pérdida:** Se verificó cardinalidad máxima 1:1 por `PID` en cada rama antes de unir y se validó mediante el notebook que la salida tenga 2.930 filas con 0 `PID` duplicados, faltantes o adicionales frente al CSV base.

| Paso en `df_ames` | Entrada izquierda | Entrada derecha | Preparación previa | Tipo de unión | Llave | Filas antes → después | Justificación |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`joinpostgress`** | `csv` (2.930) | Consulta SQL `Postgress` (2.930) | `PID` proyectado como `long`; agregación de `floordetail` y catálogos resueltos en SQL | Externa izquierda (`left`) | `csv@PID` | 2.930 → **2.930** | 1:1; conserva el universo exacto del CSV base |
| **`joinpool`** | Flujo anterior (2.930) | `DsPool` (13) | Copia previa con `DS_Mongo` a `raw/pool`; `_id` excluido; `$numberLong` extraído | Externa izquierda (`left`) | `csv@PID` | 2.930 → **2.930** | 0..1; conserva 2.917 casas sin piscina |
| **`joingarage`** | Flujo anterior (2.930) | `DsGarage` (2.773) | Igual; campo `Order` excluido | Externa izquierda (`left`) | `csv@PID` | 2.930 → **2.930** | 0..1; conserva 157 casas sin garaje |
| **`joinbmst`** | Flujo anterior (2.930) | `DsBsmt` (2.851) | Igual | Externa izquierda (`left`) | `csv@PID` | 2.930 → **2.930** | 0..1; conserva 79 casas sin sótano |
| **`joinmisc`** | Flujo anterior (2.930) | `DsMisc` (106) | Igual | Externa izquierda (`left`) | `csv@PID` | 2.930 → **2.930** | 0..1; conserva 2.824 casas sin extras |

---

## 5. Transformaciones

### 5.1 Área habitable (`gr_liv_area`)
* **Origen y expresión:** Calculada en la consulta SQL de `Postgress` como `am."1st Flr SF" + am."2nd Flr SF" + am."Low Qual Fin SF"`.
* **Faltantes y validación:** Las tres columnas son enteras sin nulos en `amesdbtemp`. El notebook recalculó la suma fila por fila sobre `salida.csv` contra las fuentes: 0 diferencias (mín. `334`, máx. `5.642` en `PID 908154235`).

### 5.2 Habitaciones y baños (`bedroom`, `full_bath`, `half_bath`)
* **Origen y agregación:** Subconsulta SQL sobre `floordetail` ejecutada antes del `LEFT JOIN` con `amesdbtemp`: `SELECT pid, CAST(SUM(bedrooms) AS integer) AS bedroom, CAST(SUM("Full Bath") AS integer) AS fullbath, CAST(SUM("Half Bath") AS integer) AS halfbath FROM public.floordetail GROUP BY pid` (con `COALESCE(..., 0)` en la consulta externa).
* **Cardinalidad y validación:** Reduce 4.182 filas a 2.930. El notebook contrastó los tres totales por `PID` contra `floordetail.sql`: 0 diferencias.

### 5.3 Fecha de venta (`mo_sold`, `yr_sold`)
* **Origen y expresión:** Extraídos en la consulta SQL desde `saleproperty."Sale Date"` (`DATE`) mediante `CAST(EXTRACT(MONTH FROM sp."Sale Date") AS integer)` y `CAST(EXTRACT(YEAR FROM sp."Sale Date") AS integer)`.
* **Validación:** Comparación por `PID` en el notebook contra `saleproperty.sql`: 0 diferencias.

### 5.4 Fecha de remodelación (`year_remod_add`)
* **Regla y expresión:** En la actividad `cambios` (`Derived Column`) de `df_ames` se aplica `iifNull({Year Remod/Add}, {Year Built})`.
* **Validación:** El notebook reconstruyó el valor esperado por `PID` desde `AmesProperty.csv` (distinguiendo los 1.361 valores originales de los 1.569 sustituidos por `Year Built`): 0 diferencias y 0 registros vacíos o en `0`.

### 5.5 Variables de calidad y catálogos (`MS_zoning`, `exter_qual`, `exter_cond`, `heating_QC`, `kitchen_qual`)
* **Implementación:** En la consulta SQL de `Postgress`, `amesdbtemp` se cruza mediante `LEFT JOIN` con `mszoning` (`mz.id = am."MS Zoning"`) y cuatro alias de `typequality` (`eq`, `ec`, `hq`, `kq`) para obtener `.code`. Adicionalmente, `Central Air` se convierte con `CASE WHEN am."Central Air" THEN 'Y' ELSE 'N' END`.
* **Validación:** Cardinalidad 1:1 por propiedad, 0 códigos huérfanos y 0 diferencias por `PID` al contrastar contra `typequality.sql` en el notebook.

### 5.6 Datos ausentes de MongoDB (`NA` y `0`)
* **Regla y expresión:** Tras los cinco `Left Outer Join`, en `cambios` se aplican 22 expresiones `iifNull`: `iifNull(campo, 'NA')` para las 11 variables cualitativas y `iifNull(campo, toShort(0))` para las 11 numéricas.
* **Comportamiento:** Cubre tanto documentos completos ausentes (ej. 157 casas sin `garage`) como campos opcionales ausentes dentro de un documento existente (ej. 2 casas sin `Garage Yr Blt`, totalizando 159 ceros), sin alterar los ceros válidos de origen (como `Misc Val = 0` en 3 documentos de `misc`). Validado celda por celda por `PID` en el notebook: 0 diferencias.

### 5.7 Transformaciones opcionales (`neighborhood`, `condition_1`, `condition_2`, `lot_shape`)
* **Decisión:** Se aplicó la conversión completa a códigos en las cuatro columnas mediante expresiones `case(...)` en `cambios`:
  * `lot_shape`: `Regular` → `Reg`, `Slightly irregular` → `IR1`, `Moderately Irregular` → `IR2`, `Irregular` → `IR3`.
  * `neighborhood`: 21 descripciones traducidas a código (`North Ames` → `Names`, `College Creek` → `CollgCr`, etc.), conservando sin cambios los valores que ya venían en código (`Greens`, `GrnHill`, `Landmrk`, `Edwards`, `Gilbert`, `Sawyer`, `Veenker`).
  * `condition_1` y `condition_2`: 9 reglas `case` (coincidencia exacta y `startsWith`/`endsWith`) manteniendo los códigos existentes.
* **Validación:** El notebook verificó consistencia total de códigos en las cuatro columnas (0 mezclas).

### 5.8 Ajustes adicionales de limpieza y esquema
* `lot_frontage`: los 490 vacíos se imputaron con `'0'` mediante `iifNull({Lot Frontage}, '0')` en `cambios`.
* `mas_vnr_type` (23 nulos) → `'NA'`, `mas_vnr_area` (23 nulos) → `0` y `electrical` (1 nulo) → `'NA'` con `COALESCE` en SQL.
* `sale_type`: `trim({Sale Type})` en `cambios` para eliminar el espacio final de `"WD "`.
* `SelectorFinal`: descarta los `PID` redundantes de las uniones, ordena las 81 columnas y las renombra a `snake_case` conservando `PID`, `MS` y `QC` en mayúsculas.

### Matriz de transformaciones

| Campo de salida | Fuente y campo original | Regla | Implementación en ADF | Faltantes | Evidencia de validación |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `gr_liv_area` | PG `amesdbtemp`: `1st`, `2nd`, `Low Qual` | Suma aritmética | Consulta SQL en `Postgress` | Sin nulos | Por fila, notebook (0 dif.) |
| `bedroom`, `full_bath`, `half_bath` | PG `floordetail` por piso | Suma por `PID` | Subconsulta `GROUP BY` en SQL | `COALESCE(..., 0)` | Por `PID`, notebook (0 dif.) |
| `mo_sold`, `yr_sold` | PG `saleproperty.Sale Date` | Extraer mes y año | `EXTRACT` en SQL | Sin nulos | Por `PID`, notebook (0 dif.) |
| `year_remod_add` | CSV `Year Remod/Add` | Vacío → `Year Built` | `iifNull` en `cambios` | 1.569 vacíos sustituidos | Por `PID`, notebook (0 dif.) |
| `MS_zoning`, 4 calidades | PG `id` de catálogos | `id` → `code` | `LEFT JOIN` en SQL | 0 huérfanos | Por `PID`, notebook (0 dif.) |
| 22 campos de Mongo | Atlas `garage`, `bsmt`, `pool`, `misc` | Ausente → `NA` / `0` | `iifNull` en `cambios` | 157 / 79 / 2.917 / 2.824 casas | Por `PID`, notebook (0 dif.) |
| `neighborhood`, `condition_1/2`, `lot_shape` | CSV descripciones | Descripción → código | `case()` en `cambios` | Conserva códigos | Consistencia total, notebook |
| `lot_frontage` | CSV `Lot Frontage` | Vacío → `0` | `iifNull` en `cambios` | 490 vacíos | Marcadores, notebook |
| `central_air` | PG booleano | `True`/`False` → `Y`/`N` | `CASE` en SQL | Sin nulos | `2.734 Y / 196 N` |
| `sale_type` | PG `Sale Type` | Quitar espacio final | `trim()` en `cambios` | Sin nulos | Espacios alrededor, notebook |
| 81 encabezados | Todas las fuentes | `snake_case` sin espacios | Renombrado en `SelectorFinal` | — | Orden exacto 1..81, notebook |

---

## 6. Implementación en Azure Data Factory

* **Linked Services (sin credenciales):** `BlobStorage_Ames` (Azure Blob Storage, contenedor `ames-etl-data`), `AzurePostgreSql1` (PostgreSQL v2.0 con SSL obligatorio hacia `amesDB` en Neon) y `MongoNoSql` (MongoDB Atlas driver v2 hacia `test`).
* **Datasets:** `Ds_CSV_ames` (`raw/AmesProperty.csv`, `;`), `Ds_Postgress_ames` (consulta SQL), `DS_Mongo_pool|garage|bsmt|misc` (colecciones origen), `Ds_Pool|Garage|Bsmt|Misc` (archivos delimitados por coma en `raw/`) y `df_csv_salida` (carpeta `salida/`, separador `;`, encabezado).
* **Pipeline `DS_Mongo` (ingesta):** Ejecuta 4 actividades `Copy` en paralelo (`Copy Pool`, `Copy Garage`, `Copy bsmt`, `Copy Misc`) con `batchSize: 100`. Usa `TabularTranslator` para extraer los enteros desde `$['campo']['$numberLong']` y excluir `_id` y `Order`, dejando campos ausentes como vacíos en `raw/`.
* **Pipeline `ames_etl` (integración):** Ejecuta la actividad `ETL` (`ExecuteDataFlow` sobre `df_ames` con 8 núcleos `General` y `traceLevel: "Fine"`):
  1. **6 orígenes (`csv`, `Postgress`, `DsPool`, `DsGarage`, `DsBsmt`, `DsMisc`):** con `PID as long` y consulta SQL en `Postgress`.
  2. **5 uniones (`joinpostgress` → `joinmisc`):** `Left Outer Join` encadenados por `csv@PID`.
  3. **Transformación (`cambios`):** 29 expresiones (`iifNull`, `case()` y `trim()`).
  4. **Selección (`SelectorFinal`) y destino (`salida`):** ordena y renombra los 81 campos a `snake_case` y escribe un único archivo `salida.csv` (`partitionBy('hash', 1)`) en `salida/`.

#### Evidencias de implementación en ADF
* **Figura 4. Pipeline `DS_Mongo` y mapeo de `$numberLong`:** `[Insertar captura del canvas de DS_Mongo y TabularTranslator, sin credenciales]`.
* **Figura 5. Pipeline `ames_etl` y Data Flow `df_ames`:** `[Insertar captura del canvas completo de df_ames y detalle legible de expresiones en cambios]`.
* **Figura 6. Monitoreo de ejecución exitosa:** `[Insertar captura de Monitor mostrando estado Succeeded y conteos de filas leídas/escritas]`.

#### Problemas resueltos durante la implementación

| Problema | Causa | Decisión |
| :--- | :--- | :--- |
| Lectura directa de MongoDB no soportada en Data Flow y campos `Int64` anidados | El conector Atlas usa `Copy` y expone enteros como objetos `$numberLong` | Crear el pipeline `DS_Mongo` con mapeo explícito `$['campo']['$numberLong']` hacia `raw/` |
| Cruces sin coincidencia por diferencia de tipos en `PID` | `PID` entra como `String` en CSV, entero en SQL e `Int64` en Mongo (con un `PID` de 10 dígitos) | Proyectar `PID as long` en los 6 orígenes de `df_ames` |
| Errores de sintaxis en expresiones y columnas que empiezan por número | Nombres con espacios y campos como `1st_flr_sf`, `2nd_flr_sf` y `3ssn_porch` requieren delimitadores en el script `.dsl` | Encerrar esos identificadores entre llaves `{ }` en `cambios` y `SelectorFinal` |
| Advertencias por espacios finales en `Sale Type` y vacíos en `Lot Frontage` | `"WD "` conservaba espacio final y `Lot Frontage` tenía 490 celdas vacías | Incorporar `trim({Sale Type})` e `iifNull({Lot Frontage}, '0')` en `cambios` |

---

## 7. Validación de `salida.csv`

El archivo generado (`salida/salida.csv`, copiado en el repositorio como `out/salida.csv`) se auditó con el notebook oficial `notebooks/validar_salida_ames.ipynb` (`NAMING_STYLE = "snake_case"`, comparación contra fuentes locales en `sources/` activada). Las **28 pruebas** distribuidas en los **7 criterios de la rúbrica** finalizaron en estado **APROBADO** (0 errores y 0 advertencias):

| Criterio de la rúbrica | Pruebas ejecutadas | Resultado observado | Estado | Evidencia |
| :--- | :--- | :--- | :--- | :--- |
| **1. Dimensiones correctas** (4 pruebas) | Carga con `;`, dimensiones `2930 × 81`, `PID` válido y único, cobertura frente a `AmesProperty.csv` | `2.930 × 81`; 0 `PID` inválidos, duplicados, faltantes o adicionales | **APROBADO** | Resumen notebook (criterio 1) |
| **2. Ausencia de nulos** (3 pruebas) | Marcadores de faltantes, compatibilidad de tipos, espacios alrededor de valores | Solo `NA` en cualitativos y valores numéricos (incluyendo `0`) en numéricos; 0 espacios sobrantes | **APROBADO** | Resumen notebook (criterio 2) |
| **3. Nombres de columnas** (3 pruebas) | Encabezados exactos en `snake_case` y orden 1..81, sin duplicados ni espacios | Los 81 encabezados coinciden exactamente con el contrato | **APROBADO** | Resumen notebook (criterio 3) |
| **4. Transformaciones obligatorias** (5 pruebas) | `gr_liv_area` por fila, `year_remod_add` sin `0` y vs. CSV, agregación de `floordetail`, `mo_sold`/`yr_sold` vs. SQL | 0 filas con diferencia en las 5 pruebas contra origen | **APROBADO** | Resumen notebook (criterio 4) |
| **5. Transformaciones opcionales** (4 pruebas) | Consistencia de códigos en `neighborhood`, `condition_1`, `condition_2` y `lot_shape` | Las 4 columnas usan de forma consistente los códigos definidos | **APROBADO** | Resumen notebook (criterio 5) |
| **6. Variables de calidad** (1 prueba) | Traducción desde `typequality` (`exter_cond`, `exter_qual`, `heating_QC`, `kitchen_qual`) | 0 diferencias por campo contra `amesdbtemp` y `typequality` | **APROBADO** | Resumen notebook (criterio 6) |
| **7. Extracción de MongoDB** (8 pruebas) | Cardinalidad y comparación por `PID` de `garage` (2.773), `pool` (13), `bsmt` (2.851) y `misc` (106) | 0 duplicados y 0 diferencias en los 22 campos (`NA` en 157, 2.917, 80 y 2.824 casas respectivamente) | **APROBADO** | Resumen notebook (criterio 7) |

* **Figura 7. Resultados de autovalidación (`validar_salida_ames.ipynb`):** `[Insertar capturas limpias del cuadro Resumen (7 criterios) y Detalle de pruebas (28/28 APROBADO), omitiendo rutas locales de usuario]`.

---

## 8. Tecnologías utilizadas

| Categoría | Tecnología | Función concreta en el proyecto |
| :--- | :--- | :--- |
| **Orquestación e integración** | **Azure Data Factory (ADF)** | Ejecución del pipeline `DS_Mongo` (4 `Copy` en paralelo) y del pipeline `ames_etl` con el Data Flow `df_ames`. |
| **Almacenamiento** | **Azure Blob Storage** | Contenedor `ames-etl-data`: aloja `AmesProperty.csv` y las extracciones de MongoDB en `raw/`, y escribe `salida.csv` en `salida/`. |
| **Base de datos relacional** | **PostgreSQL 17 (Neon)** | Aloja `amesDB` (6 tablas); la consulta SQL de origen resuelve catálogos, agregación de pisos y cálculos. |
| **Base de datos documental** | **MongoDB Atlas** | Aloja la base `test` con las cuatro colecciones (`garage`, `bsmt`, `pool`, `misc`). |
| **Exploración y consulta** | **pgAdmin y MongoDB Compass** | Inspección de esquemas, conteos, llaves e integridad referencial antes de configurar ADF. |
| **Validación y análisis** | **Python  (`pandas`, `psycopg2`, `pymongo`)** | Auditoría inicial de calidad y ejecución de comparaciones fila a fila contra las fuentes locales. |
| **Entorno de pruebas** | **Jupyter Notebook / VS Code** | Ejecución del notebook de autovalidación `validar_salida_ames.ipynb` sobre `out/salida.csv`. |
| **Control de versiones** | **GitHub** | Gestión de artefactos exportados de ADF (`pipeline/`, `dataflow/`, `dataset/`, `linkedService/`), documentación y entrega. |

---

## 9. Resultados, limitaciones y reproducibilidad

* **Producto final:** `salida/salida.csv` (copiado en `out/salida.csv`), 2.930 filas × 81 columnas en `snake_case`, delimitador `;`, UTF-8, sin celdas vacías, con 28/28 pruebas aprobadas.
* **Artefactos en el repositorio:** definiciones JSON y `.dsl` de ADF (`pipeline/`, `dataflow/`, `dataset/`, `linkedService/` sin credenciales), plantilla ARM, notebook `validar_salida_ames.ipynb` ejecutado y `out/salida.csv`.
* **Parámetros que no se pueden cambiar para la ejecución:** contenedor `ames-etl-data`, carpetas `raw/` y `salida/`, consulta SQL incluida en `df_ames.dsl`, separador `;` y `NAMING_STYLE = "snake_case"`.
* **Limitaciones documentadas:** La regla global `NA`/`0` en MongoDB unifica viviendas sin la infraestructura con viviendas que omitieron un campo opcional (ej. `garage_yr_blt = 0` en 159 casas: 157 sin garaje y 2 sin año registrado). Asimismo, se conservaron sin alterar anomalías de origen (`garage_yr_blt = 2207` en `PID 916384070` y `year_remod_add = 2001 < year_built = 2002` en `PID 907194160`).
* **Verificaciones no ejecutadas:** Ninguna; todas las pruebas del notebook fueron evaluadas.



---

## 10. Conclusiones y lecciones aprendidas

1. **La cardinalidad define la estrategia de unión.** Conservar 2.930 propiedades dependió de agrupar `floordetail` por `pid` en SQL antes de cruzarlo con `amesdbtemp` y de usar `Left Outer Join` hacia las cuatro colecciones de MongoDB; de lo contrario, se habrían duplicado 1.252 viviendas o perdido registros sin piscina/garaje.
2. **Homologar la llave `PID` evita fallos.** Al recibir `PID` como texto en CSV, entero en PostgreSQL e `Int64` en MongoDB (incluyendo un valor de 10 dígitos), fijar `PID as long` en los seis orígenes aseguró cruces exactos.
3. **Delegar cruces relacionales al motor SQL simplifica el Data Flow.** Al Resolver catálogos, agregación de pisos y cálculos aritméticos en la consulta de origen redujo la complejidad de `df_ames` y facilitó verificar la rama relacional por separado.
4. **No confundir falta de información con cero observado.** Al Exportar MongoDB a CSV conservando los vacíos permitió que iifNull rellenara solo los documentos y campos ausentes, respetando los ceros válidos de origen (Misc Val = 0).
5. **Validar contra la rúbrica automatizada ayuda a detectar puntos claves del proceso.** Al Ejecutar `validar_salida_ames.ipynb` confirmó el cumplimiento estricto de los 81 encabezados en `snake_case`, la limpieza de espacios (`trim`) y la integridad celda por celda contra las tres fuentes.

---

## 11. Referencias y anexos

### Referencias (Formato IEEE)
[1] D. De Cock, "Ames, Iowa: Alternative to the Boston Housing Data as an End of Semester Regression Project," *Journal of Statistics Education*, vol. 19, no. 3, 2011.

[2] Microsoft, "Azure Data Factory documentation," *Microsoft Learn*. [En línea]. Disponible: https://learn.microsoft.com/azure/data-factory/. [Consultado: nov. 2026].

[3] Microsoft, "Mapping data flows in Azure Data Factory," *Microsoft Learn*. [En línea]. Disponible: https://learn.microsoft.com/azure/data-factory/concepts-data-flow-overview.

[4] Microsoft, "Copy data from or to MongoDB Atlas," *Microsoft Learn*. [En línea]. Disponible: https://learn.microsoft.com/azure/data-factory/connector-mongodb-atlas.

[5] Neon, "Neon PostgreSQL documentation." [En línea]. Disponible: https://neon.tech/docs.

[6] MongoDB, "MongoDB Atlas documentation." [En línea]. Disponible: https://www.mongodb.com/docs/atlas/.

[7] A. Villota, Material del curso *Ingeniería de Datos I*, Universidad Icesi, 2026-2.

### Anexos
* **Anexo A.** Consulta SQL completa ejecutada en el origen `Postgress` (`dataflow/df_ames.dsl`).
* **Anexo B.** Contrato de los 81 encabezados en formato `snake_case` (`notebooks/validar_salida_ames.ipynb`).
* **Anexo C.** Expresiones `case()` de correspondencia descripción → código para `neighborhood`, `condition_1`, `condition_2` y `lot_shape` (`dataflow/df_ames.dsl`).
* **Anexo D.** Resultados completos de las 28 pruebas de autovalidación (`notebooks/validar_salida_ames.ipynb`).

* **Anexo E.** Figuras de evidencia del proceso (`anexos`)
