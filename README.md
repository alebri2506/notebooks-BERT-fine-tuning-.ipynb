# Detección de Intrusiones en Tráfico IoT mediante Fine-Tuning de BERT (LogBERT)

Sistema de Detección de Intrusiones (IDS) basado en el modelo de lenguaje **BERT** (`bert-base-uncased`) aplicado a la clasificación y detección de ciberataques en redes IoT mediante la serialización semántica de logs de red (*Tabular-to-Text*).

---

## 1. Objetivo del Proyecto

El objetivo principal de este proyecto es adaptar y evaluar la arquitectura Transformer de **BERT** para la ciberseguridad en entornos de Internet de las Cosas (IoT). Específicamente, se busca:
- Transformar registros de tráfico estructurado de red en oraciones descriptivas de lenguaje natural comprensibles por modelos de procesamiento de lenguaje natural (NLP).
- Clasificar tráfico de red en múltiples categorías (tráfico benigno y distintos tipos de malware) sin necesidad de ingeniería manual avanzada de características estadísticas.
- Evaluar el desempeño de generalización del modelo frente a condiciones realistas de tráfico futuro mediante una partición temporal estricta.

---

## 2. Dataset Utilizado

El proyecto utiliza datos extraídos del dataset de referencia **IoT-23**, específicamente la captura etiquetada generada por el analizador de red **Zeek**:
`CTU-IoT-Malware-Capture-34-1conn.log.labeled.csv`.

### Características del Dataset:
- **Volumen total:** 23,145 registros de conexiones de red.
- **Estructura inicial:** 23 columnas con identificadores, métricas de conexión y etiquetas de seguridad.
- **Clases detectadas (4 etiquetas):**
  1. `Benign`: Tráfico regular y seguro de la red.
  2. `Malicious   C&C`: Conexiones hacia servidores de Comando y Control de botnets.
  3. `Malicious   DDoS`: Ataques de denegación de servicio distribuido.
  4. `Malicious   PartOfAHorizontalPortScan`: Intentos de escaneo horizontal de puertos.

### Variables Seleccionadas (12 Atributos):
Se descartaron los identificadores unívocos (`uid`, IPs de origen y destino, timestamps) para prevenir memorización y fuga de información (*data leakage*). Las variables empleadas son:
- **Identificación de servicios:** `id.resp_p` (puerto destino), `proto` (protocolo de transporte), `service` (servicio de aplicación).
- **Comportamiento de la sesión:** `conn_state` (estado de la conexión), `history` (secuencia de flags TCP), `duration` (duración en segundos).
- **Métricas de volumen y paquetes:** `orig_bytes`, `resp_bytes`, `orig_ip_bytes`, `resp_ip_bytes`, `orig_pkts`, `resp_pkts`.

---

## 3. Metodología

### 3.1. Limpieza e Imputación Semántica
En los logs de Zeek, el símbolo `-` indica ausencia de transmisión técnica, no un dato perdido convencional:
- **Variable `service`:** Se imputó `-` por `'unknown'`, permitiendo al modelo aprender que la falta de resolución de servicio suele correlacionarse con tráfico sospechoso.
- **Variables `orig_bytes`, `resp_bytes` y `duration`:** Se imputaron con valor `0`, reflejando la realidad física de conexiones sin carga útil transferida.

### 3.2. Transformación Tabular a Texto (Tabular-to-Text)
Cada fila del dataset se convirtió en una secuencia textual estructurada mediante un mapeo de etiquetas descriptivas. 

**Ejemplo de entrada procesada:**
> `"connection history is S. duration seconds is 3. total bytes sent by origin is 180. total bytes sent by responder is 0. connection state is S0. application service is unknown. transport protocol is tcp. destination port is 80. packets sent by origin is 3. packets sent by responder is 0. payload bytes sent by origin is 0. payload bytes sent by responder is 0."`

### 3.3. División de Datos Temporal
Para emular un despliegue en tiempo real y evitar el sesgo de mezclar pasado con futuro:
- **Entrenamiento (80% inicial):** 18,516 registros cronológicamente más antiguos.
- **Evaluación / Test (20% final):** 4,629 registros cronológicamente posteriores (`shuffle=False`).

### 3.4. Tokenización y Fine-Tuning de BERT
- **Tokenizador:** `BertTokenizer` (`bert-base-uncased`) con longitud máxima de 300 tokens (`truncation=True`, `padding="max_length"`).
- **Arquitectura:** `BertForSequenceClassification` con cabeza de clasificación lineal multiclase.
- **Configuración de entrenamiento:**
  - Épocas: 3.
  - Tamaño de lote (Batch Size): 16.
  - Tasa de aprendizaje (Learning Rate): $2 \times 10^{-5}$.
  - Decaimiento de peso (Weight Decay): 0.01.
  - Guardado del mejor modelo al finalizar la evaluación (`load_best_model_at_end=True`).

---

## 4. Resultados

### 4.1. Métricas Globales (Conjunto de Evaluación)
Evaluación sobre 4,629 conexiones de prueba en la partición temporal:

| Métrica | Resultado |
| :--- | :--- |
| **Exactitud (Accuracy)** | **97.11%** (4,495 / 4,629 aciertos) |
| **Precisión Ponderada (Precision)** | **98.39%** |
| **Sensibilidad Ponderada (Recall)** | **97.11%** |
| **F1-Score Ponderado** | **97.48%** |
| **Total de Errores** | **134 muestras** |

### 4.2. Reporte de Clasificación por Clase

| Clase | Precisión | Sensibilidad (Recall) | F1-Score | Muestras Reales (Support) |
| :--- | :---: | :---: | :---: | :---: |
| **Benign** | 0.56 | 1.00 | 0.72 | 170 |
| **Malicious   C&C** | 1.00 | 1.00 | 1.00 | 2,198 |
| **Malicious   DDoS** | 1.00 | 0.94 | 0.97 | 2,261 |
| **Malicious   PartOfAHorizontalPortScan** | 0.00 | 0.00 | 0.00 | 0 |

### 4.3. Análisis de Errores y Matriz de Confusión
- De los 134 errores totales, **133 correspondieron a ataques DDoS clasificados erróneamente como tráfico benigno**.
- Esta confusión redujo la precisión de la clase `Benign` al 56%: el modelo predijo "Benigno" en 303 ocasiones, pero sólo 170 eran realmente legítimas (las restantes 133 eran ataques camuflados con características similares a peticiones web normales).
- La clase de escaneo de puertos no tuvo presencia en el set de prueba debido a que los escaneos ocurrieron como ráfagas aisladas en la ventana de tiempo del conjunto de entrenamiento.

### 4.4. Curva de Pérdida y Dinámica de Entrenamiento

| Época | Training Loss | Validation Loss | Accuracy (%) |
| :---: | :---: | :---: | :---: |
| 1 | 0.000300 | 0.173028 | 97.11% |
| 2 | 0.000100 | 0.267726 | 97.11% |
| 3 | 0.000100 | 0.291691 | 97.11% |

Se evidencia **sobreajuste (overfitting)**: la pérdida de entrenamiento desciende a valores cercanos a cero (memorización), mientras que la pérdida de validación asciende de 0.17 a 0.29 a lo largo de las épocas.

---

## 5. Conclusiones

1. **Efectividad del Enfoque Textual:** La conversión de telemetría de red a oraciones descriptivas permite a los Transformers extraer correlaciones complejas entre puertos, servicios y volumen de tráfico, alcanzando un 97.11% de exactitud global.
2. **Detección Perfecta de C&C:** El modelo identificó el tráfico de comando y control de botnets con un F1-Score de 1.00, reconociendo la combinación de conexiones persistentes de bajo volumen en puertos característicos (e.g., IRC 6667).
3. **Vulnerabilidad ante Ataques Camuflados:** Las conexiones DDoS dirigidas al puerto 80 que usan flags o estructuras similares a intentos HTTP fallidos generaron los falsos negativos del modelo, confundiéndose con tráfico benigno.
4. **Líneas de Mejora:** Para mitigar el sobreajuste y mejorar la discriminación en zonas ambiguas, se recomienda incorporar parada temprana (*early stopping*), regularización con mayor tasa de *dropout*, ajuste de funciones de pérdida penalizadas (e.g., *Focal Loss*) y entrenamiento con tráfico sintético adversario con solapamiento controlado.
