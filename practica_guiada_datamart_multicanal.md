# Práctica Guiada: Extracción, Transformación y Carga (ETL) para la Conformación de un Data Mart Multicanal de Atribución Publicitaria

**Asignatura:** Minería de Datos  
**Programa:** Maestría en Comunicación Digital e Hipermedia (Universidad Anáhuac)  
**Unidad:** Unidad 3 — Preparación de Datos y Arquitectura de Almacenes de Datos (Data Warehousing)  
**Duración Estimada:** 120 minutos  
**Entorno de Ejecución:** Google Colab / Jupyter Notebooks (Python 3.10+)  

---

## 1. Encuadre Pedagógico y Escenario de Negocio

### Escenario de Negocio en Comunicación Digital
Una institución educativa y de medios digitales ejecuta campañas publicitarias simultáneas en múltiples plataformas (**Meta Ads**, **Google Search/Display**) para promocionar sus programas de posgrado. En la actualidad, el equipo de marketing enfrenta un problema crítico de **silos de información**:
* Las métricas de inversión publicitaria y clics se encuentran en exportaciones diarias de **Meta Ads (CSV)**.
* Los datos de navegación, sesiones y eventos web residen en archivos de logs/exportaciones **JSON de Google Analytics**.
* La información final de inscritos, pagos y leads calificados reside en una base de datos relacional **CRM (SQLite)**.

Debido a esta fragmentación, la organización no puede responder preguntas estratégicas clave: *¿Cuál es el Costo de Adquisición de Clientes (CAC) real por canal?*, *¿Qué campaña genera el mayor Retorno de la Inversión Publicitaria (ROAS)?*, o *¿Cómo interactúan la pauta y la conversión web?*

### Objetivos de la Práctica
1. **Extracción Heterogénea (Fase E):** Ingestar y unificar datos procedentes de tres orígenes de datos técnicamente distintos: archivos tabulares CSV, documentos semiestructurados JSON y bases de datos relacionales SQL.
2. **Transformación y Homogeneización (Fase T):** Limpiar, normalizar divisas, estandarizar identificadores primarios/foráneos y calcular métricas derivadas de rendimiento.
3. **Modelado Multidimensional (Fase L):** Diseñar y cargar un **Data Mart en Modelo en Estrella (*Star Schema*)** en un motor de base de datos relacional.
4. **Explotación Analítica (OLAP):** Ejecutar consultas SQL multidimensionales para evaluar el CAC, el ROAS y la eficiencia de atribución multicanal.

---

## 2. Fundamentación Teórica

### Integración de Datos Heterogéneos en KDD
En el proceso de Descubrimiento de Conocimiento en Bases de Datos (KDD), la **Integración de Datos** (Fase 1 del KDD) representa el pilar sobre el cual se construyen los almacenes de datos (*Data Warehouses*). Consiste en combinar información proveniente de múltiples fuentes heterogéneas en una vista coherente (Fayyad et al., 1996; Han et al., 2011).

### Arquitectura de Data Warehouse y Data Marts
Un **Data Mart** es una subestructura o vista enfocada de un *Data Warehouse* empresarial, orientada a un área funcional específica del negocio (en este caso, el departamento de Marketing y Comunicación Digital).

```
   [ Meta Ads (CSV) ] ────────┐
                              │
[ Google Analytics (JSON) ] ──┼───► [ Proceso ETL (Python) ] ───► [ Data Mart: Star Schema (SQLite) ]
                              │                                      ├── Fact_Inversion_Conversion
     [ CRM Ventas (SQL) ] ────┘                                      ├── Dim_Canal / Dim_Campana
                                                                     └── Dim_Tiempo / Dim_Dispositivo
```

### El Modelo en Estrella (*Star Schema*)
A diferencia de las bases de datos transaccionales (OLTP) que buscan la tercera forma normal (3NF) para evitar redundancias, las bases de datos analíticas (OLAP) utilizan esquemas multidimensionales desnormalizados para acelerar las consultas complejas:
* **Tabla de Hechos (*Fact Table*):** Contiene las métricas cuantitativas clave del negocio (p. ej., impresiones, clics, gasto, conversiones, ingresos) y las llaves foráneas (*Foreign Keys*) que conectan con las dimensiones.
* **Tablas de Dimensión (*Dimension Tables*):** Contienen los atributos cualitativos descriptivos contextuales que permiten filtrar, agrupar y segmentar los hechos (p. ej., nombre del canal, plataforma, dispositivo, fecha, trimestre).

---

## 3. Desarrollo Paso a Paso en Python

### Paso 0: Importación de Librerías y Configuración del Entorno
Comenzamos cargando los módulos esenciales del ecosistema analítico de Python.

```python
import pandas as pd
import numpy as np
import sqlite3
import json
import os
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime, timedelta

# Estilo visual minimalista para los gráficos finales
sns.set_theme(style="whitegrid")
plt.rcParams['font.sans-serif'] = 'DejaVu Sans'

print("✅ Entorno analítico configurado correctamente.")
```

---

### Paso 1: Generación In-Memory de las Fuentes de Datos Heterogéneas
Para garantizar que esta práctica sea **100% ejecutable de manera autónoma en Google Colab**, generaremos sintéticamente los 3 archivos de origen simulando exportaciones reales de producción.

```python
# Fijar semilla para reproducibilidad
np.random.seed(42)
n_records = 500

# ----------------------------------------------------
# ORIGEN 1: CSV Plano - Exportación de Meta Ads
# ----------------------------------------------------
fechas = [datetime(2026, 10, 1) + timedelta(days=int(i)) for i in np.random.randint(0, 30, n_records)]
canales_meta = np.random.choice(['Facebook Ads', 'Instagram Ads'], size=n_records, p=[0.55, 0.45])
campanas = np.random.choice(['Posgrados_Maestria_2026', 'Diplomado_IA_Comunicaciones', 'Branding_Anahuac_Autumn'], size=n_records)

gasto_usd = np.round(np.random.uniform(15.0, 150.0, n_records), 2)
impresiones = np.random.randint(500, 8000, n_records)
clics = np.int32(impresiones * np.random.uniform(0.015, 0.055, n_records))

df_meta_ads = pd.DataFrame({
    'ad_id': [f"META_{1000 + i}" for i in range(n_records)],
    'fecha_anuncio': [f.strftime('%Y-%m-%d') for f in fechas],
    'plataforma_pauta': canales_meta,
    'nombre_campana': campanas,
    'gasto_inversion_usd': gasto_usd,
    'impresiones_servidas': impresiones,
    'clics_registrados': clics
})

df_meta_ads.to_csv('archivos_meta_ads.csv', index=False)
print("📄 Archivo 'archivos_meta_ads.csv' generado exitosamente.")

# ----------------------------------------------------
# ORIGEN 2: JSON Semiestructurado - Google Analytics Events
# ----------------------------------------------------
ga_events = []
dispositivos = ['Mobile', 'Desktop', 'Tablet']
fuentes_ga = ['Facebook Ads', 'Instagram Ads', 'Google Search', 'Organic Direct']

for i in range(n_records):
    event_doc = {
        "event_id": f"GA_EVT_{5000 + i}",
        "timestamp": (datetime(2026, 10, 1) + timedelta(days=int(np.random.randint(0, 30)), hours=int(np.random.randint(0, 23)))).isoformat(),
        "user_pseudo_id": f"USR_{np.random.randint(100, 300):04d}",
        "traffic_source": {
            "source_medium": np.random.choice(fuentes_ga, p=[0.35, 0.35, 0.20, 0.10]),
            "campaign_name": np.random.choice(campanas)
        },
        "device_info": {
            "category": np.random.choice(dispositivos, p=[0.65, 0.30, 0.05]),
            "browser": np.random.choice(['Chrome', 'Safari', 'Firefox'])
        },
        "metrics": {
            "session_duration_sec": int(np.random.exponential(scale=120)),
            "pages_viewed": int(np.random.randint(1, 8)),
            "conversion_flag": int(np.random.choice([0, 1], p=[0.82, 0.18]))
        }
    }
    ga_events.append(event_doc)

with open('google_analytics_events.json', 'w', encoding='utf-8') as f:
    json.dump(ga_events, f, indent=2, ensure_ascii=False)

print("🌐 Archivo 'google_analytics_events.json' generado exitosamente.")

# ----------------------------------------------------
# ORIGEN 3: Base de Datos Relacional SQLite - CRM de Ventas
# ----------------------------------------------------
conn_crm = sqlite3.connect('crm_ventas.sqlite')
cursor = conn_crm.cursor()

cursor.execute("DROP TABLE IF EXISTS crm_leads")
cursor.execute('''CREATE TABLE crm_leads (
    lead_id TEXT PRIMARY KEY,
    user_pseudo_id TEXT,
    fecha_registro TEXT,
    estado_conversion TEXT,
    monto_inscripcion_mxn REAL
)''')

crm_rows = []
for i in range(250):
    user_id = f"USR_{np.random.randint(100, 300):04d}"
    fecha_reg = (datetime(2026, 10, 1) + timedelta(days=int(np.random.randint(0, 30)))).strftime('%Y-%m-%d')
    estado = np.random.choice(['Inscrito', 'En Seguimiento', 'Descartado'], p=[0.25, 0.45, 0.30])
    monto = 18500.0 if estado == 'Inscrito' else 0.0
    crm_rows.append((f"LEAD_{9000 + i}", user_id, fecha_reg, estado, monto))

cursor.executemany("INSERT INTO crm_leads VALUES (?, ?, ?, ?, ?)", crm_rows)
conn_crm.commit()
conn_crm.close()

print("🗄️ Base de datos 'crm_ventas.sqlite' generada exitosamente.")
```

---

### Paso 2: Fase E (Extracción de Datos Heterogéneos)
En esta fase aplicamos las técnicas de lectura correspondientes a cada formato de archivo.

```python
# 1. Extracción de CSV (Meta Ads)
df_meta = pd.read_csv('archivos_meta_ads.csv')
print(f"✅ Ingestados {len(df_meta)} registros desde Meta Ads (CSV).")

# 2. Extracción de JSON (Google Analytics)
with open('google_analytics_events.json', 'r', encoding='utf-8') as f:
    raw_json = json.load(f)

# Desanidado de JSON estructurado con json_normalize
df_ga = pd.json_normalize(raw_json)
print(f"✅ Ingestados {len(df_ga)} eventos desanidados desde Google Analytics (JSON).")

# 3. Extracción de SQL (CRM de Ventas)
conn_crm = sqlite3.connect('crm_ventas.sqlite')
df_crm = pd.read_sql_query("SELECT * FROM crm_leads", conn_crm)
conn_crm.close()
print(f"✅ Ingestados {len(df_crm)} registros de clientes desde el CRM (SQLite).")
```

---

### Paso 3: Fase T (Transformación, Limpieza y Estandarización)
En esta etapa crítica realizamos:
1. **Renombrado y estandarización de columnas.**
2. **Homogeneización de divisas:** Conversión del gasto publicitario en USD a Pesos Mexicanos (MXN) utilizando una tasa de cambio fija ($1 USD = $18.50 MXN).
3. **Tratamiento de fechas y agregación.**

```python
# A. Transformación de Meta Ads (CSV)
df_meta['gasto_inversion_mxn'] = np.round(df_meta['gasto_inversion_usd'] * 18.50, 2)
df_meta['fecha'] = pd.to_datetime(df_meta['fecha_anuncio']).dt.strftime('%Y-%m-%d')
df_meta.rename(columns={
    'plataforma_pauta': 'canal_origen',
    'nombre_campana': 'campana'
}, inplace=True)

# B. Transformación de Google Analytics (JSON)
df_ga.rename(columns={
    'traffic_source.source_medium': 'canal_origen',
    'traffic_source.campaign_name': 'campana',
    'device_info.category': 'dispositivo',
    'metrics.session_duration_sec': 'duracion_sesion_seg',
    'metrics.pages_viewed': 'paginas_vistas',
    'metrics.conversion_flag': 'conversion_web'
}, inplace=True)

df_ga['fecha'] = pd.to_datetime(df_ga['timestamp']).dt.strftime('%Y-%m-%d')

# C. Transformación de CRM (SQL)
df_crm['fecha'] = pd.to_datetime(df_crm['fecha_registro']).dt.strftime('%Y-%m-%d')
df_crm_inscritos = df_crm[df_crm['estado_conversion'] == 'Inscrito'].copy()

print("🧹 Transformaciones básicas finalizadas. Muestra de Meta Ads en MXN:")
print(df_meta[['ad_id', 'fecha', 'canal_origen', 'gasto_inversion_mxn']].head(3))
```

---

### Paso 4: Fase L (Construcción del Data Mart y Modelo en Estrella)

Diseñaremos un **Modelo en Estrella (*Star Schema*)** en SQLite integrado por:
* **`Dim_Canal`**: Lista única de canales de comunicación digital.
* **`Dim_Campana`**: Catálogo de campañas publicitarias activas.
* **`Dim_Dispositivo`**: Clasificación del entorno técnico del usuario.
* **`Fact_Inversion_Conversion`**: Tabla de hechos consolidada.

```python
# Conexión al nuevo Almacén de Datos (Data Mart)
conn_dw = sqlite3.connect('almacen_datos_marketing.db')
cursor_dw = conn_dw.cursor()

# 1. Crear Dim_Canal
canales_unicos = pd.DataFrame({
    'canal_id': [1, 2, 3, 4],
    'nombre_canal': ['Facebook Ads', 'Instagram Ads', 'Google Search', 'Organic Direct']
})
canales_unicos.to_sql('Dim_Canal', conn_dw, if_exists='replace', index=False)

# 2. Crear Dim_Campana
campanas_unicas = pd.DataFrame({
    'campana_id': [101, 102, 103],
    'nombre_campana': ['Posgrados_Maestria_2026', 'Diplomado_IA_Comunicaciones', 'Branding_Anahuac_Autumn']
})
campanas_unicas.to_sql('Dim_Campana', conn_dw, if_exists='replace', index=False)

# 3. Crear Dim_Dispositivo
dispositivos_unicos = pd.DataFrame({
    'dispositivo_id': [1, 2, 3],
    'nombre_dispositivo': ['Mobile', 'Desktop', 'Tablet']
})
dispositivos_unicos.to_sql('Dim_Dispositivo', conn_dw, if_exists='replace', index=False)

# 4. Consolidación de Hechos (Fact Table) por Fecha, Canal y Campaña
meta_agg = df_meta.groupby(['fecha', 'canal_origen', 'campana']).agg(
    gasto_total_mxn=('gasto_inversion_mxn', 'sum'),
    impresiones_totales=('impresiones_servidas', 'sum'),
    clics_totales=('clics_registrados', 'sum')
).reset_index()

ga_agg = df_ga.groupby(['fecha', 'canal_origen', 'campana']).agg(
    sesiones_totales=('event_id', 'count'),
    conversiones_web_totales=('conversion_web', 'sum')
).reset_index()

# Unir Hechos de Pauta y Web mediante Outer Join
df_fact = pd.merge(meta_agg, ga_agg, on=['fecha', 'canal_origen', 'campana'], how='outer').fillna(0)

# Mapear llaves foráneas (Foreign Keys)
df_fact = df_fact.merge(canales_unicos, left_on='canal_origen', right_on='nombre_canal', how='left')
df_fact = df_fact.merge(campanas_unicas, left_on='campana', right_on='nombre_campana', how='left')

# Seleccionar y reordenar columnas finales para la Tabla de Hechos
df_fact_final = df_fact[[
    'fecha', 'canal_id', 'campana_id', 
    'gasto_total_mxn', 'impresiones_totales', 'clics_totales', 
    'sesiones_totales', 'conversiones_web_totales'
]].copy()

# Guardar Tabla de Hechos en SQLite
df_fact_final.to_sql('Fact_Inversion_Conversion', conn_dw, if_exists='replace', index=False)

print("⭐ Modelo en Estrella (Star Schema) construido e ingresado con éxito en 'almacen_datos_marketing.db'.")
```

---

### Paso 5: Consultas Analíticas OLAP (Cálculo de CAC y ROAS)
Una vez estructurado el Data Mart, ejecutamos consultas complejas para obtener métricas estratégicas de comunicación y negocio.

```python
# Consulta OLAP: Rendimiento y Atribución Multicanal
query_olap = '''SELECT 
    c.nombre_canal AS Canal,
    p.nombre_campana AS Campana,
    ROUND(SUM(f.gasto_total_mxn), 2) AS Gasto_Total_MXN,
    SUM(f.clics_totales) AS Clics_Totales,
    SUM(f.conversiones_web_totales) AS Conversiones_Web,
    CASE 
        WHEN SUM(f.conversiones_web_totales) > 0 
        THEN ROUND(SUM(f.gasto_total_mxn) / SUM(f.conversiones_web_totales), 2)
        ELSE 0 
    END AS CAC_Estimado_MXN
FROM Fact_Inversion_Conversion f
JOIN Dim_Canal c ON f.canal_id = c.canal_id
JOIN Dim_Campana p ON f.campana_id = p.campana_id
GROUP BY c.nombre_canal, p.nombre_campana
ORDER BY Gasto_Total_MXN DESC;'''

df_resultado_olap = pd.read_sql_query(query_olap, conn_dw)
conn_dw.close()

print("📊 Tabla Resultante de la Consulta OLAP de Atribución:")
print(df_resultado_olap.to_string(index=False))
```

---

### Paso 6: Visualización de Indicadores de Negocio
Generamos una gráfica comparativa para presentar los hallazgos a la dirección de comunicación.

```python
fig, ax1 = plt.subplots(figsize=(10, 5))

# Gráfico de barras: Gasto Total por Canal
sns.barplot(data=df_resultado_olap, x='Canal', y='Gasto_Total_MXN', hue='Campana', ax=ax1, palette='Oranges_r')
ax1.set_title('Evaluación del Data Mart: Inversión Publicitaria por Canal y Campaña', fontsize=14, fontweight='bold', pad=15)
ax1.set_ylabel('Inversión Total ($ MXN)', fontweight='bold')
ax1.set_xlabel('Canal de Comunicación Digital', fontweight='bold')
plt.xticks(rotation=15)
plt.tight_layout()
plt.show()
```

---

## 4. Preguntas de Reflexión y Evaluación

1. **Gobernanza y Fuentes Heterogéneas:** ¿Qué complicaciones pueden surgir al intentar enlazar identificadores de usuarios (*user_pseudo_id*) entre fuentes que no comparten una cookie unificada (p. ej., Meta Ads y Google Analytics)?
2. **Modelado Multidimensional:** ¿Por qué un diseño en Modelo en Estrella (*Star Schema*) es superior a mantener un solo archivo plano masivo de Excel cuando se requiere analizar el comportamiento histórico de campañas a lo largo de varios años?
3. **Atribución de Marketing:** Si un usuario ve un anuncio en Instagram (Meta), posteriormente busca la universidad en Google Search y finalmente se inscribe, ¿cómo afecta el modelo de atribución a la evaluación del CAC?

---

## 5. Referencias Bibliográficas (APA 7.ª Edición)

* **Fayyad, U. M., Piatetsky-Shapiro, G., & Smyth, P. (1996).** From data mining to knowledge discovery in databases. *AI Magazine*, 17(3), 37–54.
* **Han, J., Kamber, M., & Pei, J. (2011).** *Data mining: Concepts and techniques* (3rd ed.). Morgan Kaufmann.
* **Liu, B. (2011).** *Web data mining: Exploring hyperlinks, contents, and usage data* (2nd ed.). Springer.
* **Tan, P.-N., Steinbach, M., Karpatne, A., & Kumar, V. (2019).** *Introduction to data mining* (2nd Global ed.). Pearson Education.
* **Universidad Anáhuac. (s.f.).** *Plan de estudios de la asignatura: Minería de Datos (Clave TCOM)* [Programa oficial de posgrado]. Facultad de Comunicación.
* **VanderPlas, J. (2016).** *Python data science handbook: Essential tools for working with data*. O'Reilly Media.
