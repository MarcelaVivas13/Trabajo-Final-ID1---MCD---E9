## Aquí va el archivo de salida con alguna descripción
## Descripción del Archivo de Salida (`salida.csv`)

El archivo `salida.csv` constituye el producto final consolidado del proceso ETL ejecutado en Azure Data Factory. Contiene el historial integral de transacciones inmobiliarias de la ciudad de Ames (Iowa), estructurado bajo un modelo tabular plano listo para tareas de análisis exploratorio y modelado predictivo.

### Especificaciones Técnicas Generales

| Característica | Detalle |
| :--- | :--- |
| **Nombre del archivo** | `salida.csv` |
| **Granularidad** | Una fila por propiedad identificada mediante su llave primaria única (`PID`) |
| **Volumen de registros** | **2,930 registros** de propiedades (2,931 líneas incluyendo encabezado) |
| **Dimensionalidad** | **81 atributos** organizados en el orden exacto definido por el esquema objetivo |
| **Convención de encabezados** | Formato `snake_case` (ej. `MS_sub_class`, `gr_liv_area`, `sale_price`) |
| **Codificación y delimitador** | UTF-8, valores separados por tabulador (`\t`) / coma (`,`) según configuración del *Sink* |
| **Variable objetivo** | `sale_price` (última columna, valor de venta en dólares estadounidenses) |

### Estructura y Grupos de Variables (81 Columnas)

El esquema resultante unifica los atributos provenientes de las tres fuentes heterogéneas (`Ames Property.csv`, `BD Ames` en SQL y colecciones en `MongoDB Atlas`):

1. **Identificación y Zonificación (Cols. 1–17):** `PID`, clasificación de construcción (`MS_sub_class`), zonificación (`MS_zoning`), dimensiones y geometría del lote (`lot_frontage`, `lot_area`, `lot_shape`, `land_contour`, `lot_config`, `land_slope`), accesos (`street`, `alley`), servicios (`utilities`), ubicación geográfica (`neighborhood`, `condition_1`, `condition_2`) y tipología arquitectónica (`bldg_type`, `house_style`).
2. **Calidad General y Antigüedad (Cols. 18–21):** Calificación general de materiales y estado (`overall_qual`, `overall_cond`), año de construcción (`year_built`) y año de remodelación (`year_remod_add`, imputado con `year_built` en ausencia de remodelación).
3. **Estructura Exterior y Cimentación (Cols. 22–30):** Cubierta (`roof_style`, `roof_matl`), revestimientos exteriores (`exterior_1st`, `exterior_2nd`), mampostería (`mas_vnr_type`, `mas_vnr_area`), calidad exterior (`exter_qual`, `exter_cond`) y cimentación (`foundation`).
4. **Sótano — Fuente MongoDB `bsmt` (Cols. 31–39, 48–49):** Altura, estado y exposición (`bsmt_qual`, `bsmt_cond`, `bsmt_exposure`), áreas terminadas y sin terminar (`bsmt_fin_type_1`, `bsmt_fin_sf_1`, `bsmt_fin_type_2`, `bsmt_fin_sf_2`, `bsmt_unf_sf`, `total_bsmt_sf`) y baños en sótano (`bsmt_full_bath`, `bsmt_half_bath`).
5. **Instalaciones y Distribución Interior (Cols. 40–47, 50–58):** Calefacción, aire acondicionado y sistema eléctrico (`heating`, `heating_QC`, `central_air`, `electrical`), áreas por planta (`1st_flr_sf`, `2nd_flr_sf`, `low_qual_fin_sf`), área habitable calculada sobre nivel del suelo (`gr_liv_area`), totales consolidados de habitaciones y baños agregados desde `FloorDetail` (`full_bath`, `half_bath`, `bedroom`), cocinas (`kitchen_abv_gr`, `kitchen_qual`), total de cuartos (`tot_rms_abv_grd`), funcionalidad (`functional`) y chimeneas (`fireplaces`, `fireplace_qu`).
6. **Garaje — Fuente MongoDB `garage` (Cols. 59–65):** Ubicación, año, acabado, capacidad de vehículos, área, calidad y condición (`garage_type`, `garage_yr_blt`, `garage_finish`, `garage_cars`, `garage_area`, `garage_qual`, `garage_cond`).
7. **Áreas Exteriores, Piscina y Extras — MongoDB `pool` y `misc` (Cols. 66–76):** Entrada pavimentada (`paved_drive`), terrazas y porches (`wood_deck_sf`, `open_porch_sf`, `enclosed_porch`, `3ssn_porch`, `screen_porch`), piscina (`pool_area`, `pool_QC`), cerca (`fence`) y características adicionales (`misc_feature`, `misc_val`).
8. **Información de Venta (Cols. 77–81):** Mes y año extraídos de la fecha de venta (`mo_sold`, `yr_sold`), tipo y condición de venta (`sale_type`, `sale_condition`) y precio final de venta (`sale_price`).

### Reglas de Calidad e Integridad Garantizadas
* **Ausencia de valores nulos (`NULL`):** Todos los registros inexistentes tras los cruces con las colecciones documentales de MongoDB fueron estandarizados asignando `"NA"` a variables cualitativas y `0` a variables numéricas.
* **Consistencia referencial:** Se conservan íntegros los 2,930 registros únicos sin duplicación derivada de la relación uno a muchos (`1:N`) de la tabla `FloorDetail`.
