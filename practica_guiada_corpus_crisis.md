# Práctica Guiada: Generador de Corpus Sintético de Comentarios y Detección de Crisis de Reputación en Redes Sociales

**Asignatura:** Minería de Datos (Clave: TCOM)  
**Programa:** Maestría en Comunicación Digital e Hipermedia — Universidad Anáhuac  
**Duración:** 120 minutos (Taller de Laboratorio en Google Colab)  
**Nivel:** Posgrado  
**Entregable:** Libreta ejecutada (`.ipynb`) + Dataset sintético exportado (`.csv` / `.jsonl`) en Brightspace  

---

## 1. Encuadre Pedagógico y Escenario de Negocio

### 1.1 El Desafío de Comunicación Digital
En el entorno actual de las redes sociales, las marcas e instituciones se enfrentan a volúmenes masivos de menciones diarias. Identificar tempranamente si un pico de comentarios negativos corresponde a una **queja aislada de servicio** o al inicio de una **crisis de reputación de marca** es crítico para los equipos de comunicación estratégica y *Social Listening*.

Sin embargo, en el mundo real, los científicos de datos y comunicadores enfrentan barreras importantes:
1. **Restricciones de Privacidad y Legales (GDPR / Leyes de Protección de Datos):** Capturar y procesar comentarios con nombres de usuarios reales sin consentimiento previo conlleva riesgos legales.
2. **Bloqueos y Costos de APIs:** Las principales plataformas sociodigitales (X/Twitter, Meta) han restringido el acceso gratuito a sus APIs de extracción masiva.
3. **Escasez de Datos de Crisis (Desbalance Severo):** Afortunadamente para las organizaciones, las crisis graves ocurren con baja frecuencia (5-10% del total de menciones), lo que dificulta el entrenamiento de modelos predictivos si no se cuenta con suficiente representación de esa clase.

### 1.2 La Solución: Bases de Datos Sintéticas Controladas
La **generación de datos sintéticos** consiste en crear conjuntos de datos artificiales que imitan matemáticamente las propiedades estadísticas, la estructura de lenguaje y las correlaciones del mundo real. 

En esta práctica, aprenderás a programar un **generador estocástico de corpus textual y metadatos** en Python. La base generada contendrá comentarios en español, métricas de difusión y etiquetas de riesgo de crisis (`nivel_riesgo`), dejándola 100% lista para alimentar modelos predictivos supervisados de clasificación multiclase.

---

## 2. Fundamentación Teórica

### 2.1 Minería de Textos (*Text Mining*) y Procesamiento de Lenguaje Natural (PLN)
La minería de texto busca transformar información textual no estructurada en representaciones numéricas estructuradas sobre las cuales se pueden aplicar algoritmos de minería de datos y aprendizaje automático (*Machine Learning*).

### 2.2 Ponderación TF-IDF (*Term Frequency - Inverse Document Frequency*)
Para que un algoritmo entienda un texto, debemos convertir los comentarios en una matriz matemática. El método **TF-IDF** evalúa la importancia de una palabra en un comentario en relación con todo el corpus de datos:

$$\text{TF-IDF}(t, d, D) = \text{TF}(t, d) \times \text{IDF}(t, D)$$

Donde:
* $\text{TF}(t, d)$: Frecuencia del término $t$ dentro del documento/comentario $d$.
* $\text{IDF}(t, D) = \log\left(\frac{N}{|\{d \in D : t \in d\}|}\right)$: Frecuencia inversa de documento, que penaliza palabras excesivamente comunes (como "el", "de", "que") y otorga mayor peso a términos discriminantes (como "fraude", "boicot", "pésimo").

### 2.3 Definición de las Clases de Riesgo de Reputación
El objetivo predictivo del modelo será clasificar cada mención en uno de tres niveles de riesgo:
* **Clase 0 — Inofensivo / Neutro-Positivo (70% de la muestra):** Preguntas de usuarios, comentarios positivos o interacción habitual.
* **Clase 1 — Queja de Servicio (20% de la muestra):** Reclamaciones operativas individuales (demoras, dudas de facturación) que no comprometen la reputación global.
* **Clase 2 — Crisis de Reputación / Alerta Temprana (10% de la muestra):** Menciones cargadas de desacreditación masiva, llamados al boicot, hashtags de protesta o virilidad de contenido negativo.

---

## 3. Desarrollo Paso a Paso en Google Colab

### Paso 0: Configuración del Entorno e Importación de Librerías

Comenzamos importando las librerías necesarias para la generación de datos, manipulación tabular, vectorización y modelado predictivo.

```python
# ==========================================
# PASO 0: IMPORTACIÓN DE LIBRERÍAS ANALÍTICAS
# ==========================================
import numpy as np
import pandas as pd
import random
import re

# Ecosistema de Scikit-Learn para PLN y Clasificación
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, confusion_matrix, roc_auc_score

# Visualización limpia
import matplotlib.pyplot as plt
import seaborn as sns

# Fijar semilla estocástica para reproducibilidad exacta
SEED = 42
np.random.seed(SEED)
random.seed(SEED)

print("✅ Entorno analítico configurado correctamente.")
```

---

### Paso 1: Generación Sintética de Lenguaje y Diccionarios Léxicos en Español

Diseñaremos un motor combinatorio de sintaxis que construya comentarios creíbles en español organizados por las tres intenciones/categorías de riesgo.

```python
# ==========================================
# PASO 1: CONSTRUCCIÓN DEL GENERADOR LÉXICO
# ==========================================

# Diccionario léxico estructurado por intenciones de comunicación
componentes_texto = {
    "Inofensivo": {
        "sujetos": ["Me encanta", "Excelente trabajo de", "Muy buena la atención de", "Felicidades a", "Interesante la propuesta de", "Me gustaría saber más sobre"],
        "entidades": ["la marca", "el equipo de soporte", "su campaña digital", "la nueva app", "la estrategia de comunicación", "el servicio al cliente"],
        "complementos": ["en su plataforma web.", "continúen así.", "me sirvió mucho la información.", "excelente experiencia de usuario.", "los recomiendo totalmente.", "quedé muy satisfecho."],
        "hashtags": ["#ExcelenteServicio", "#InnovacionDigital", "#Recomendado", "#AnahuacDigital", ""]
    },
    "Queja_Servicio": {
        "sujetos": ["Tengo problemas con", "No funciona correctamente", "Llevo días esperando respuesta de", "Pésima atención en", "Sigue fallando", "Muy lento el proceso de"],
        "entidades": ["la plataforma", "la solicitud de reembolso", "el servicio al cliente", "la aplicación móvil", "el pago en línea", "el sistema de registro"],
        "complementos": ["y nadie me resuelve nada.", "necesito ayuda urgente.", "exijo una solución hoy mismo.", "el sistema marca error.", "no responden los correos.", "es muy frustrante."],
        "hashtags": ["#FallaDeServicio", "#SoporteTecnico", "#AtencionAlCliente", "#SinRespuesta", ""]
    },
    "Crisis_Reputacion": {
        "sujetos": ["Es un verdadero fraude lo de", "Indignante el abuso de", "Exigimos justicia ante", "Totalmente inaceptable la postura de", "Basta de mentiras de", "Escándalo total en"],
        "entidades": ["esta empresa", "la dirección de la marca", "sus directivos", "la campaña engañosa", "el tratamiento de datos", "la institución"],
        "complementos": ["son unos estafadores y corruptos.", "vamos a cancelar nuestras cuentas de inmediato.", "hacemos un llamado masivo al boicot.", "deberían quitarles la licencia.", "difundan esto para que todos sepan la verdad.", "es una vergüenza pública."],
        "hashtags": ["#Boicot", "#FraudeElectoral", "#EscandaloDigital", "#CancelacionDeMarca", "#ExigimosRespuestas", "#Fraude"]
    }
}

def generar_comentario_sintetico(categoria):
    # Genera un texto sintético coherente combinando piezas léxicas según la categoría.
    componente = componentes_texto[categoria]
    suj = random.choice(componente["sujetos"])
    ent = random.choice(componente["entidades"])
    comp = random.choice(componente["complementos"])
    ht = random.choice(componente["hashtags"])
    
    # Construcción final del texto
    texto = f"{suj} {ent} {comp} {ht}".strip()
    return texto

# Prueba del generador léxico
print("--- EJEMPLOS DE TEXTOS SINTÉTICOS GENERADOS ---")
print("Inofensivo:        ", generar_comentario_sintetico("Inofensivo"))
print("Queja de Servicio: ", generar_comentario_sintetico("Queja_Servicio"))
print("Crisis de Reputación:", generar_comentario_sintetico("Crisis_Reputacion"))
```

---

### Paso 2: Generación del Corpus Tabular con Metadatos de Redes Sociales

En este paso ensamblaremos $N = 1,500$ registros que contendrán tanto el comentario en texto libre como los metadatos de difusión en la plataforma (plataforma, interacciones, alcance del usuario, etc.).

```python
# ==========================================
# PASO 2: CONSTRUCCIÓN DEL DATASET MULTIDIMENSIONAL
# ==========================================

N_MUESTRES = 1500

# Proporciones realistas del mercado (Desbalance de clases)
# 70% Inofensivo, 20% Queja de Servicio, 10% Crisis de Reputación
categorias_prob = ["Inofensivo", "Queja_Servicio", "Crisis_Reputacion"]
pesos_prob = [0.70, 0.20, 0.10]

registros = []

for i in range(N_MUESTRES):
    # Selección estocástica de la categoría real
    cat_real = np.random.choice(categorias_prob, p=pesos_prob)
    
    # Generar texto
    texto = generar_comentario_sintetico(cat_real)
    
    # Generar metadatos asociados según la categoría (inyección de patrones)
    if cat_real == "Crisis_Reputacion":
        plataforma = np.random.choice(["X_Twitter", "TikTok", "Facebook", "Instagram"], p=[0.55, 0.25, 0.10, 0.10])
        retweets = int(np.random.exponential(scale=350) + 50)  # Alto volumen de compartidos
        seguidores = int(np.random.lognormal(mean=9.5, sigma=1.2)) # Usuarios de mayor alcance
        hora = np.random.choice(range(0, 24))
        contiene_hashtag = 1 if "#" in texto else np.random.choice([0, 1], p=[0.2, 0.8])
    elif cat_real == "Queja_Servicio":
        plataforma = np.random.choice(["X_Twitter", "Facebook", "Instagram", "TikTok"], p=[0.40, 0.35, 0.15, 0.10])
        retweets = int(np.random.exponential(scale=25) + 1)
        seguidores = int(np.random.lognormal(mean=6.5, sigma=1.0))
        hora = np.random.choice(range(8, 22))  # Horario laboral
        contiene_hashtag = 1 if "#" in texto else np.random.choice([0, 1], p=[0.6, 0.4])
    else: # Inofensivo
        plataforma = np.random.choice(["Instagram", "Facebook", "TikTok", "X_Twitter"], p=[0.40, 0.30, 0.20, 0.10])
        retweets = int(np.random.exponential(scale=5))
        seguidores = int(np.random.lognormal(mean=5.5, sigma=0.8))
        hora = np.random.choice(range(6, 23))
        contiene_hashtag = 1 if "#" in texto else np.random.choice([0, 1], p=[0.7, 0.3])
    
    # Asignar la variable target numérica (0, 1, 2)
    target = 0 if cat_real == "Inofensivo" else (1 if cat_real == "Queja_Servicio" else 2)
    
    registros.append({
        "id_comentario": f"MSG_{i+1001:04d}",
        "texto_comentario": texto,
        "plataforma": plataforma,
        "retweets_compartidos": retweets,
        "seguidores_usuario": seguidores,
        "hora_publicacion": hora,
        "tiene_hashtag": contiene_hashtag,
        "categoria_etiqueta": cat_real,
        "nivel_riesgo_target": target
    })

# Convertir a DataFrame de Pandas
df_corpus = pd.DataFrame(registros)

# Inyección de nulos sintéticos en ubicacion_usuario (opcional)
df_corpus["ubicacion_usuario"] = np.random.choice(["CDMX", "Monterrey", "Guadalajara", "Puebla", np.nan], size=N_MUESTRES, p=[0.3, 0.15, 0.15, 0.1, 0.3])

print("✅ Dataset Sintético generado exitosamente. Forma:", df_corpus.shape)
print("\n--- DISTRIBUCIÓN DE LAS CLASES DE RIESGO (TARGET) ---")
print(df_corpus["categoria_etiqueta"].value_counts(normalize=True) * 100)
```

---

### Paso 3: Análisis Exploratorio del Corpus Sintético (EDA)

Inspeccionaremos las primeras filas y verificaremos las diferencias estadísticas en el nivel de interacciones entre publicaciones normales y crisis.

```python
# Visualizar las primeras filas del dataset
df_corpus.head(10)
```

```python
# ==========================================
# PASO 3: ANÁLISIS EXPLORATORIO DE DATOS
# ==========================================
print("--- MÉTRICAS PROMEDIO POR NIVEL DE RIESGO ---")
resumen_stats = df_corpus.groupby("categoria_etiqueta")[["retweets_compartidos", "seguidores_usuario", "tiene_hashtag"]].mean()
print(resumen_stats)

# Graficar la distribución de clases
plt.figure(figsize=(8, 4))
sns.countplot(data=df_corpus, x="categoria_etiqueta", palette=["#2ca02c", "#ff7f0e", "#d62728"])
plt.title("Distribución de Clases de Riesgo de Reputación (Desbalance Realista)")
plt.xlabel("Categoría de Riesgo")
plt.ylabel("Cantidad de Comentarios")
plt.grid(axis='y', linestyle='--', alpha=0.7)
plt.show()
```

---

### Paso 4: Preprocesamiento de Texto y Vectorización TF-IDF

Para alimentar el modelo predictivo, procesaremos la columna de texto removiendo puntuación y convirtiendo las cadenas a una matriz **TF-IDF**.

```python
# ==========================================
# PASO 4: PREPROCESAMIENTO Y TF-IDF
# ==========================================

# Limpieza ligera de texto mediante Expresiones Regulares (Regex)
def limpiar_texto(texto):
    texto = texto.lower() # Convertir a minúsculas
    texto = re.sub(r'[^\w\s#]', '', texto) # Mantener letras, espacios y hashtags
    return texto

df_corpus["texto_limpio"] = df_corpus["texto_comentario"].apply(limpiar_texto)

# Lista de palabras vacías (stopwords) personalizadas en español
stopwords_es = ["de", "la", "que", "el", "en", "y", "a", "los", "del", "se", "las", "por", "un", "para", "con", "no", "una", "su", "al", "lo", "como", "más", "pero", "sus", "le", "ya", "o", "este", "sí", "porque", "esta", "entre", "cuando", "muy", "sin", "sobre", "también", "me", "hasta", "hay", "donde", "quien", "desde", "todo", "nos", "durante", "todos", "uno", "les", "ni", "contra", "otros", "ese", "eso", "ante", "ellos", "e", "esto", "mí", "antes", "algunos", "qué", "unos", "yo", "otro", "otras", "otra", "él", "tanto", "esa", "estos", "mucho", "quienes", "nada", "muchos", "cual", "poco", "ella", "estar", "estas", "algunas", "algo", "nosotros"]

# Instanciar el Vectorizador TF-IDF
tfidf = TfidfVectorizer(max_features=100, stop_words=stopwords_es)

# Ajustar y transformar el cuerpo de texto
X_tfidf = tfidf.fit_transform(df_corpus["texto_limpio"]).toarray()
y = df_corpus["nivel_riesgo_target"].values

print("✅ Matriz TF-IDF generada con éxito. Dimensión de características textuales:", X_tfidf.shape)
```

---

### Paso 5: Entrenamiento y Evaluación del Clasificador Multiclase (Random Forest)

Dividiremos los datos en conjuntos de **Entrenamiento (80%)** y **Prueba (20%)** estratificados, entrenaremos un modelo predictivo `RandomForestClassifier` y evaluaremos la matriz de confusión y el reporte de clasificación.

```python
# ==========================================
# PASO 5: ENTRENAMIENTO Y EVALUACIÓN
# ==========================================

# Partición estratificada Train / Test
X_train, X_test, y_train, y_test = train_test_split(
    X_tfidf, y, test_size=0.20, random_state=SEED, stratify=y
)

# Instanciar y entrenar el modelo Random Forest
modelo_crisis = RandomForestClassifier(n_estimators=100, max_depth=8, random_state=SEED, class_weight="balanced")
modelo_crisis.fit(X_train, y_train)

# Predicciones en el conjunto de prueba
y_pred = modelo_crisis.predict(X_test)
y_prob = modelo_crisis.predict_proba(X_test)

# Reporte de Métricas
nombres_clases = ["0: Inofensivo", "1: Queja Servicio", "2: Crisis Reputación"]
print("--- REPORTE DE CLASIFICACIÓN DEL MODELO PREDICTIVO ---")
print(classification_report(y_test, y_pred, target_names=nombres_clases))

# Matriz de Confusión
cm = confusion_matrix(y_test, y_pred)
plt.figure(figsize=(7, 5))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues', xticklabels=nombres_clases, yticklabels=nombres_clases)
plt.title("Matriz de Confusión: Detección de Crisis de Reputación")
plt.xlabel("Clase Predicha por el Modelo")
plt.ylabel("Clase Real")
plt.show()
```

---

### Paso 6: Identificación de Palabras Clave y Atributos Clave de Alerta Temprana

Extraeremos la importancia de las variables (*Feature Importance*) del modelo para descubrir qué términos o palabras desencadenan la clasificación de crisis.

```python
# ==========================================
# PASO 6: IMPORTANCIA DE CARACTERÍSTICAS
# ==========================================

nombres_vocabulario = tfidf.get_feature_names_out()
importancias = modelo_crisis.feature_importances_

df_importancia = pd.DataFrame({
    "Termino": nombres_vocabulario,
    "Importancia": importancias
}).sort_values(by="Importancia", ascending=False)

print("--- TOP 10 TÉRMINOS MÁS RELEVANTES PARA EL MODELO ---")
print(df_importancia.head(10).to_string(index=False))

# Graficar los términos principales
plt.figure(figsize=(9, 4))
sns.barplot(data=df_importancia.head(10), x="Importancia", y="Termino", palette="Oranges_r")
plt.title("Top 10 Términos Clave en la Detección de Crisis de Reputación")
plt.xlabel("Puntaje de Importancia en Random Forest")
plt.ylabel("Término / Hashtag")
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.show()
```

---

### Paso 7: Exportación del Dataset Sintético Completo

Guardaremos la base sintética en formato CSV y JSONL en la carpeta local de la libreta para su posterior descarga o entrega en Brightspace.

```python
# ==========================================
# PASO 7: EXPORTACIÓN DE ARCHIVOS DE NEGOCIO
# ==========================================

ruta_csv = "corpus_sintetico_crisis_reputacion.csv"
ruta_jsonl = "corpus_sintetico_crisis_reputacion.jsonl"

# Guardar en CSV
df_corpus.to_csv(ruta_csv, index=False, encoding="utf-8-sig")

# Guardar en JSONL (formato preferido en Procesamiento de Lenguaje Natural)
df_corpus.to_json(ruta_jsonl, orient="records", lines=True, force_ascii=False)

print(f"✅ Dataset exportado exitosamente a:\n - {ruta_csv}\n - {ruta_jsonl}")
```

---

## 4. Preguntas de Reflexión y Discusión Cualitativa

1. **Ética y Gobernanza de Datos:** ¿Qué ventajas metodológicas y legales ofrece el desarrollo de datasets sintéticos para los equipos de comunicación antes de adquirir o solicitar acceso a datos reales de usuarios?
2. **Control del Desbalance de Clases:** En este laboratorio, la clase *Crisis_Reputacion* representaba únicamente el 10% del corpus. ¿Por qué el parámetro `class_weight='balanced'` en Random Forest es fundamental para evitar que el modelo ignore las alertas de crisis?
3. **Escalabilidad en Social Listening:** ¿Cómo integraría este flujo predictivo un Director de Comunicación Digital (*CCO*) para automatizar las alertas en tiempo real dentro de un tablero ejecutivo de crisis?

---

## 5. Referencias Bibliográficas (Format APA 7.ª Edición)

* **Géron, A. (2022).** *Hands-on machine learning with Scikit-Learn, Keras, and TensorFlow* (3rd ed.). O'Reilly Media.
* **Han, J., Kamber, M., & Pei, J. (2011).** *Data mining: Concepts and techniques* (3rd ed.). Morgan Kaufmann.
* **Liu, B. (2011).** *Web data mining: Exploring hyperlinks, contents, and usage data* (2nd ed.). Springer.
* **Tan, P.-N., Steinbach, M., Karpatne, A., & Kumar, V. (2019).** *Introduction to data mining* (2nd Global ed.). Pearson Education.
* **VanderPlas, J. (2016).** *Python data science handbook: Essential tools for working with data*. O'Reilly Media.
* **Witten, I. H., Frank, E., Hall, M. A., & Pal, C. J. (2017).** *Data mining: Practical machine learning tools and techniques* (4th ed.). Morgan Kaufmann.
