# 🧬 Detección de Interacciones Fármaco-Fármaco (DDI) con NLP

Proyecto de Práctica Profesional — DATOMICS · UNAB
Equipo de 3 integrantes · Supervisor: Dr. Eduardo Martínez

> 👥 **Trabajo grupal.** El pipeline completo (limpieza, extracción, etiquetado manual y modelos) se desarrolló en conjunto por un equipo de 3 estudiantes, entre ellos Guillermo Suitt.

⚠️ **Aviso de uso:** Este repositorio contiene un prototipo experimental de NLP. No constituye una herramienta de diagnóstico clínico, no reemplaza fuentes farmacológicas oficiales y no debe usarse para tomar decisiones de prescripción. Toda salida del modelo requiere supervisión humana.

Este repositorio contiene los **scripts del pipeline** de procesamiento, extracción y modelado desarrollados para detectar y clasificar interacciones fármaco-fármaco (DDI) a partir de DrugBank y una base clínica local. **Los datos, la muestra etiquetada y los pesos de los modelos no se incluyen** por restricciones de tamaño y licencia.

## 📋 Contexto y Metodología

El objetivo del proyecto es entrenar un modelo de NLP que, dada una oración clínica y un par de fármacos, determine si existe una interacción real y de qué tipo.

**Hito metodológico clave:** se descartó la *Closed-World Assumption* (generar negativos artificiales con texto de relleno). En su lugar, se implementó una **extracción trazable** mediante patrones de texto (negación/afirmación) sobre las fichas clínicas, cruzadas contra un vocabulario de 4.333 fármacos. Cada fila del dataset queda asociada a una **oración trazable hasta DrugBank o la base clínica local**, no a texto inventado.

**Etiquetado:** las filas fueron etiquetadas manualmente por el equipo, siguiendo criterios definidos de antemano que incluían palabras clave (por ejemplo *should be avoided*, *metabolism*, *risk*). Los casos ambiguos se agruparon bajo la clase `duda_vaga`, que fusiona los cuatro subtipos de duda (origen, destino, no atómica y vaga).

> **Nota sobre el tamaño del dataset:** la versión utilizada ronda las **2.700 filas**; el número exacto depende del archivo de etiquetado final, que no se incluye en este repositorio.

## 🗂️ Estructura del Repositorio

### 1. Preprocesamiento y Extracción

- **`01_limpieza_drugbank.py`**: limpieza del archivo original `drugbank.csv`, corrección de filas rotas y generación de un diccionario médico.
- **`02_cruce_y_extraccion.py`**: cruce con la base clínica local y extracción de pares de fármacos (E1/E2) y sus oraciones mediante expresiones regulares.

**Nota:** las correcciones de calidad aplicadas al dataset (desduplicación por mayúsculas/minúsculas y reasignación de destinos en listas) se realizaron en una etapa de curación posterior a la extracción automática y **no están automatizadas en estos scripts**.

### 2. Modelos Clásicos (Línea Base)

- **`03_modelo_svm.py`**: SVM lineal con TF-IDF, entrenado **descartando** las filas ambiguas (4 clases: `effect`, `advise`, `mechanism`, `no_interaction`).
- **`03.1_modelo_svm_8clases.py`**: SVM lineal con TF-IDF, entrenado **subdividiendo** las filas de "duda" en 4 subcategorías (origen, destino, no atómica, vaga), totalizando 8 clases.

### 3. Modelos de Deep Learning (Transformers)

- **`04_modelo_biobert.py`**: fine-tuning de BioBERT v1.1 con 3 épocas fijas.
- **`05_modelo_biobert_v2.py`**: BioBERT v1.1 con Early Stopping (patience=2, hasta 10 épocas) y marcado de entidades E1/E2.
- **`06_modelo_pubmedbert.py`**: fine-tuning de PubMedBERT con Early Stopping y marcado de entidades E1/E2.

## 🏆 Resultados

### Modelos Transformers (5 clases)

Evaluación con **partición estratificada 80/20** mediante `train_test_split` con `stratify` sobre la clase.

| Modelo | Arquitectura | Épocas | F1 Macro | F1 Weighted | Accuracy |
|---|---|---|---|---|---|
| Modelo 3 | BioBERT v1.1 | 3 (fijas) | 0.8464 | 0.8762 | 0.88 |
| Modelo 3.1 | BioBERT v1.1 + Early Stopping | 7 (early stop) | 0.8685 | 0.8945 | 0.90 |
| **Modelo 4 (mayor F1 macro)** | **PubMedBERT + Early Stopping** | **7 (early stop)** | **0.8759** | **0.9000** | **0.90** |

### Modelos Clásicos (SVM, 4 y 8 clases)

> ⚠️ **No comparables directamente con los transformers:** el SVM de `03` se entrenó con **4 clases** (descartando las dudas) y el de `03.1` con **8 clases**, mientras que los transformers usan **5 clases**. Distinto esquema de clases significa tareas distintas, por lo que no se debe concluir que los transformers superan al SVM.

Los resultados del SVM se generan al ejecutar los scripts (el script `03.1` guarda un `estadisticas.json`), pero **no se incluyen en este repositorio** los artefactos de esa corrida.

### Lectura de los resultados

- **Mejora de 2,21 puntos de F1 Macro** entre Modelo 3 y Modelo 3.1. No puede atribuirse únicamente al Early Stopping, porque también cambió el máximo de épocas (de 3 a hasta 10). Un experimento de ablación permitiría separar el efecto de cada cambio.
- **Diferencia de 0,74 puntos** entre Modelo 3.1 (BioBERT) y Modelo 4 (PubMedBERT). Una **posible explicación** es el pre-entrenamiento en dominio biomédico y su vocabulario especializado, pero no se probó causalmente: es una sola corrida por modelo.

> **Nota metodológica:** la partición fue aleatoria estratificada. Además, el conjunto de prueba se utilizó también como `eval_dataset` durante el Early Stopping, por lo que la estimación de rendimiento es **optimista**. Como mejoras futuras se sugieren: (1) una **partición agrupada por fármaco** (`GroupShuffleSplit`) para evitar fuga de información entre oraciones del mismo fármaco, y (2) separar un **conjunto de validación** independiente del de prueba.

### Análisis de Errores (hipótesis)

- **Posible confusión `mechanism` vs `effect`**: el solapamiento lingüístico en la literatura biomédica podría hacer que modificaciones farmacocinéticas se confundan con manifestaciones clínicas. Debe verificarse revisando la matriz de confusión del modelo final.
- **Heterogeneidad de `duda_vaga`**: al fusionar cuatro subtipos lingüísticamente dispares, el modelo podría no generalizar un patrón sintáctico único. También debe contrastarse con la matriz de confusión.

## 🛠️ Tecnologías Utilizadas

- **Datos**: pandas, numpy, openpyxl, re (regex)
- **Machine Learning Clásico**: scikit-learn (TF-IDF, LinearSVC, métricas)
- **Deep Learning**: transformers (Hugging Face), torch (PyTorch), datasets
- **Visualización**: matplotlib, seaborn

## 📦 Qué incluye y qué no incluye este repositorio

**Incluye:**
- Scripts del pipeline (limpieza, cruce, extracción, modelado).
- Configuración de hiperparámetros y estructura de entrenamiento.

**No incluye:**
- Los datos brutos (`drugbank.csv`, `ddiData.xlsx`).
- La muestra etiquetada final (`Muestra_Etiquetado`).
- Los pesos de los modelos entrenados (`*.pkl`, `modelo/`, `tokenizer/`).
- Los resultados guardados de los SVM ni los archivos `estadisticas.json` de cada modelo.

## 🚀 ¿Cómo ejecutar este proyecto?

Este proyecto fue desarrollado y ejecutado en **Google Colab**. Para replicarlo:

1. Clona este repositorio.
   Instala las dependencias con `pip install -r requirements.txt` (en Colab ya vienen casi todas).
2. Sube tus archivos de datos (`drugbank.csv` y `ddiData.xlsx`) a tu Google Drive.
3. Modifica las rutas absolutas dentro de los scripts (`/content/drive/MyDrive/...`) para que apunten a tus archivos.
4. Para los modelos de Transformers, asegúrate de tener activada la GPU (**Entorno de ejecución > Cambiar tipo de entorno > GPU T4**).
5. Si ejecutas los scripts fuera de Colab, elimina las líneas que empiezan con `!` (por ejemplo `!pip install ...`) y el montaje de Drive.
