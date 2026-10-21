# Práctica Guiada de Data Wrangling: Pipeline de Limpieza y Transformación del Dataset `retail_x_clientes`

## Introducción y Objetivos de la Práctica

El presente documento constituye una **guía práctica, paso a paso, para el procesamiento, limpieza y transformación (Data Wrangling)** del conjunto de datos del sector minorista e-commerce titulado **`retail_x_clientes`**.

En el entorno profesional de la ciencia de datos, más del 80% del tiempo de desarrollo se dedica a la preparación de datos crudos. Este proceso no debe ser arbitrario; requiere un **pipeline secuencial y lógico** donde cada decisión técnica (imputación, corrección de texto, tratamiento de atípicos, discretización o escalamiento) esté fundamentada en **pruebas diagnósticas previas** y respaldada por la **teoría estadística correspondiente**.

### Objetivos de Aprendizaje:
1. **Ejecutar un pipeline de ingeniería de datos ordenado** en Python utilizando la librería `pandas`.
2. **Realizar diagnósticos exploratorios y estadísticos** antes de ejecutar cualquier modificación en el dataset.
3. **Fundamentar teóricamente** cada acción técnica (mecanismos de datos faltantes, reglas de detección de atípicos, criterios de discretización y transformaciones de escala).
4. **Construir un dataset maestro limpio**, estructurado y optimizado para analítica descriptiva o modelos predictivos.

---

## Estructura del Pipeline de Procesamiento (Orden Lógico)

Para evitar la propagación de errores durante la manipulación de datos, se sigue un flujo de trabajo estructurado en 8 fases secuenciales:

```
[Fase 1: Ingesta e Inspección Diagnóstica Inicial]
                       │
                       ▼
[Fase 2: Alineación y Corrección de Desplazamientos]
                       │
                       ▼
[Fase 3: Diagnóstico e Imputación de Faltantes]
                       │
                       ▼
[Fase 4: Normalización de Texto y Codificación Categórica]
                       │
                       ▼
[Fase 5: Parseo y Extracción de Variables Temporales]
                       │
                       ▼
[Fase 6: Diagnóstico y Tratamiento de Outliers e Inconsistencias]
                       │
                       ▼
[Fase 7: Discretización y Segmentación (Binning)]
                       │
                       ▼
[Fase 8: Escalamiento, Estandarización y Exportación Maestro]
```

---

## Fase 1: Ingesta de Datos e Inspección Diagnóstica Inicial

### 1.1 Base Teórica
La **ingesta de datos** es el proceso de lectura de archivos almacenados de forma persistente (CSV, Excel, bases de datos SQL) para introducirlos en la memoria RAM dentro de una estructura tabular bidimensional denominada `DataFrame`.

Antes de realizar cualquier modificación, es obligatorio realizar una **inspección estructural diagnóstica** para evaluar:
- **Dimensiones:** Cantidad total de registros (filas) y atributos (columnas).
- **Tipos de datos asignados (`dtypes`):** Identificar si las variables numéricas, categóricas o temporales fueron interpretadas correctamente por el analizador (*parser*).
- **Codificación de caracteres:** Detectar posibles fallas de decodificación (como el fenómeno *mojibake*, producido al leer caracteres UTF-8 como ISO-8859-1 o Latin-1).

### 1.2 Implementación en Python: Recreación e Inspección

A continuación se carga la muestra representativa extraída del archivo `retail_x_clientes (muestra).pdf`, conservando todas sus anomalías reales.

```python
import pandas as pd
import numpy as np
import io

# Muestra extraída del dataset retail_x_clientes (muestra).pdf
raw_data = """customer_id,edad,ingresos,ciudad,fecha_ultima_compra,num_compras,deuda
1860,50,23962.22,Ciudad de MÃ©xico30/08/2023,4,5365.92
4772,49,,puebla,05/06/2023,5,4692.15
4092,17,4617.36,CDMX,29/11/2022,9,2101.78
1466,40,9998.37,Puebla,09/10/2021,5,-1798.42
5426,21,12263.72,Ciudad de MÃ©xico02/09/2023,3,5512.77
4444,32,5317.75,MÃ©xico D.F.,24/01/2022,7,2705.17
4171,29,-744.64,Puebla,18/07/2023,5,5981.60
3919,50,20257.68,CDMX,23/05/2023,4,8548.70
1130,41,8548.70,MÃ©xico D.F.,17/05/2021,6,2617.45
2685,25,17142.44,Guadalajara,15/10/2021,2,4293.69
1769,51,21050.67,puebla,11/09/2022,4,6893.34
3391,23,10101.31,Puebla,16/02/2022,6,6590.41
3433,44,24433.88,MÃ©xico D.F.,11/02/2022,3,4129.35
2184,30,27826.50,Ciudad de MÃ©xico23/08/2022,7,50000.00
5555,36,15857.36,CDMX,23/02/2024,6,4039.72
4385,46,10173.63,MÃ©xico D.F.,14/09/2022,5,464.26
5117,46,10916.89,CDMX,08/12/2023,3,8446.80
5843,56,14134.13,Puebla,10/03/2021,4,4149.30
3904,39,24299.61,puebla,27/11/2022,8,-1038.51
1474,44,15897.99,puebla,10/05/2022,5,3986.45
2082,35,19703.82,puebla,03/06/2023,3,8393.79
3558,42,3200.04,CDMX,19/10/2021,6,5388.21
3047,45,9682.94,Puebla,18/10/2023,5,4713.60
3747,39,24765.86,Guadalajara,03/01/2023,6,8512.59
1975,18,20214.51,puebla,20/02/2022,5,-1428.33
2806,44,1890.56,CDMX,16/06/2023,2,5071.93
1189,44,10302.53,puebla,22/02/2022,3,6572.02
3734,32,22257.51,Puebla,17/07/2022,4,6929.23
4005,25,10915.29,CDMX,28/12/2022,2,4398.57
5658,58,15457.74,MÃ©xico D.F.,30/03/2024,4,5975.03
2899,46,18473.46,CDMX,01/01/2021,5,8414.35
2267,37,19164.43,Puebla,27/12/2023,5,2584.09
2528,26,17467.74,CDMX,12/06/2021,7,7053.46
4202,30,6949.70,Ciudad de MÃ©xico11/11/2021,7,4556.07
4556,56,5762.08,Guadalajara,29/11/2021,7,17467.74
4890,42,15411.76,puebla,09/06/2022,10,2509.08
1646,33,4157.52,Puebla,25/02/2021,10,1108.99
3888,46,17837.90,Puebla,24/01/2022,6,1292.42
3435,30,4059.24,Puebla,16/08/2022,5,4643.30
"""

# Carga inicial del DataFrame
df = pd.read_csv(io.StringIO(raw_data))

# Diagnóstico estructural
print("--- DIMENSIONES ---")
print(f"Filas: {df.shape[0]}, Columnas: {df.shape[1]}\n")

print("--- INSPECCIÓN DE TIPOS DE DATOS ---")
df.info()

print("\n--- MUESTRA REGISTROS CRUDOS ---")
print(df.head(3))
```

---

## Fase 2: Alineación y Corrección de Desplazamientos de Columnas

### 2.1 Base Teórica
En la ingesta de archivos CSV sin entrecomillado estricto (*quoting*), cuando una variable de texto incluye comas o se concatenan múltiples campos durante la extracción (ej. `Ciudad de México30/08/2023`), se genera un **desplazamiento de datos hacia la izquierda o derecha**.

Si un analista no diagnostica este desplazamiento, las columnas posteriores recibirán tipos de datos incompatibles (ej. la fecha se ubica en `num_compras` o la deuda queda como `NaN`), corrompiendo las fases posteriores de limpieza.

### 2.2 Diagnóstico del Desplazamiento
Identificamos los registros donde la columna `fecha_ultima_compra` no contiene el patrón de fecha esperado (`DD/MM/YYYY`) sino números enteros correspondientes a `num_compras`.

```python
# Mascara de detección de registros desplazados
mask_desplazados = ~df['fecha_ultima_compra'].str.contains(r'\d{2}/\d{2}/\d{4}', na=False)

print(f"Registros con columnas desplazadas detectados: {mask_desplazados.sum()}")
print("Muestra de filas desplazadas:")
print(df[mask_desplazados][['ciudad', 'fecha_ultima_compra', 'num_compras', 'deuda']])
```

### 2.3 Implementación de Re-alineación en Python

Reestructuramos manualmente las celdas afectadas separando la fecha incrustada mediante expresiones regulares (*regex*):

```python
for idx in df[mask_desplazados].index:
    # 1. Extraer la fecha (ultimos 10 caracteres) y la ciudad limpia de la columna 'ciudad'
    cadena_compuesta = str(df.loc[idx, 'ciudad'])
    fecha_extraida = cadena_compuesta[-10:]
    ciudad_limpia = cadena_compuesta[:-10].strip()
    
    # 2. Re-alinear los campos hacia la derecha
    df.loc[idx, 'deuda'] = df.loc[idx, 'num_compras']
    df.loc[idx, 'num_compras'] = df.loc[idx, 'fecha_ultima_compra']
    df.loc[idx, 'fecha_ultima_compra'] = fecha_extraida
    df.loc[idx, 'ciudad'] = ciudad_limpia

# Cast correcto de tipos de datos tras la re-alineación
df['num_compras'] = df['num_compras'].astype(float)
df['deuda'] = df['deuda'].astype(float)

print("\n--- VERIFICACIÓN TRAS REALINEACIÓN ---")
print(df.loc[mask_desplazados, ['ciudad', 'fecha_ultima_compra', 'num_compras', 'deuda']])
```

---

## Fase 3: Diagnóstico e Imputación de Valores Faltantes

### 3.1 Base Teórica

Los **datos faltantes** ocurren cuando no se registra un valor en una o más observaciones (`NaN`).

#### Mecanismos de Pérdida de Datos:
1. **MCAR (*Missing Completely at Random*):** La causa de la ausencia no se relaciona con ninguna variable del estudio.
2. **MAR (*Missing at Random*):** La ausencia depende de otras variables observadas pero no del propio valor faltante.
3. **MNAR (*Missing Not at Random*):** La ausencia depende del valor que falta (introduce sesgos severos).

#### Criterios para Selección del Método de Imputación:
- **Porcentaje $< 5\%:$** Se puede imputar con métricas de tendencia central o eliminar registros sin alterar sustancialmente la distribución.
- **Elección entre Media y Mediana:**
  - **Media ($\mu$):** Solo aplica si la variable es **simétrica y gaussiana** (sin atípicos).
  - **Mediana ($	ilde{x}$):** Se utiliza si la variable presenta **sesgo (*skewness*) o atípicos**, por ser un estadístico robusto.

### 3.2 Pruebas Diagnósticas de Nulos y Asimetría

```python
# 1. Conteo y Porcentaje de Valores Faltantes
nulos_diag = pd.DataFrame({
    'Nulos_Absolutos': df.isna().sum(),
    'Porcentaje_%': (df.isna().mean() * 100).round(2)
})
print("--- DIAGNÓSTICO DE VALORES FALTANTES ---")
print(nulos_diag)

# 2. Evaluación del Coeficiente de Asimetría (Skewness) para 'ingresos'
skew_ingresos = df['ingresos'].skew()
print(f"\nCoeficiente de Asimetría (Skewness) de 'ingresos': {skew_ingresos:.4f}")
```

### 3.3 Implementación de la Imputación en Python

Dado que el porcentaje de nulos en `ingresos` es bajo ($2.56\%$) y la variable muestra asimetría, la teoría indica utilizar la **mediana**:

```python
# Imputación por la Mediana de 'ingresos'
mediana_ingresos = df['ingresos'].median()
df['ingresos'] = df['ingresos'].fillna(mediana_ingresos)

print(f"Valor imputado en 'ingresos' (Mediana): {mediana_ingresos:.2f}")
print("Verificación de nulos en 'ingresos' post-imputación:", df['ingresos'].isna().sum())
```

---

## Fase 4: Normalización de Texto y Codificación Categórica

### 4.1 Base Teórica

Los atributos cualitativos en sistemas distribuidos presentan inconsistencias de caja (mayúsculas/minúsculas) y corrupción de caracteres (*Mojibake*).

#### Codificación Categórica (*One-Hot Encoding*):
Dado que los algoritmos estadísticos operan sobre matrices numéricas, el **One-Hot Encoding** convierte una variable categórica con $k$ niveles únicos en $k$ (o $k-1$) vectores binarios ($0$ o $1$), evitando atribuir relaciones de orden artificiales entre categorías sin jerarquía (como nombres de ciudades).

### 4.2 Pruebas Diagnósticas Categóricas

```python
print("--- CATEGORÍAS ÚNICAS EN 'ciudad' ANTES DE LIMPIAR ---")
print(df['ciudad'].value_counts(dropna=False))
```

### 4.3 Implementación de Normalización y Codificación en Python

```python
# 1. Mapeo de Estandarización y Corrección de Mojibake
mapeo_ciudades = {
    'Ciudad de MÃ©xico': 'CDMX',
    'MÃ©xico D.F.': 'CDMX',
    'Ciudad de México': 'CDMX',
    'puebla': 'Puebla'
}

df['ciudad'] = df['ciudad'].replace(mapeo_ciudades)

print("\n--- CATEGORÍAS ESTANDARIZADAS ---")
print(df['ciudad'].value_counts())

# 2. Generación de Variables Indicadoras (Dummies)
df_dummies = pd.get_dummies(df['ciudad'], prefix='ciudad', dtype=int)
df = pd.concat([df, df_dummies], axis=1)

print("\nVariables binarias (dummies) integradas:")
print(df_dummies.head(3))
```

---

## Fase 5: Parseo y Extracción de Variables Temporales

### 5.1 Base Teórica
Las fechas almacenadas como cadenas de texto impedimentos numéricos y cronológicos. Convertirlas al tipo nativo `datetime64[ns]` permite calcular métricas cuantitativas como la **recencia** (días transcurridos desde la última transacción hasta la fecha de análisis).

### 5.2 Pruebas Diagnósticas Temporales

```python
print("Tipo de dato de 'fecha_ultima_compra':", df['fecha_ultima_compra'].dtype)
print("Muestra de fechas:", df['fecha_ultima_compra'].head(3).tolist())
```

### 5.3 Implementación del Parseo de Fechas en Python

```python
# 1. Parseo a datetime64
df['fecha_ultima_compra'] = pd.to_datetime(df['fecha_ultima_compra'], format='%d/%m/%Y')

# 2. Extracción de componentes temporales
df['año_compra'] = df['fecha_ultima_compra'].dt.year
df['mes_compra'] = df['fecha_ultima_compra'].dt.month

# 3. Cálculo de Recencia (Días transcurridos a la fecha de corte 01/04/2024)
fecha_corte = pd.to_datetime('2024-04-01')
df['recencia_dias'] = (fecha_corte - df['fecha_ultima_compra']).dt.days

print("\n--- NUEVAS VARIABLES TEMPORALES ---")
print(df[['fecha_ultima_compra', 'año_compra', 'mes_compra', 'recencia_dias']].head(3))
```

---

## Fase 6: Diagnóstico y Tratamiento de Outliers e Inconsistencias Numéricas

### 6.1 Base Teórica

#### Inconsistencias Lógicas:
Valores fuera del dominio válido (ej. `deuda` o `ingresos` con signo negativo). Deben ajustarse mediante reglas lógicas de negocio (ej. convertir deudas negativas a $0$).

#### Valores Atípicos (*Outliers*):
Observaciones extremas que se distancian del grueso muestral.

#### Métodos de Detección:
1. **Rango Intercuartílico ($IQR$) - Para distribuciones sesgadas:**
   $$IQR = Q_3 - Q_1$$
   - Límite Inferior: $LI = Q_1 - 1.5 \cdot IQR$
   - Límite Superior: $LS = Q_3 + 1.5 \cdot IQR$
2. **$Z$-Score - Para distribuciones gaussianas:**
   $$z = \frac{x - \mu}{\sigma}$$
   Atípico si $|z| > 3$.

#### Tratamiento por Acotamiento (*Capping* / Winsorización):
Para conservar el tamaño muestral, los valores que superan $LS$ se acotan exactamente a $LS$.

### 6.2 Pruebas Diagnósticas de Outliers

```python
# 1. Diagnóstico de Inconsistencias de Signo
print(f"Deudas negativas: {(df['deuda'] < 0).sum()}")
print(f"Ingresos negativos: {(df['ingresos'] < 0).sum()}")

# 2. Diagnóstico de Outliers mediante IQR en 'ingresos'
Q1_ing = df['ingresos'].quantile(0.25)
Q3_ing = df['ingresos'].quantile(0.75)
IQR_ing = Q3_ing - Q1_ing

LI_ing = Q1_ing - 1.5 * IQR_ing
LS_ing = Q3_ing + 1.5 * IQR_ing

outliers_count = ((df['ingresos'] < LI_ing) | (df['ingresos'] > LS_ing)).sum()
print(f"\n[Ingresos] Q1={Q1_ing:.2f}, Q3={Q3_ing:.2f}, IQR={IQR_ing:.2f}")
print(f"[Ingresos] Límite Superior (LS): {LS_ing:.2f}")
print(f"[Ingresos] Outliers detectados: {outliers_count}")
```

### 6.3 Implementación de Correcciones y Winsorización en Python

```python
# 1. Rectificación de Signos Negativos
df['deuda'] = np.where(df['deuda'] < 0, 0, df['deuda'])
df['ingresos'] = np.where(df['ingresos'] < 0, mediana_ingresos, df['ingresos'])

# 2. Acotamiento (Capping) de Outliers en 'ingresos' al LS
df['ingresos_capped'] = np.where(df['ingresos'] > LS_ing, LS_ing, df['ingresos'])

print("\n--- RESULTADOS POST-TRATAMIENTO ---")
print(f"Deuda Mínima corregida: {df['deuda'].min():.2f}")
print(f"Ingreso Máximo original vs Capped: {df['ingresos'].max():.2f} -> {df['ingresos_capped'].max():.2f}")
```

---

## Fase 7: Discretización y Segmentación de Variables Continuas (*Binning*)

### 7.1 Base Teórica

La **discretización** transforma una variable numérica continua en rangos o contenedores categóricos discretos (*bins*).

#### Criterios de Selección:
1. **Amplitud Constante (`pd.cut`):** Divide el rango $[Min, Max]$ en intervalos del mismo ancho.
2. **Frecuencia Constante / Cuantiles (`pd.qcut`):** Utiliza cuantiles muestrales para asegurar que cada contenedor albergue exactamente el mismo número de observaciones ($33.3\%$ en terciles). Recomendado para distribuciones sesgadas.

### 7.2 Implementación de Discretización en Python

```python
# 1. Discretización por Límites de Dominio (pd.cut) para 'edad'
bordes_edad = [0, 25, 45, 100]
etiquetas_edad = ['Joven', 'Adulto', 'Adulto_Mayor']
df['grupo_edad'] = pd.cut(df['edad'], bins=bordes_edad, labels=etiquetas_edad)

# 2. Discretización por Cuantiles (pd.qcut) para 'ingresos_capped'
df['nivel_ingreso_tercil'] = pd.qcut(df['ingresos_capped'], q=3, labels=['Bajo', 'Medio', 'Alto'])

print("--- CONTEO DE REGISTROS POR CONTENEDOR ---")
print("Grupos de Edad:\n", df['grupo_edad'].value_counts())
print("\nNivel de Ingresos (Terciles):\n", df['nivel_ingreso_tercil'].value_counts())
```

---

## Fase 8: Escalamiento, Estandarización y Exportación Maestro

### 8.1 Base Teórica

Los algoritmos basados en distancias (KNN, K-Means) o optimización por gradiente son sensibles a las diferencias de magnitud entre variables. 

#### Matriz de Decisión de Transformación:

| Técnica | Fórmula | Rango | Criterio de Uso |
| :--- | :--- | :--- | :--- |
| **Min-Max Scaling** | $$x' = \frac{x - Min}{Max - Min}$$ | $[0, 1]$ | Sin supuesto de normalidad; requiere rangos acotados rígidos. |
| **Estandarización $Z$-Score** | $$z = \frac{x - \mu}{\sigma}$$ | $\mu=0, \sigma=1$ | Variables gaussianas o algoritmos lineales/PCA. |

### 8.2 Implementación de Escalamiento y Exportación en Python

```python
from sklearn.preprocessing import MinMaxScaler, StandardScaler

# 1. Escalamiento Min-Max para 'ingresos_capped' y 'deuda'
scaler_minmax = MinMaxScaler()
df[['ingresos_minmax', 'deuda_minmax']] = scaler_minmax.fit_transform(df[['ingresos_capped', 'deuda']])

# 2. Estandarización Z-Score para 'recencia_dias' y 'num_compras'
scaler_zscore = StandardScaler()
df[['recencia_zscore', 'num_compras_zscore']] = scaler_zscore.fit_transform(df[['recencia_dias', 'num_compras']])

# 3. Consolidación de Atributos Maestros
columnas_maestras = [
    'customer_id', 'edad', 'grupo_edad',
    'ingresos_capped', 'ingresos_minmax', 'nivel_ingreso_tercil',
    'deuda', 'deuda_minmax',
    'num_compras', 'num_compras_zscore',
    'recencia_dias', 'recencia_zscore',
    'ciudad_CDMX', 'ciudad_Guadalajara', 'ciudad_Puebla'
]

df_matriz_limpia = df[columnas_maestras].copy()

print("--- INFORMACIÓN DATASET MAESTRO LIMPIO ---")
df_matriz_limpia.info()

print("\n--- PRIMEROS REGISTROS DEL RESULTADO FINAL ---")
print(df_matriz_limpia.head(3))

# Exportación
df_matriz_limpia.to_csv("/workspace/scratch/retail_x_clientes_procesado.csv", index=False)
print("\nDataset procesado exportado exitosamente como 'retail_x_clientes_procesado.csv'.")
```

---

## Referencias Bibliográficas (APA 7.ª Edición)

* Google. (2026). *Gemini Notebook* [Asistente de inteligencia artificial generativa]. https://notebook.google.com/
* McKinney, W. (2022). *Python for Data Analysis: Data Wrangling with Pandas, NumPy, and Jupyter* (3rd ed.). O'Reilly Media.
* VanderPlas, J. (2016). *Python Data Science Handbook: Essential Tools for Working with Data*. O'Reilly Media.
