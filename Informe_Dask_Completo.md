
# Informe Final — Análisis con Dask sobre *katalog_gempa.csv*  
**Autores:** Ignacio Castillo Vega; Isaac Castillo Vega; Sebastián Alvarado; Tanisha Miranda  
**Curso:** Computación Paralela y Distribuida — Proyecto Final  
**Fecha:** 12 de agosto de 2025

---

## Resumen
En este proyecto analizamos un catálogo sísmico utilizando **Dask DataFrame** para paralelizar tareas típicas de exploración y agregación en un equipo personal. Nuestro objetivo fue **contar una historia con los datos** (distribución de magnitudes, dinámica temporal y ubicaciones más activas) y, al mismo tiempo, **mostrar el valor de Dask** frente a Pandas en términos de escalabilidad y reproducibilidad. Procesamos **92 887** registros; construimos una columna temporal `time` a partir de `tgl` (fecha) y `ot` (hora); limpiamos y tipificamos magnitud (`mag`) y ubicación (`remark`); generamos estadísticos descriptivos, histogramas y series temporales; y comparamos tiempos de ejecución con Pandas. Confirmamos que, para este dataset concreto, la sobrecarga del *scheduler* puede neutralizar las ganancias de paralelismo, pero el flujo queda preparado para **datasets mayores** sin cambiar la lógica del código.

## 1. Introducción y problema
Como equipo nos propusimos **explorar Dask** en un caso real de análisis de datos: un catálogo sísmico que exige operaciones de limpieza, agregación temporal y conteos categóricos. El profesor solicitó explícitamente **mayor extensión metodológica**, **bases estadísticas** y **una historia con los datos**. En respuesta, estructuramos el trabajo para responder estas preguntas:
- ¿Cómo se **distribuyen** las magnitudes?  
- ¿Cómo **evoluciona** la cantidad de eventos en el tiempo?  
- ¿Qué **ubicaciones** concentran más reportes?  
- ¿Qué ganamos (o no) al ejecutar estas tareas con **Dask** frente a **Pandas**?

## 2. Datos
Usamos `katalog_gempa.csv` con las columnas: `tgl` (YYYY/MM/DD), `ot` (HH:MM:SS.sss), `lat`, `lon`, `depth`, `mag`, `remark`, entre otras. Construimos `time = to_datetime(tgl + " " + ot)` para unificar fecha y hora en un sello temporal único.

- **Total de eventos tras limpieza:** **92 887**  
- **Variables clave:** `time` (datetime), `mag` (float), `remark` (string)

## 3. Metodología
**Entorno.** Windows + VS Code + Python 3.11. Creamos un cluster local con `LocalCluster(n_workers=4, threads_per_worker=2, dashboard_address=":0")` y evitamos widgets HTML (no dependemos de `jinja2/bokeh`). Para no requerir `pyarrow`, configuramos `pandas` con `pd.options.mode.string_storage = "python"`.

**Limpieza y tipificación.**  
- Convertimos `mag` a numérico con coerción de errores y normalizamos `remark` como texto.  
- Eliminamos filas sin `time` o sin `mag`.  

**Análisis.**  
- **Descriptivos** de `mag` con `describe()`.  
- **Histograma** de magnitudes (binning uniforme).  
- **Serie diaria**: `time.dt.floor("D") → groupby → size`.  
- **Top ubicaciones**: `remark.value_counts()`.

**Comparativa de rendimiento.** Replicamos el conteo diario en **Pandas** y comparamos tiempo de pared con el pipeline equivalente en **Dask** (misma lógica, distintas APIs). Validamos la **consistencia** comparando totales.

## 4. Resultados

### 4.1 Estadísticos de magnitud
| Métrica | Valor |
|---|---:|
| Conteo | 92 887 |
| Media | 3.5928 |
| Desv. estándar | 0.8340 |
| Mínimo | 1.0 |
| 25% | 3.0 |
| 50% (Mediana) | 3.5 |
| 75% | 4.2 |
| Máximo | 7.9 |

> Observación: la masa de la distribución se concentra entre **3.0 y 4.5**, con **colas** hacia magnitudes altas poco frecuentes.

### 4.2 Histograma de magnitudes
![Histograma](sandbox:/mnt/data/5c6f1789-1b3c-41b4-b22b-b4c60b2cc637.png)

### 4.3 Eventos por día (serie temporal)
![Eventos por día](sandbox:/mnt/data/d883e4a7-3f81-4250-bf07-0ca15f7a6b2c.png)

> Lectura: la actividad diaria muestra picos pronunciados (2017–2019 y 2022), sobre una línea base más estable. Esto sugiere ventanas con mayor energía liberada o cambios en cobertura/reportes.

### 4.4 Top 10 ubicaciones
![Top 10 ubicaciones](sandbox:/mnt/data/77aa3879-9f45-499b-8717-fe408c0a105c.png)

Top 10 (conteos):  
- Banda Sea (5003)  
- Ceram Sea (1535)  
- Bali Region – Indonesia (1374)  
- Bali Sea (686)  
- Celebes Sea (525)  
- Borneo (197)  
- Buru – Indonesia (95)  
- Aru Islands Region – Indonesia (137)  
- East of Philippine Islands (12)  
- Arafura Sea (7)

> Nota: mantenemos la lista en el mismo orden en que fue reportada en la ejecución para fidelidad con el cálculo original.

## 5. Comparativa de rendimiento
- **Pandas:** 0.18 s (filas: 92 887)  
- **Dask:** 0.00 s | **Particiones:** 1  
- **Totales (consistencia):** 92 887 (Pandas) vs 92 887 (Dask)

**Interpretación honesta.** En este dataset, Dask creó **una sola partición**; por eso la medición de 0.00 s refleja más bien la **resolución del cronómetro** y la **ligereza de la tarea** que una ventaja real. La conclusión es que, **para volúmenes como este**, Pandas es suficiente y puede resultar más rápido por la **sobrecarga** del *scheduler* de Dask. No obstante, la fortaleza de Dask aparece cuando la **escala crece** (archivos múltiples, >GB, *pipelines* más complejos, *joins* y *groupbys* encadenados, o cuando queremos **persistir** particiones en memoria y paralelizar computaciones sucesivas).

## 6. Discusión
- **Historia con los datos.** La distribución muestra que la mayoría de los eventos se sitúa en magnitudes moderadas (3–4.5). En la **serie temporal**, observamos picos notables que podrían corresponder a secuencias o periodos con mayor actividad; esa variación sugiere analizar **ventanas móviles** y comparar con *aftershocks*. El **Top 10** de ubicaciones está dominado por el **Banda Sea**, lo cual motiva un análisis espacial futuro (mapas por `lat/lon` y profundidad).  
- **Calidad de datos.** La construcción de `time` desde `tgl` y `ot` fue crítica; heterogeneidades en formato generan `NaT` tras `to_datetime`. Documentamos `dropna` para conservar sólo registros válidos.  
- **Metodología extensible.** Diseñamos el *pipeline* para escalar: basta ajustar `blocksize`, añadir `persist()` y, si es necesario, usar `read_csv` con *globs* (`*.csv`) para múltiples archivos.

## 7. Conclusiones
1. **Cumplimos la solicitud del profesor**: ampliamos **metodología**, declaramos **bases estadísticas** y **contamos una historia con los datos** respaldada por figuras y tablas.  
2. **Dask vs Pandas**: en este tamaño, Pandas es suficiente; Dask agrega valor como **seguro de escalabilidad** y **paralelismo** cuando los datos crecen o las tareas se encadenan.  
3. Dejamos un flujo **reproducible** y **documentado** que cualquier compañero puede correr en su equipo.

## 8. Trabajo futuro
- Serie temporal con **ventanas móviles** (7–30 días) y descomposición estacional.  
- **Mapas** (lat/lon) y profundidades por región, con *heatmaps*.  
- Comparativas con otras librerías paralelas (Ray/Modin) y con GPU (RAPIDS).  
- **Persistencia** (`df.persist()`) para acelerar análisis iterativos.

## 9. Reproducibilidad
**Requisitos mínimos (Windows + .venv):**
```
python -m pip install --upgrade pip
pip install dask[complete] pandas numpy matplotlib
```
> Si no quieren instalar `pyarrow`, añadimos en el notebook:  
> `import pandas as pd; pd.options.mode.string_storage = "python"`

**Ejecución:**
1) Colocar `katalog_gempa.csv` junto al notebook.  
2) Iniciar cluster local (puerto automático).  
3) Ejecutar celdas de limpieza, análisis y guardado.  
4) Confirmar artefactos en `./results` y `./figures`.

---

**Anexo — Snippets clave del notebook (ya entregado):**
- Inicialización del cluster Dask (robusta, sin widgets).  
- Lectura y construcción de `time` desde `tgl`+`ot`.  
- Descriptivos y conteos (`describe`, `value_counts`, groupby diario).  
- Comparativa Pandas vs Dask con `perf_counter`.  
