# ANI · Asistente Normativo Institucional

Asistente tipo RAG que responde dudas de apoderados sobre evaluación, promoción y convivencia escolar del **Liceo Guillermo Labarca Hubertson**, citando siempre el documento, la ubicación y la página de donde sale la respuesta.

Todo el proyecto está en un solo notebook de Google Colab: `P2_evaluacion_IL14.ipynb`.

## Qué hace

- Lee los reglamentos en PDF y los corta en fragmentos citables (con documento, ubicación y página).
- Los indexa en **ChromaDB** con embeddings `multilingual-e5-small`, en dos colecciones: norma interna y norma externa.
- Un agente con **Gemini** decide qué herramienta usar:
  - `buscar_norma_interna`: reglamentos del liceo (dominio evaluación o convivencia).
  - `buscar_norma_externa`: Decreto 67/2018 del MINEDUC.
  - `calcular_asistencia`: porcentaje de asistencia vs. el 85% mínimo.
  - `calcular_plazo_habil`: vencimiento de plazos en días hábiles (con feriados de Chile).
- Si algo no está en los documentos, responde "No está regulado en los documentos disponibles."
- Interfaz de chat con **Gradio**, panel de trazabilidad y filtro que oculta RUT, correos y teléfonos antes de enviar el texto al modelo.

## Archivos necesarios

Deben estar en una carpeta llamada `ANI` dentro de tu Google Drive:

| Archivo | Para qué sirve |
|---|---|
| `Reglamento_Evaluacion_2026.pdf` | Norma interna de evaluación y promoción |
| `Reglamento_Convivencia.pdf` | Norma interna de convivencia escolar |
| `Decreto_67_2018.pdf` | Norma externa (MINEDUC) |
| `ANI_set_preguntas_prueba.xlsx` | Set de preguntas para evaluar (hoja `Preguntas`, subir como .xlsx) |

## Cómo usarlo

1. Abre el notebook en Google Colab.
2. Agrega tu clave en los *Secrets* de Colab con el nombre `GEMINI_API_KEY`.
3. Ejecuta las celdas en orden. La primera monta el Drive y busca la carpeta `ANI` sola.
4. Para ver el chat, ejecuta el Paso 11C (el 11 y 11B son versiones anteriores que 11C reemplaza).
5. Para evaluar, ejecuta los Pasos 12 y 13.

Con `PROBAR = True` en la primera celda se activan las pruebas de cada paso. Por defecto está en `False` para no gastar cuota.

## Pasos del notebook

| Paso | Qué hace |
|---|---|
| 6 | Limpia y fragmenta los PDFs → `fragmentos.json` |
| 7 | Crea el índice vectorial (ChromaDB) |
| 8 | Búsqueda filtrada por dominio |
| 9 / 9B | RAG simple con Gemini + reintentos y modelos de respaldo |
| 10 / 10B | Agente con herramientas y memoria de la conversación |
| 11 / 11B / 11C / 11D | Interfaz Gradio, búsqueda obligatoria, memoria por usuario y jerarquía normativa |
| 12 | Evaluación automática con el set de preguntas |
| 13 | Evaluación IL1.4 con LLM como juez |

## Evaluación

**Paso 12** mide: cita correcta (meta ≥ 90%), rechazo correcto, recuperación mixta (interna + externa), artículos inventados y falsos rechazos.

**Paso 13** mide recuperación (Context Precision, Context Recall, Hit@k) y generación (Fidelidad, Relevancia, Exactitud), con un modelo juez distinto al que responde. Además diagnostica si cada falla viene del recuperador o del generador.

Para comparar mejoras, cambia `VERSION` (`"v1"`, `"v2"`, ...) antes de volver a correr los pasos 12 y 13.

## Archivos que genera (en la carpeta `ANI`)

- `fragmentos.json`
- `ANI_resultados_evaluacion.json` y `.xlsx`, `ANI_metricas.png`
- `ANI_juez_IL14.json`, `ANI_evaluacion_IL14.xlsx` y `.png`

## Librerías

`pymupdf`, `chromadb`, `sentence-transformers`, `google-genai`, `holidays`, `gradio`, `pandas`, `matplotlib`

## Notas

- Es un apoyo informativo: no reemplaza la decisión de UTP, Convivencia o Dirección.
- Los pasos 12 y 13 se pueden retomar si se corta la cuota de Gemini; siguen desde donde quedaron.
