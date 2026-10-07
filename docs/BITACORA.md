# Bitácora de iteraciones · ANI

Cada iteración partió de un diagnóstico medido con el set de 40 preguntas: qué falló, si el fallo fue de **recuperación** (el dato no llegó al modelo) o de **generación** (llegó, pero la respuesta fue incorrecta o incompleta) y qué se cambió por eso.

---

## P1 · Prototipo del agente (Felipe) → v1

**Notebook:** `notebooks/P1_prototipo_agente.ipynb` (Pasos 4 a 11D)

- Lectura de los tres PDF con PyMuPDF y corte en fragmentos citables de unos 1.500 caracteres, con documento, ubicación y páginas.
- Índice semántico con embeddings `multilingual-e5-small` y ChromaDB en dos colecciones: **norma interna** (reglamentos) y **norma externa** (Decreto 67).
- RAG simple con Gemini y llamada robusta, con reintentos y modelos de respaldo ante errores 429 y 503.
- Agente con *function calling* y 4 herramientas: buscar en norma interna, buscar en norma externa, calcular asistencia y calcular plazos en días hábiles con feriados de Chile.
- Interfaz Gradio con memoria por usuario, búsqueda obligatoria y ocultamiento de RUT, correo y teléfono.
- Regla de jerarquía normativa: el Decreto 67 prevalece sobre el reglamento (Art. 3°).

## P2 · Evaluación automática (Adolfo) → mide v1

**Notebook:** `notebooks/P2_evaluacion_IL14.ipynb` (agrega los Pasos 12 y 13)

- **Paso 12:** ANI responde las 40 preguntas del set y se mide si cita el documento y la página correctos, si rechaza lo que no está regulado y si responde bien las preguntas mixtas.
- **Paso 13:** un LLM juez evalúa con los criterios de RA1 · IL1.4: exactitud, fidelidad, relevancia, Context Precision, Context Recall y Hit@k.
- **Resultado v1:** exactitud 65 % y Context Recall 62,5 %. De 19 fallos, **12 fueron de recuperación**.

## P3a · Mejoras de recuperación (Felipe) → v2 y v3

**Notebook:** `notebooks/P3a_mejoras_recuperacion_v2_v3.ipynb` (Pasos 14 a 26)

**v2 (Pasos 14 a 16)**

- Fragmentos de 800 caracteres en vez de 1.500, para que cada dato quede menos diluido.
- **Búsqueda híbrida:** semántica más léxica (BM25, que encuentra frases exactas como "10 días hábiles"), fusionadas con *Reciprocal Rank Fusion*.
- Seis fragmentos por búsqueda, más una cuota del otro reglamento por si el agente elige mal el dominio.
- **Resultado:** exactitud 78,8 % y Context Recall 78,1 %. Quedaron 7 fallos de recuperación y 4 de generación (respuestas incompletas).

**v3 (Pasos 17 a 20)**

- **Multi-consulta:** cada búsqueda usa la consulta reformulada por el agente y también la pregunta original del apoderado.
- **Reranker** cross-encoder multilingüe que relee los 20 mejores candidatos. En el Paso 18 se compararon 4 buscadores y el reranker recuperó 5 de 6 casos difíciles.
- Reglas 11 a 13 del prompt: responder completo (todos los plazos y condiciones), citar ambas fuentes en promoción y asistencia, y rechazar sin recomendar cosas externas.
- **Resultado:** exactitud 90 % y Context Recall 87,5 %.

**Ajustes posteriores (Pasos 21 a 26)**

- **Paso 21:** simulación de un corte por puntaje para subir Context Precision. Ningún corte llegó a 50 % sin perder recall, así que no se aplicó.
- **Paso 22:** interfaz final con colores y logos del liceo, fuentes desplegables, preguntas por tema y 👍/👎 con comentario, todo en una sola pantalla.
- **Pasos 23 a 26:** caso "¿en cuánto tiempo el profe tiene que entregar las notas?" (10 días hábiles). Con "calificaciones notas" el fragmento quedaba en el lugar 56. Se agregó por código una **re-búsqueda obligatoria** antes de responder "no está regulado" o una respuesta a medias.

## P3b · Notebook limpio (Felipe) → v4 y v5

**Notebook:** `notebooks/P3b_notebook_limpio_v4_v5.ipynb`

- Se consolidaron las 26 celdas de iteración en un notebook ordenado de 10 secciones.
- **v4:** caso "atrasos" (citación al apoderado al quinto atraso). BM25 lo encontraba primero, pero el reranker lo descartaba. Se agregó una "red de seguridad léxica" que **reemplazaba** fragmentos del reranker, pero la exactitud bajó a 80 %.
- **v5:** el mejor resultado léxico se **agrega** como fragmento extra en vez de reemplazar. Exactitud 87,5 %.

## P4 · Páginas exactas y preguntas hipotéticas (Adolfo) → v6

**Notebook:** `notebooks/P4_paginas_exactas_doc2query_v6.ipynb`

- **Diagnóstico:** cada fragmento heredaba el rango de páginas de su sección completa (3,2 páginas en promedio en Evaluación, 6,9 en Convivencia y hasta 16), así que la cita de página era imprecisa.
- **Páginas exactas:** un mapa línea → página asigna a cada fragmento su página real, usando la numeración impresa del reglamento (página del PDF − 1). El resultado es 1,08 páginas por fragmento.
- Divisor recursivo (`RecursiveCharacterTextSplitter`, 800/150) con separadores propios. Los trozos de menos de 120 caracteres se unen al vecino. Total: 1.142 fragmentos.
- **Preguntas hipotéticas (doc2query):** Gemini escribió 3 preguntas tipo apoderado por fragmento (3.057 en total), que se indexan como un ranking más en la fusión RRF.
- Intercalado de los rankings de cada consulta, **anti-duplicados** (similitud > 0,95) y **contexto ampliado padre-hijo**: la sección completa si mide 2.000 caracteres o menos y, si no, el fragmento con sus vecinos.
- **Celda 4b:** comparación de estrategias de fragmentación medida solo con la recuperación, sin gastar cuota (`evaluacion/ANI_comparacion_fragmentacion.csv`).
- **Resultado:** exactitud 92,5 %, Context Recall 93,8 % y Hit@k 96,9 %. Se ejecutó sin GPU, por lo que la latencia mediana fue de 36,8 s.

## P5 · Versión final (Felipe y Adolfo) → v7

**Notebook:** `notebooks/P5_ANI_final.ipynb`, el ejecutable.

- Parte de v6. Se corrigió la etiqueta de ubicación: dentro de los protocolos del Reglamento de Convivencia (página 40 en adelante), los fragmentos se citaban con el título anterior ("Título VI…"). Ahora citan su protocolo, por ejemplo "Protocolo 6: Acoso escolar…".
- Limpieza: portada con arquitectura y resultados, sin líneas ni variables repetidas.
- Medición final con GPU T4.
- **Resultado:** exactitud 92,5 %, fidelidad 9,8, relevancia 9,7, Context Recall 93,8 %, Hit@k 96,9 %, rechazo 100 % y cita correcta 96,9 % (31/32). Latencia mediana de 19,2 s con GPU. Quedan 4 casos por mejorar: #11 y #37 (recuperación: acta del PAI y supletoriedad del Decreto 67) y #17 y #38 (generación: respuesta incompleta).

---

## Pendientes y trabajo futuro

- **Context Precision** bajo el umbral (50 %): el buscador prioriza no omitir información. Una línea futura es un corte adaptativo por pregunta.
- **Normalizar el set de preguntas:** las páginas esperadas de Evaluación usan la numeración impresa y las de Convivencia, la del PDF. Con la tolerancia de ±1 página no cambia la medición, pero conviene unificarlas.
- **Mejora continua:** usar los votos 👎 y sus comentarios, revisados por UTP, para ampliar el set de prueba.
- **Fuentes oficiales externas:** consultar en línea sitios oficiales (MINEDUC, Superintendencia de Educación) para preguntas fuera de los reglamentos.
