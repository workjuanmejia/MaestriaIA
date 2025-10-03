# 📊 PIPELINE: Datos de cáncer de cuello uterino en Cali y Bogotá

En este proyecto se aplicó un proceso de **Extracción, Transformación y Carga (ETL)** a bases de datos de salud sobre cáncer de cuello uterino en Bogotá y Cali, con el fin de analizar inequidades en los **tiempos de atención oportuna**.  

- **Bogotá**: más de **35.000 registros**.  
- **Cali**: muestra de **883 registros**.  

En la **transformación** se estandarizaron fechas, categorías de régimen y estrato socioeconómico, además de crear la variable **tiempo de espera entre examen y diagnóstico**.  
En la **carga**, se consolidaron las bases depuradas para su análisis.  

🔎 **Resultados principales**:
- En **Bogotá**, el tiempo promedio de espera fue **12 días**.  
- En **Cali**, ascendió a **76 días**, con mayor variabilidad.  
- Se observaron diferencias según **estrato socioeconómico, edad y régimen de afiliación**.  

Este proyecto demuestra cómo un pipeline **ETL asegura la calidad de los datos** y permite obtener evidencias sólidas para comprender desigualdades en la salud pública.

---

El pipeline se estructuró de la siguiente manera:
![Blank diagram (1)](https://github.com/user-attachments/assets/5e5530bc-2496-4652-867d-e1741a79f712)

1. **Extracción** → Datos abiertos en formato CSV.  
2. **Transformación** → Limpieza, normalización de variables y creación de indicadores.  
3. **Almacenamiento** → BigQuery (Google Cloud).  
4. **Visualización** → Power BI.  

## Código Carga de Datos

La sección de datos de Cali y Bogotá corresponde a la **extracción** desde los archivos CSV.  
En este paso:  
- Se convierten todos los campos a **minúsculas**.  
- Se eliminan espacios en blanco.  
- Se estandarizan los valores de **régimen**: `subsidiado`, `contributivo`, `no_afiliado`, `excepcion_especial`.
- Para poder replicar el resultado se debe especificar la ruta correcta de los archivos CSV 

```python
FILE_CALI = "/content/drive/MyDrive/Estadistica/43.-cancer-de-cuello-uterino-d.abiertos-piii-2023-2022-2021.csv"

# Lectura del archivo CSV
df_cl = pd.read_csv(FILE_CALI, sep=",", low_memory=False)

# Normalización de texto
for col in df_cl.select_dtypes(include="object").columns:
    df_cl[col] = df_cl[col].str.lower().str.strip()

# Estandarización de categorías
df_cl["tip_ss_"] = df_cl["tip_ss_"].replace(mapa_tip_ss)

print(f"[Cali] Filas: {len(df_cl):,} | Columnas: {df_cl.shape[1]}")
```
## Consolidacion de campos de los Dataframes

- En esta seccion se definió un formato para estandarizar el formato de la fecha en las dos fuentes de datos.
- Se selecionaron los siguientes parametros que se van a analizar para los datos de ambas Ciudades: `Ciudad`, `Año`, `Estrato`, `Regimen`, `Etnia`, `Fecha de examen`, `Fecha de diagnostico`, `Territorio`, `FEdad`, `Sexo`.
  
```python
#Bogotá
bog = pd.DataFrame()
bog["ciudad"]    = df_bg['ciudad']
bog["ano"]      = df_bg["ano"]
bog["estrato"]   = df_bg["estrato_"]
bog["regimen"]   = df_bg["tip_ss_"]
bog["etnia"]     = df_bg["nom_grupo_"]
bog["fecha_examen"] = df_bg["fec_toma_e"].apply(parse_fecha_espanol)
bog["fecha_diag"]   = df_bg["fec_res_ex"].apply(parse_fecha_espanol)
bog["territorio"]   = df_bg["LOCALIDAD_RESIDENCIA"]
bog["edad"]         = pd.to_numeric(df_bg["EDAD_"], errors="coerce")
bog["sexo"]         = df_bg["SEXO"]

#Cali
cll = pd.DataFrame()
cll["ciudad"]    = df_cl["ciudad"]
cll["ano"]      = parse_date(df_cl["fec_toma_e"]).dt.year
cll["estrato"]   = df_cl["codigo_sspm"]   # No está en este dataset
cll["regimen"]   = df_cl["tip_ss_"]
cll["etnia"]     = df_cl["per_etn_"]
# Convert to string before applying the parsing function
cll["fecha_examen"] = df_cl["fec_toma_e"].astype(str).apply(parse_fecha_es)
cll["fecha_diag"]   = df_cl["fec_con_"].astype(str).apply(parse_fecha_es)
cll["territorio"]   = df_cl["ciudad"]   # solo ciudad, no comuna
cll["edad"]         = pd.to_numeric(df_cl["edad_"], errors="coerce")
cll["sexo"]         = df_cl["sexo_"]

```

## Agregar columna de tiempo de espera en días

- Se agregó una columna calculada a cada uno de los dataframes que calcula la cantidad de días que tomó el diagnostico de cada paciente (fecha diagnostico - Fecha de examen)
  
```python
#Tiempo de espera en días Cali
cll["tiempo_espera_dias"] = (cll["fecha_diag"] - cll["fecha_examen"]).dt.days

#Filtro datos de Cali por filas donde esten las fechas de diagnostico y el examen
cll_filtrado = cll[(cll["fecha_examen"].notna()) &
              (cll["fecha_diag"].notna()) &
              (cll["fecha_diag"] >= cll["fecha_examen"])].copy()

#Tiempo de espera en días Bogotá
bog["tiempo_espera_dias"] = (bog["fecha_diag"] - bog["fecha_examen"]).dt.days

#Filtro datos de Cali por filas donde esten las fechas de diagnostico y el examen
bog_filtrado = bog[(bog["fecha_examen"].notna()) &
              (bog["fecha_diag"].notna()) &
              (bog["fecha_diag"] >= bog["fecha_examen"])].copy()
```

## Carga de Datos a big query

Primero se debe establecer la informacion de conexion basica a la base de datos. (Nombre del proyecto, dataset, Id de la tabla donde se va a cargar la informacion)
```python
from google.cloud import bigquery
from google.colab import auth

auth.authenticate_user()
project_id = "analisis-ccu"
dataset_id = "Datos_CCU"      

client = bigquery.Client(project=project_id)

dataset_ref = bigquery.Dataset(f"{project_id}.{dataset_id}")
dataset_ref.location = "southamerica-east1"  #se escogio esta region porque es la mas cercana a colombia
client.create_dataset(dataset_ref, exists_ok=True)

table_id =  f"{project_id}.{dataset_id}.Fact_data_CCU"

# 2) Esquema datos
schema = [
    bigquery.SchemaField("ciudad", "STRING"),
    bigquery.SchemaField("ano", "INTEGER"),
    bigquery.SchemaField("estrato", "INTEGER"),
    bigquery.SchemaField("regimen", "STRING"),
    bigquery.SchemaField("etnia", "STRING"),
    bigquery.SchemaField("fecha_examen", "DATE"),
    bigquery.SchemaField("fecha_diag", "DATE"),
    bigquery.SchemaField("territorio", "STRING"),
    bigquery.SchemaField("edad", "INTEGER"),
    bigquery.SchemaField("sexo", "STRING"),
]

job_config = bigquery.LoadJobConfig(
    schema=schema,
    write_disposition="WRITE_TRUNCATE",  # usa WRITE_APPEND si quieres anexar
)

job = client.load_table_from_dataframe(data_unificada, table_id, job_config=job_config)
job.result()
print("df unificadso cargado en:", table_id_bog)

```

## Reporte Power BI

Despues de cargar los datos se diseña un reporte en Power BI para facilitar la presentacion de datos y el analisis de los mismos. Para ello se creo una llave para el usuario donde se creó el reporte y con ello se realizó la ocnfiguracion en DNS del conector ODCB, de esta manera se garantiza la conectividad con la base de datos, así como también ya no se requiere autenticacion cada que se realice la conexión. 
Tras realizar este proceso, traer los datos mediante ODBC se acomodan las tablas, se crean las relaciones de las tablas de hechos con la de calendario para la funcionalidad de filtro, se crean las metricas que se requieren para el reporte. A continuacion se presenta el diseño visual del reporte.

<img width="939" height="518" alt="ReportePBI" src="https://github.com/user-attachments/assets/cc8ad1be-3ccb-41dd-83b8-ae9c26625432" />

