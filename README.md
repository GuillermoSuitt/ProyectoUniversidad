# ProyectoUniversidad

Desarrollo de una prueba de concepto computacional enfocada en el Procesamiento de Lenguaje Natural (NLP) biomédico.
El proyecto se realiza en colaboración con el Laboratorio de Ciencia de Datos Aplicada a las Ómicas (DATOMICS) y la Universidad Autónoma de Bucaramanga (UNAB).  

Objetivo Principal:
El objetivo es desarrollar y evaluar un pipeline (flujo de trabajo) automatizado mediante ciencia de datos e inteligencia artificial para detectar y clasificar interacciones entre medicamentos (fármaco-fármaco) presentes en textos biomédicos.

Problema que Resuelve:
La información sobre cómo interactúan los medicamentos está dispersa en miles de artículos científicos y textos médicos, lo que hace que su revisión manual sea muy costosa y lenta. Este proyecto busca automatizar la extracción de esta información para facilitar la vigilancia médica.  Funciones Clave del Código (Scripts de Python):El código desarrollado en este repositorio se encarga de:Procesamiento de Datos: Leer y procesar datos derivados del corpus especializado DrugDDI/DDI Corpus (y fuentes como DrugBank o MedLine), extrayendo textos y validando las anotaciones (entidades y offsets).  Generación de Pares: Identificar y construir pares candidatos de medicamentos mencionados dentro de una misma oración para analizar si existe interacción entre ellos.  Modelamiento y Clasificación: Implementar modelos de Machine Learning. Específicamente, se compara una línea base clásica (usando TF-IDF) frente a un modelo de lenguaje avanzado y preentrenado en biomedicina
(Transformer como BioBERT o PubMedBERT) para clasificar el tipo de interacción(ej. advise, effect, mechanism, int o ninguna).  Evaluación: Medir el rendimiento de los modelos (usando métricas como F1, precisión y matrices de confusión) y realizar análisis de errores computacionales.  
