#PIPELINE Datos de cáncer de cuello uterino en la ciudad de Cali y Bogotá

En este proyecto se aplicó un proceso de Extracción, Transformación y Carga (ETL) a bases de datos de salud sobre cáncer de cuello uterino en Bogotá y Cali, con el fin de analizar inequidades en los tiempos para la atención oportuna. En la extracción se recopilaron dos fuentes principales: Bogotá (con más de 35.000 registros) y una muestra más pequeña para Cali (con un total de 883 registros). En la transformación se estandarizaron fechas, categorías de régimen y estrato socioeconómico, además de crear la variable tiempo de espera entre examen y diagnóstico.
Finalmente, en la carga se consolidaron las bases depuradas para su análisis. Los resultados muestran que en Bogotá el tiempo promedio de espera fue de 12 días, mientras que en Cali ascendió a 76 días, con mayor variabilidad. También se hallaron diferencias en el número de díassegún el estrato socioeconómico, edad y régimen de afiliación del paciente. El proyecto demuestra cómo un pipeline ETL asegura la calidad de los datos y permite obtener evidencias para comprender desigualdades en la salud pública.

El proceso del pipeline tiene la siguiente estructura y tecnologias que se presentan en la imagen:
![Blank diagram (1)](https://github.com/user-attachments/assets/5e5530bc-2496-4652-867d-e1741a79f712)

El codigo esta separado por secciones la sección de datos de Cali y Bogotá es la parte en la cual se hace la extraccion de los datos desde el archivo .CSV donde se extrajeron los datos, en esta seccion pone todos los campos en minuscula y se quitan los espacios, así como también se se estandariza las opciones de tipo de regimen: subsidiado, contributivo, no_afiliado, excepcion_expecial.

FILE_CALI = "/content/drive/MyDrive/Estadistica/43.-cancer-de-cuello-uterino-d.abiertos-piii-2023-2022-2021.csv" #Archivo que va a leer en la ruta de drive
df_cl = pd.read_csv(FILE_CALI, sep=",",low_memory=False) #Se lee desde un archivo csv con delimitador ,
for col in df_cl.select_dtypes(include="object").columns:
    df_cl[col] = df_cl[col].str.lower()
    df_cl[col] = df_cl[col].str.strip()

df_cl["tip_ss_"] = df_cl["tip_ss_"].replace(mapa_tip_ss)
print(f"[Cali] Filas: {len(df_cl):,} | Columnas: {df_cl.shape[1]}")

#Función para mostrar los tipos de datos
display(pd.DataFrame({
    "Nombre Columna": df_cl.columns,
    "Tipo de Dato": [str(df_cl[c].dtype) for c in df_cl.columns],
    "Valores Unicos": [df_cl[c].nunique(dropna=True) for c in df_cl.columns],
    "Ejemplo de Valores": [df_cl[col].unique()[:5] for col in df_cl.columns],
}))



