# Práctica Guiada: Extracción de Catálogos Dinámicos con Selenium y BeautifulSoup

**Asignatura:** Minería de Datos (Posgrado en Comunicación)  
**Programa:** Maestría en Comunicación Digital e Hipermedia — Universidad Anáhuac  
**Tema:** Minería de Contenido y Estructura Web (Web Content & Structure Mining)  
**Duración Estimada:** 120 minutos  
**Entorno de Ejecución:** Google Colab / Jupyter Notebook (Python 3.10+)  

---

## 🎯 **Objetivo de la Práctica**
Al finalizar esta práctica guiada, el estudiante será capaz de:
1. Comprender la diferencia estructural entre páginas web estáticas (renderizadas en el servidor) y portales dinámicos respaldados por **Renderizado en el Cliente (CSR)** y **carga perezosa (*Lazy Loading / Infinite Scroll*)**.
2. Configurar un entorno automatizado con **Selenium WebDriver en modo Headless** para simular la navegación y desplazamiento vertical (*scroll*) en el navegador.
3. Extraer el árbol **Document Object Model (DOM)** expandido utilizando **BeautifulSoup4** para aislar metadatos de productos (nombre, precio de venta, precio original, unidad y URL).
4. Limpiar, transformar y estructurar la información no estructurada en un dataset tabular con **Pandas**, calculando métricas de anaquel digital y exportando el resultado en formatos interoperables (**CSV** y **JSONL**).

---

## 📋 **1. Encuadre Teórico y Escenario de Negocio**

### **1.1 Escenario de Comunicación Digital e Inteligencia Competitiva**
En la gestión estratégica de canales de comercio electrónico, el análisis del **Anaquel Digital (*Digital Shelf Analytics*)** y la **Inteligencia de Precios** resultan indispensables para evaluar la visibilidad de marca, la efectividad promocional y el posicionamiento competitivo en tiempo real.

Para un analista de comunicación y marketing digital, extraer los datos de un catálogo de retail en línea (tomando como caso de estudio la categoría de *Frutas y Verduras* en plataformas de comercio electrónico) permite responder preguntas clave de negocio:
* ¿Cuál es el diferencial de precio promedio entre productos frescos y empaquetados?
* ¿Qué proporción del catálogo activo cuenta con descuentos u ofertas promocionales vigentes?
* ¿Cómo se distribuyen los productos en las primeras posiciones de visibilidad a lo largo del scroll?

---

### **1.2 El Desafío Técnico: Carga Paginada mediante Scroll (*Lazy Loading / Infinite Scroll*)**
Las plataformas modernas de e-commerce emplean **Renderizado en el Cliente (CSR)**. Cuando se realiza una petición HTTP inicial a la URL, el servidor devuelve un documento HTML base que contiene únicamente los primeros productos del catálogo.

Los productos adicionales **no existen en el documento HTML primario**; son solicitados dinámicamente mediante peticiones AJAX/Fetch a medida que el usuario desplaza la barra de navegación verticalmente (*scroll down*).

```
┌─────────────────────────┐       Petición HTTP GET Inicial       ┌────────────────────────┐
│  Navegador / Selenium   │ ────────────────────────────────────> │  Servidor Web Retail   │
│                         │ <──────────────────────────────────── │  (HTML Base Parcial)   │
└─────────────────────────┘       Devuelve HTML con ~12 ítems     └────────────────────────┘
             │
             │  (Ejecución de Script: window.scrollBy(0, 1200))
             ▼
┌─────────────────────────┐       Petición AJAX / Fetch           ┌────────────────────────┐
│ Evento JS de Scroll     │ ────────────────────────────────────> │  API Backend / Cloud   │
│ Inyecta nuevos nodos    │ <──────────────────────────────────── │  (Bloque JSON N° 2)    │
└─────────────────────────┘       Devuelve nuevos productos       └────────────────────────┘
```

Por esta razón, librerías tradicionales de extracción como `requests` o `urllib` resultan insuficientes, ya que solo capturan la respuesta inicial sin ejecutar el motor JavaScript que carga el resto de la página.

---

### **1.3 Sinergia Tecnológica: Selenium + BeautifulSoup + Pandas**

| Herramienta | Rol en el Pipeline | Función Técnica Principal |
| :--- | :--- | :--- |
| **Selenium WebDriver (Chrome Headless)** | **Navegación e Interacción Dinámica** | Simula un navegador web real sin interfaz gráfica, ejecuta scripts JavaScript de desplazamiento (*scroll*) y aguarda la inyección de nodos en el DOM. |
| **BeautifulSoup4 (`bs4`)** | **Parseo y Extracción del DOM** | Analiza el código HTML fuente completo capturado por Selenium (`driver.page_source`), aislando etiquetas y atributos mediante selectores CSS. |
| **Pandas** | **Limpieza y Estructuración Tabular** | Transforma la lista de diccionarios en un `DataFrame`, limpia cadenas monetarias a valores numéricos `float`, desduplica registros y exporta a CSV/JSONL. |

---

## 🛠️ **2. Desarrollo Paso a Paso de la Práctica en Python**

---

### **Paso 0: Configuración del Entorno Virtualizado (Selenium Headless en Colab)**

#### **Explicación del Paso**
En entornos de la nube como Google Colab o servidores Linux sin pantalla física, Selenium debe ejecutarse en **Modo Headless** (sin ventana gráfica). En esta etapa instalamos las dependencias del sistema operativo y configuramos el objeto `ChromeOptions` para la ejecución asíncrona.

```python
# ==============================================================================
# PASO 0: Configuración del Entorno de Selenium Headless
# ==============================================================================
import time
import re
import pandas as pd
from bs4 import BeautifulSoup

# Importación de componentes clave de Selenium WebDriver
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def iniciar_driver_headless():
    """
    Inicializa una instancia de Chrome WebDriver configurada en modo Headless
    optimizada para entornos Linux virtualizados (Google Colab / Jupyter).
    """
    options = webdriver.ChromeOptions()
    options.add_argument('--headless')                  # Modo sin interfaz gráfica
    options.add_argument('--no-sandbox')               # Requerido para entornos de contenedores
    options.add_argument('--disable-dev-shm-usage')    # Optimización de memoria compartida
    options.add_argument('--disable-gpu')               # Desactiva aceleración por GPU
    
    # En entornos Linux / Google Colab la ruta predeterminada del ejecutable es:
    service = Service('/usr/bin/chromedriver')
    driver = webdriver.Chrome(service=service, options=options)
    driver.set_window_size(1920, 1080)
    return driver

print("✅ Entorno de Selenium Headless configurado exitosamente.")
```

---

### **Paso 1: Navegación Automatizada y Bucle de Desplazamiento Vertical (*Scroll Loop*)**

#### **Explicación del Paso**
En este paso, Selenium abre la URL del catálogo de *Frutas y Verduras*. A continuación, ejecutamos un bucle `for` que invoca el comando de JavaScript `window.scrollBy(0, 1200)` para desplazar la pantalla hacia abajo. Entre cada iteración, agregamos una pausa de sincronización (`time.sleep(2.5)`) para permitir que la petición AJAX responda e inyecte los nuevos nodos de productos en el árbol DOM. Al finalizar, capturamos el código HTML completo con `driver.page_source`.

```python
# ==============================================================================
# PASO 1: Apertura de la URL y Ejecución del Bucle de Scroll
# ==============================================================================
URL_CATALOGO = "https://example.com/frutas-y-verduras"

# Inicializar el navegador en segundo plano
driver = iniciar_driver_headless()
print(f"🚀 Conectando a la plataforma objetivo: {URL_CATALOGO}")

try:
    driver.get(URL_CATALOGO)
    time.sleep(4)  # Espera inicial para el renderizado del documento base

    # Parámetros del desplazamiento vertical
    MAX_SCROLLS = 8              # Número de iteraciones de desplazamiento
    PAUSA_AJAX = 2.5             # Segundos de espera para la carga de nuevos nodos
    PIXELES_DESPLAZAMIENTO = 1200 # Píxeles a desplazar por cada ciclo

    print("\n🔄 Iniciando secuencia de desplazamiento vertical (Scroll Loop)...")

    for i in range(1, MAX_SCROLLS + 1):
        # Ejecución de script JavaScript en el navegador simulado
        driver.execute_script(f"window.scrollBy(0, {PIXELES_DESPLAZAMIENTO});")
        print(f"   ── Iteración #{i}/{MAX_SCROLLS}: Scroll ejecutado. Esperando respuesta del servidor...")
        time.sleep(PAUSA_AJAX)
        
        # Conteo en tiempo real de nodos de producto presentes en el DOM
        tarjetas_detectadas = driver.find_elements(By.CSS_SELECTOR, "div.product-item, div.product-card, div.vtex-search-result-3-x-galleryItem")
        print(f"      [DOM Check] Nodos de productos detectados actualmente: {len(tarjetas_detectadas)}")

    # Captura del código HTML fuente totalmente renderizado y expandido
    html_fuente_renderizado = driver.page_source
    print("\n✅ Extracción del HTML renderizado finalizada correctamente.")

finally:
    driver.quit()
    print("🔒 Instancia de Selenium cerrada de forma segura.")
```

---

### **Paso 2: Parseo Quirúrgico del DOM con BeautifulSoup**

#### **Explicación del Paso**
Con el HTML fuente cargado en memoria (`html_fuente_renderizado`), instanciamos BeautifulSoup para analizar la estructura de nodos. Utilizando expresiones regulares (`re.compile`) y selectores CSS flexibles, localizamos los contenedores de los productos y extraemos 5 atributos fundamentales:
1. **Posición en Anaquel:** Orden de aparición del producto durante el scroll.
2. **Nombre del Producto:** Título comercial del producto.
3. **Precio de Venta:** Precio actual de compra (cadena de texto original).
4. **Precio Lista / Anterior:** Precio original sin descuento (si aplica).
5. **Unidad de Medida / URL:** Presentación (kg, pieza) y enlace directo.

```python
# ==============================================================================
# PASO 2: Extracción de Metadatos con BeautifulSoup
# ==============================================================================
soup = BeautifulSoup(html_fuente_renderizado, 'html.parser')

# Búsqueda defensiva de contenedores de tarjetas de producto
tarjetas_productos = soup.find_all(['div', 'article'], class_=re.compile(r'(product-item|product-card|vtex-search-result|product-summary)'))

if not tarjetas_productos:
    tarjetas_productos = soup.select('.vtex-search-result-3-x-galleryItem, .product-item, .shelf-item')

print(f"📦 Total de contenedores de productos aislados para parseo: {len(tarjetas_productos)}")

registros_extraidos = []

for idx, tarjeta in enumerate(tarjetas_productos, start=1):
    # 1. Extracción del Nombre
    nodo_nombre = tarjeta.find(['h2', 'h3', 'span'], class_=re.compile(r'(name|title|product-name|vtex-product-summary)'))
    nombre = nodo_nombre.get_text(strip=True) if nodo_nombre else "Producto Sin Nombre"
    
    # 2. Extracción del Precio de Venta (Actual)
    nodo_precio = tarjeta.find(['span', 'div'], class_=re.compile(r'(price|selling-price|vtex-product-price)'))
    precio_raw = nodo_precio.get_text(strip=True) if nodo_precio else None
    
    # 3. Extracción del Precio de Lista (Anterior / Descuento)
    nodo_precio_lista = tarjeta.find(['span', 'del', 's'], class_=re.compile(r'(list-price|old-price|strike)'))
    precio_lista_raw = nodo_precio_lista.get_text(strip=True) if nodo_precio_lista else None

    # 4. Extracción de la Unidad de Medida
    nodo_unidad = tarjeta.find(['span', 'small'], class_=re.compile(r'(unit|measurement|unidade)'))
    unidad = nodo_unidad.get_text(strip=True) if nodo_unidad else "No especificada"

    # 5. Extracción de la URL del Enlace
    nodo_enlace = tarjeta.find('a', href=True)
    url_producto = nodo_enlace['href'] if nodo_enlace else ""
    if url_producto and not url_producto.startswith('http'):
        url_producto = f"https://example.com{url_producto}"

    # Almacenamiento si el producto es válido
    if nombre != "Producto Sin Nombre":
        registros_extraidos.append({
            "posicion_anaquel": idx,
            "producto": nombre,
            "precio_venta_raw": precio_raw,
            "precio_lista_raw": precio_lista_raw,
            "unidad": unidad,
            "url_producto": url_producto
        })

print(f"✨ Extracción de texto completada. {len(registros_extraidos)} registros almacenados.")
```

---

### **Paso 3: Limpieza de Datos, Expresiones Regulares y Conversión Tabular en Pandas**

#### **Explicación del Paso**
Los precios extraídos del HTML suelen contener caracteres no numéricos (`"$24.50 / kg"`). En este paso, construimos una función auxiliar basada en expresiones regulares (`re.search`) para aislar la parte numérica y convertirla a tipo `float`. Posteriormente, calculamos variables analíticas derivadas:
* **`tiene_oferta` (Booleano):** Indica si el producto cuenta con descuento.
* **`descuento_monto` (Flotante):** Diferencia en pesos entre el precio original y el precio de venta.
* **Desduplicación:** Eliminación de registros repetidos según el nombre del producto.

```python
# ==============================================================================
# PASO 3: Limpieza y Conversión a DataFrame de Pandas
# ==============================================================================
df_catalogo = pd.DataFrame(registros_extraidos)

def limpiar_precio_flotante(cadena_precio):
    """
    Parsea cadenas de texto monetarias ('$35.90 / kg') y devuelve un valor numérico float.
    """
    if pd.isna(cadena_precio) or not cadena_precio:
        return None
    match = re.search(r'[\d\.,]+', str(cadena_precio))
    if match:
        num_clean = match.group(0).replace(',', '')
        try:
            return float(num_clean)
        except ValueError:
            return None
    return None

# Aplicación de limpieza numérica de precios
df_catalogo['precio_venta_num'] = df_catalogo['precio_venta_raw'].apply(limpiar_precio_flotante)
df_catalogo['precio_lista_num'] = df_catalogo['precio_lista_raw'].apply(limpiar_precio_flotante)

# Cálculo de indicadores promocionales
df_catalogo['tiene_oferta'] = df_catalogo['precio_lista_num'].notna() & (df_catalogo['precio_lista_num'] > df_catalogo['precio_venta_num'])
df_catalogo['descuento_monto'] = df_catalogo.apply(
    lambda row: round(row['precio_lista_num'] - row['precio_venta_num'], 2) if row['tiene_oferta'] else 0.0,
    axis=1
)

# Desduplicación de registros
df_catalogo = df_catalogo.drop_duplicates(subset=['producto']).reset_index(drop=True)

print("📊 Muestra del Dataset Tabular Limpio y Estructurado:")
print(df_catalogo[['posicion_anaquel', 'producto', 'precio_venta_num', 'precio_lista_num', 'tiene_oferta', 'descuento_monto']].head(10))
```

---

### **Paso 4: Exportación de Datasets y Análisis Exploratorio del Anaquel Digital**

#### **Explicación del Paso**
Una vez estructurado el dataset, procedemos a exportarlo en dos formatos estándar de la industria:
1. **`CSV` (UTF-8 con BOM):** Compatible con Excel, PowerBI y software de inteligencia de negocios.
2. **`JSONL` (JSON Lines):** Formato ideal para consumo por parte de modelos de lenguaje, pipelines de Big Data y bases de datos NoSQL.

Finalmente, calculamos el resumen de métricas del anaquel digital.

```python
# ==============================================================================
# PASO 4: Exportación y Análisis Resumen del Anaquel Digital
# ==============================================================================
ARCHIVO_CSV = "catalogo_frutas_verduras.csv"
ARCHIVO_JSONL = "catalogo_frutas_verduras.jsonl"

# Exportación de archivos
df_catalogo.to_csv(ARCHIVO_CSV, index=False, encoding='utf-8-sig')
df_catalogo.to_json(ARCHIVO_JSONL, orient='records', lines=True, force_ascii=False)

print(f"📁 Dataset exportado exitosamente a CSV: {ARCHIVO_CSV}")
print(f"📁 Dataset exportado exitosamente a JSONL: {ARCHIVO_JSONL}")

# Cálculo de Estadísticas Descriptivas del Anaquel Digital
total_items = len(df_catalogo)
items_oferta = int(df_catalogo['tiene_oferta'].sum())
precio_prom = df_catalogo['precio_venta_num'].mean()
precio_max = df_catalogo['precio_venta_num'].max()
precio_min = df_catalogo['precio_venta_num'].min()

print("\n" + "="*65)
print("📈 DIAGNÓSTICO DEL ANAQUEL DIGITAL: FRUTAS Y VERDURAS")
print("="*65)
print(f"• Total de productos únicos en el catálogo: {total_items}")
if total_items > 0:
    pct_oferta = (items_oferta / total_items) * 100
    print(f"• Productos con promoción/oferta activa:  {items_oferta} ({pct_oferta:.1f}%)")
    print(f"• Precio de venta promedio:                ${precio_prom:.2f} MXN")
    print(f"• Rango de precios observado:             Mín: ${precio_min:.2f} MXN | Máx: ${precio_max:.2f} MXN")
print("="*65)
```

---

## ❓ **3. Preguntas de Reflexión y Evaluación Cualitativa**

1. **Fragilidad de Selectores CSS vs. Componentes Dinámicos:**  
   *¿Por qué los selectores CSS basados en clases generadas por frameworks como VTEX o React pueden cambiar en una actualización del sitio web, y qué alternativas de localización (XPath por jerarquía o atributos de datos) ofrecen mayor estabilidad al script?*

2. **Evaluación de Eficiencia: Selenium vs. APIs REST Ocultas:**  
   *Al realizar el scroll en la página, el navegador ejecuta peticiones HTTP internas a endpoints de backend en formato JSON. ¿En qué casos resulta más eficiente inspeccionar la pestaña "Network" de las herramientas de desarrollador e interrogar directamente la API REST con `requests`, en lugar de renderizar Selenium y parsear el HTML?*

3. **Gobernanza y Ética de Extracción Web:**  
   *¿Qué impacto técnico tiene omitir las pausas de sincronización (`time.sleep`) durante un bucle de scroll automatizado masivo, y cómo se relaciona esta práctica con el cumplimiento del protocolo `robots.txt` y la gestión responsable de la tasa de peticiones (rate limiting)?*

---

## 📚 **4. Referencias Bibliográficas (APA 7.ª Edición)**

* **Fayyad, U. M., Piatetsky-Shapiro, G., & Smyth, P. (1996).** From data mining to knowledge discovery in databases. *AI Magazine*, 17(3), 37–54.
* **Liu, B. (2011).** *Web data mining: Exploring hyperlinks, contents, and usage data* (2nd ed.). Springer.
* **Tan, P.-N., Steinbach, M., Karpatne, A., & Kumar, V. (2019).** *Introduction to data mining* (2nd Global ed.). Pearson Education.
* **VanderPlas, J. (2016).** *Python data science handbook: Essential tools for working with data*. O'Reilly Media.
