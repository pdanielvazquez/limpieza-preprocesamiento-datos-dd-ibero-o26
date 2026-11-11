# Informe de auditoría de calidad: `df_retail`

**Fuente:** `retail_x_clientes.csv`, cargado como `df_retail` en `auditoria-bd.ipynb`  
**Fecha de auditoría:** 4 de noviembre de 2026  
**Alcance:** completitud, tipos, unicidad, coherencia, estandarización, valores extremos y aptitud para modelado.  
**Nota:** este informe diagnostica y propone acciones; no modifica ni limpia el conjunto original.

## 1. Resumen ejecutivo

El conjunto contiene **10,000 filas y 8 columnas**. No hay filas exactamente duplicadas, pero `customer_id` no identifica una fila única: hay **4,281 IDs distintos**, de los cuales **2,950 aparecen más de una vez**. La variación de fechas, deudas y etiquetas entre filas de un mismo ID sugiere que las filas podrían ser observaciones longitudinales y no duplicados simples. Es indispensable confirmar la granularidad y definir una etiqueta `churn` por cliente y periodo antes de entrenar.

Los problemas prioritarios son:

1. **Faltantes:** 30% de `ingresos` y 10.01% de `churn`; solo 62.68% de las filas están completas.
2. **Validez:** edades negativas o mayores de 120, ingresos negativos, deuda negativa y valores extremos/sospechosos en número de compras.
3. **Consistencia de la etiqueta:** entre los IDs repetidos, **1,762 presentan tanto `churn=0` como `churn=1` en observaciones no nulas**. Puede ser correcto en datos temporales, pero no se debe tratar `churn` como etiqueta fija del cliente sin precisar el horizonte.
4. **Estandarización:** seis etiquetas de ciudad representan tres ciudades, con variación de mayúsculas y alias.

**Dictamen:** el conjunto todavía no está listo para modelado supervisado sin resolver primero la unidad de análisis, la definición temporal de `churn` y las reglas de validez. No se recomienda borrar filas por ID ni imputar automáticamente la etiqueta.

## 2. Perfil estructural y tipos

| Variable | Tipo observado | Interpretación / observación |
|---|---|---|
| `customer_id` | `int64` | Identificador; no es único por fila. No tratar como magnitud numérica. |
| `edad` | `int64` | Numérica discreta; requiere reglas de rango y validar población objetivo. |
| `ingresos` | `float64` | Numérica continua; los faltantes explican el tipo decimal. |
| `ciudad` | `str` | Categórica nominal; contiene alias y diferencias de capitalización. |
| `fecha_ultima_compra` | `str` | Fecha almacenada como texto; convertir a fecha/hora. |
| `num_compras` | `int64` | Conteo discreto; revisar ceros y valores altos. |
| `deuda` | `float64` | Numérica continua; revisar negativos y extremos con la definición contable. |
| `churn` | `float64` | Etiqueta binaria con faltantes; los valores observados son 0 y 1. El tipo decimal se debe a los nulos. |

No se observaron valores faltantes en `customer_id`, `edad`, `ciudad`, `fecha_ultima_compra`, `num_compras` ni `deuda`.

## 3. Hallazgos por dimensión de calidad

### 3.1 Completitud y faltantes

| Variable | Faltantes | Porcentaje |
|---|---:|---:|
| `ingresos` | 3,000 | 30.00% |
| `churn` | 1,001 | 10.01% |
| Cada una de las otras seis variables | 0 | 0.00% |

- **3,732 filas (37.32%)** tienen al menos un faltante; **6,268 (62.68%)** están completas.
- No hay filas totalmente vacías.
- Hay 269 filas donde `ingresos` y `churn` faltan simultáneamente.
- Entre las etiquetas conocidas, faltan ingresos en 1,358/4,494 filas con `churn=0` (30.22%) y 1,373/4,505 con `churn=1` (30.48%). La similitud de estas tasas no demuestra que los faltantes sean aleatorios.

**Implicación:** eliminar todas las filas incompletas descartaría más de un tercio del conjunto. Además, eliminar filas con `churn` faltante puede ser apropiado para entrenamiento supervisado, pero no para inferencia o scoring.

### 3.2 Unicidad y granularidad

- Duplicados exactos de fila: **0**.
- IDs distintos: **4,281**.
- IDs presentes en varias filas: **2,950**; las filas pertenecientes a esos IDs suman **8,669**.
- **1,331 IDs** aparecen una vez; el máximo es **8 filas por ID**.
- En los IDs repetidos, 2,949 tienen más de una fecha de compra distinta y los 2,950 tienen más de un valor de deuda.
- **1,762 IDs repetidos** tienen ambos valores observados de `churn` (0 y 1).

Estos resultados no justifican `drop_duplicates(subset="customer_id")`: hacerlo descartaría observaciones diferentes y podría eliminar historia útil. Se debe confirmar si cada fila es una compra, una medición periódica o una versión del perfil del cliente.

### 3.3 Estandarización y categorías

`ciudad` presenta seis etiquetas:

| Etiqueta original | Filas |
|---|---:|
| Ciudad de México | 1,994 |
| CDMX | 1,949 |
| México D.F. | 1,499 |
| Puebla | 1,973 |
| puebla | 1,576 |
| Guadalajara | 1,009 |

Al consolidar variantes obvias, quedarían tres categorías: **Ciudad de México: 5,442 (54.42%)**, **Puebla: 3,549 (35.49%)** y **Guadalajara: 1,009 (10.09%)**. Conviene conservar un mapeo explícito de alias y no depender solo de pasar el texto a minúsculas; no se detectaron ciudades faltantes.

### 3.4 Validez y coherencia de valores

| Variable | Hallazgo observado | Revisión recomendada |
|---|---|---|
| `edad` | Rango de -7 a 999; 46 valores negativos y 80 mayores de 120 (40 son 150 y 40 son 999). Hay además 305 valores entre 0 y 17. | Verificar si 150/999 son códigos centinela. Corregir desde la fuente o convertir valores imposibles en faltantes. Confirmar si los menores pertenecen a la población objetivo. |
| `ingresos` | 10 valores negativos; mínimo -3,905.13 y máximo 34,692.51. | Validar moneda, periodo y si “ingresos” admite valores negativos. No truncar sin definición de negocio. |
| `num_compras` | Rango 0–80; 67 ceros. Hay 83 filas con más de 12 compras y 60 observaciones con valor exactamente 80. | Confirmar si 80 es un tope/código o un valor real. Un cero podría ser legítimo, pero debe ser coherente con la fecha de última compra. |
| `deuda` | 243 valores negativos; mínimo -3,640.01 y máximo 50,000. Hay 51 valores mayores de 10,000 y 40 exactamente en 50,000. | Confirmar si el signo negativo representa saldo a favor/crédito y si 50,000 es un límite o dato real. |
| `churn` | 4,505 valores 1, 4,494 valores 0 y 1,001 faltantes. Entre los no nulos, no aparecen etiquetas distintas de 0 y 1. | Confirmar cómo y cuándo se asigna la etiqueta; mantener los faltantes como “sin etiqueta”, no como clase negativa. |

Los rangos de edad e ingresos/deuda son señales que requieren validación con la fuente, no autorización automática para borrar registros. Los 305 clientes con edad 0–17 tampoco son necesariamente errores: depende de la población admisible.

### 3.5 Fechas y valores extremos

- `fecha_ultima_compra` tiene **0 faltantes**, **0 fechas no parseables** al convertirla con un parser estándar y **1,200 fechas distintas**.
- Cobertura observada: **2021-01-01 a 2024-04-14**. La fecha máxima no es inválida por sí misma; validar si la antigüedad de los datos es pertinente al uso previsto.
- Como señal exploratoria, la regla IQR (1.5 × rango intercuartílico) marca 202 valores de `edad`, 41 de `ingresos` (0.59% de los valores no nulos), 191 de `num_compras` y 102 de `deuda`. La regla estadística no determina por sí sola que sean errores: deben verificarse y, si son legítimos, conservarse o tratarse según el modelo.

## 4. Propuesta de preprocesamiento

### Prioridad 0 — Definir unidad de análisis y etiqueta

1. Confirmar si una fila representa a un cliente en un momento, una transacción o una observación agregada.
2. Definir qué significa `churn`, su ventana temporal, fecha de corte y periodo de predicción.
3. Si se modela un cliente por periodo, construir una observación por cliente/fecha con agregaciones temporales justificadas. Si se requiere una sola observación por cliente, acordar cómo seleccionar la más reciente o agregar el historial; no elegir una fila arbitrariamente.
4. Resolver los 1,762 IDs con valores observados de churn 0 y 1 de acuerdo con esa definición temporal.
5. Dividir entrenamiento y validación por `customer_id` para evitar que un mismo cliente aparezca a ambos lados. Si hay predicción temporal, preferir una separación temporal que respete el horizonte de la etiqueta.

### Prioridad 1 — Tratar faltantes sin pérdida innecesaria

1. Para entrenamiento supervisado, usar solo filas con etiqueta `churn` conocida; reservar las 1,001 filas sin etiqueta para scoring u otro uso. No imputar `churn`.
2. Para `ingresos`, primero buscar recuperación desde la fuente. Si no es posible, comparar casos completos con imputación robusta (por ejemplo, mediana por grupos pertinentes como ciudad, calculada solo en entrenamiento) y añadir un indicador de faltante. Ajustar imputador e indicador dentro de cada partición o pipeline de validación.
3. Reportar completitud antes y después de cada operación y analizar si la ausencia de ingreso depende de segmentos o del proceso de captura. No asumir MCAR a partir de tasas parecidas por churn.

### Prioridad 2 — Corregir dominios y estandarizar

1. Conservar una copia cruda y crear reglas auditables para las edades imposibles; investigar específicamente los valores 150 y 999 y los negativos. Después de validar la fuente, convertir solo los valores erróneos en faltantes o corregirlos con evidencia.
2. Acordar la interpretación de ingresos negativos y deudas negativas antes de corregir signo o convertirlos en faltantes.
3. Investigar los 60 valores de 80 compras y verificar que los ceros sean compatibles con el negocio y la fecha disponible.
4. Normalizar espacios y capitalización de `ciudad`, aplicar un diccionario explícito de alias a las tres categorías canónicas y verificar que no queden etiquetas sin mapear.
5. Convertir `fecha_ultima_compra` a `datetime`; derivar recencia y variables de calendario solo respecto a una fecha de corte definida, evitando usar información posterior al momento de predicción.

### Prioridad 3 — Preparar variables para modelos

1. Excluir `customer_id` como predictor ordinario: es un identificador y puede causar memorización. Conservarlo para agrupación, auditoría y partición.
2. Codificar `ciudad` como variable nominal (por ejemplo, one-hot encoding), no como ordinal.
3. Mantener `churn` como objetivo binario con representación entera/categórica adecuada tras separar los faltantes.
4. Aplicar escalamiento solo a variables numéricas que lo requieran. Ajustar `StandardScaler` o, dada la presencia de extremos, `RobustScaler` exclusivamente con datos de entrenamiento; muchos modelos basados en árboles no requieren escalamiento.
5. No eliminar ni winsorizar automáticamente los atípicos IQR. Verificar los casos extremos y evaluar transformaciones robustas en validación.

### Prioridad 4 — Controles de aceptación

Antes del modelado, automatizar comprobaciones para asegurar que:

- las columnas esperadas, tipos, rangos y porcentajes de faltantes coinciden con el contrato de datos;
- las etiquetas no nulas de `churn` pertenecen a `{0, 1}`;
- las edades, compras, ingresos y deudas respetan reglas acordadas y documentadas;
- las ciudades se mapean completamente al catálogo canónico;
- la granularidad y la política de duplicados están definidas, sin duplicados exactos no explicados;
- las particiones no comparten clientes y respetan las fechas/horizonte de predicción;
- cualquier imputación, codificación o escalamiento se ajusta solo con entrenamiento.

## 5. Conclusión

La completitud de variables es desigual y la clave de cliente se repite ampliamente; la repetición parece contener observaciones distintas, por lo que deduplicar por ID no es apropiado sin definir la granularidad. La principal condición para avanzar no es escalar o normalizar, sino precisar la semántica temporal de `churn`, resolver las inconsistencias de etiqueta por cliente y acordar reglas de validez con el negocio. Después se pueden imputar ingresos, estandarizar ciudades, convertir fechas y construir una tubería de transformación sin fuga de información.
