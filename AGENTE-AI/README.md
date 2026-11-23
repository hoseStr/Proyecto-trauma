# Chatbot de trauma — Registro de Trauma del HUV

Asistente local de apoyo al diagnóstico en trauma. Responde desde dos fuentes,
con cita en cada afirmación:

- **Casos del registro**: 8.198 casos anonimizados del Registro de Trauma del
  HUV (`[caso: ID]`).
- **Manuales**: ATLS 11.ª ed., ATLS 10.ª ed. en español y el manual de manejo
  inicial del paciente traumatizado del HUV (`[guía: ID]`).

Todo corre en el equipo: ningún texto sale de la máquina. El modelo redacta;
la ficha del paciente, las preguntas, el orden xABCDE y la verificación de las
citas los pone el código. No indica conductas ni reemplaza el juicio clínico.

## Estructura

El sistema tiene dos partes que comparten el entorno virtual de la raíz. Su
único vínculo es el corpus: `rag_pipeline` lee lo que `anonimizacion` deja en
`anonimizacion/outputs/corpus_indexable.jsonl`.

```
chatbot_trauma/
├── README.md
├── requirements.txt              instala las dos partes (-r a cada carpeta)
├── .gitignore                    bloquea datos, índices y fuentes
│
├── anonimizacion/                genera el corpus anonimizado desde trauma.csv
│   ├── run_pipeline.py           corre las seis fases de principio a fin
│   ├── requirements.txt
│   ├── README.md                 decisiones de diseño de la anonimización
│   ├── DICCIONARIO_DATOS.md      cada columna del corpus
│   ├── config/
│   │   ├── abreviaturas.yaml     abreviaturas de urgencias (también las usa el chatbot)
│   │   └── gazetteers.yaml       ⟵ fuera de git: instituciones, médicos y municipios (04_restringido)
│   ├── src/
│   │   ├── normalize.py          fase 1: normalización estructural
│   │   ├── quality.py            fase 2: marcas de calidad (qué entra al índice)
│   │   ├── anonymize.py          fase 3: anonimización
│   │   ├── enrich.py             fase 4: mecanismo, regiones, desenlace y abreviaturas
│   │   ├── build_corpus.py       fase 5: ensamblado y exportación
│   │   ├── verify.py             fase 6: verificación del residuo
│   │   └── ner_review.py         revisión opcional con NER (spaCy)
│   ├── scripts/
│   │   └── build_gazetteers.py   borrador de los gazetteers desde el propio registro
│   ├── tests/
│   ├── data/                     ⟵ fuera de git: trauma.csv, los 3 PDF y transcripciones_guias.yaml
│   ├── outputs/                  ⟵ fuera de git: el corpus (01_corpus en Drive)
│   └── reports/                  ⟵ fuera de git: reportes de la anonimización
│
└── rag_pipeline/                 búsqueda, generación y la app
    ├── app.py                    el chatbot (Streamlit)
    ├── requirements.txt
    ├── README.md                 cómo funciona, configuración y decisiones de diseño
    ├── .streamlit/config.toml    evita que Streamlit recargue rag/ en caliente
    ├── rag/
    │   ├── config.py             toda la configuración, por variables de entorno
    │   ├── corpus.py             carga del corpus y tokenización para BM25
    │   ├── encoder.py            embeddings (bge-m3)
    │   ├── index.py              construye los índices: BM25 y Qdrant local
    │   ├── retrieve.py           búsqueda de casos y de manuales, reranker y guardas
    │   ├── senal.py              señal de trauma en la consulta
    │   ├── consulta.py           consulta para los manuales: abreviaturas y traducción
    │   ├── ficha.py              la ficha del paciente a lo largo de la conversación
    │   ├── preguntas.py          las preguntas con botones
    │   ├── prioridades.py        la letra xABCDE de cada término y cada pasaje
    │   ├── tarjetas.py           el protocolo por mecanismo, ajustado a la ficha
    │   ├── hallazgos.py          lo documentado en los casos, sin el modelo
    │   ├── build_prompt.py       los prompts y los textos fijos de la respuesta
    │   ├── generate.py           la respuesta completa, sobre Ollama
    │   ├── verificar.py          verificación de la prosa contra sus fuentes
    │   ├── guias.py              extracción del texto de los manuales
    │   └── fragmentar.py         división de los manuales en pasajes
    ├── tarjetas/
    │   ├── tarjetas.yaml         protocolos de 5 mecanismos, con sus citas
    │   └── patrones_registro.json  conteos del registro por mecanismo (sin texto de casos)
    ├── guias/
    │   ├── perfiles.yaml         cómo se extrae cada manual
    │   └── secciones.jsonl       ⟵ fuera de git: texto extraído (derechos de autor)
    ├── scripts/
    │   ├── revisar_entorno.py    comprueba VRAM, dependencias, índices y Ollama
    │   ├── extraer_guias.py      extrae el texto de los manuales
    │   ├── patrones_registro.py  recalcula los conteos del registro
    │   └── preparar_demo.py      prueba el sistema completo; termina en "Todo listo"
    ├── eval/
    │   ├── consultas.yaml        consultas de prueba que usa preparar_demo.py
    │   └── consultas_guias.yaml  ídem, para los manuales
    ├── tests/                    sin Ollama ni índices
    └── indices/                  ⟵ fuera de git: BM25 y Qdrant (06_indices en Drive)
```

**Este repositorio no tiene datos.** El corpus, los manuales y los índices
están en la carpeta `datos_procesados` del Drive institucional (ver
`SHA256SUMS.txt` y `LEEME.txt` en esa carpeta). El `.gitignore` bloquea
cualquier `.csv`, `.parquet`, índice o fuente: no forzarlo con `git add -f`.

---

## 1. Requisitos

- Windows 11, Python 3.12, GPU NVIDIA con 8 GB de VRAM (probado en RTX 4060).
- [Ollama](https://ollama.com).
- Presupuesto de VRAM: generador `qwen3:4b-instruct` ~3,6 GB con 6.144 tokens
  de contexto, reranker ~1,3 GB. Los embeddings de la consulta corren en CPU.

## 2. Instalar

Desde la carpeta que contiene este README:

```powershell
python -m venv .venv
.\.venv\Scripts\activate
pip install torch --index-url https://download.pytorch.org/whl/cu126
pip install -r requirements.txt
ollama pull qwen3:4b-instruct          # NO qwen3:4b (variante Thinking)
setx LLAMA_ARG_CACHE_RAM 0             # y reiniciar Ollama (ver §6)
python -c "from huggingface_hub import snapshot_download; snapshot_download('BAAI/bge-reranker-v2-m3')"
```

`torch` va primero y con el índice CUDA. Si `cu126` no tiene rueda para tu
Python, probar `cu128` y luego `cu124`; `nvidia-smi` dice qué CUDA soporta el
driver.

## 3. Poner los datos desde Drive

| Carpeta de Drive | Va en |
|---|---|
| `01_corpus/*` | `anonimizacion/outputs/` |
| `05_manuales/*` (3 PDF y `transcripciones_guias.yaml`) | `anonimizacion/data/` |
| `06_indices/indices.zip` (opcional) | descomprimir en `rag_pipeline/indices/` y borrar `qdrant/.lock` |
| `04_restringido/gazetteers.yaml` (solo para anonimizar) | `anonimizacion/config/` |

Para regenerar el corpus desde cero hacen falta `trauma.csv` en
`anonimizacion/data/` (custodia del HUV) y `gazetteers.yaml`, que tiene
nombres de médicos y por eso no va en git: `cd anonimizacion; python run_pipeline.py`.
El chatbot no lo necesita.

## 4. Construir los índices

Solo si no se usó `indices.zip`. Desde `rag_pipeline\`:

```powershell
python scripts\revisar_entorno.py
python -m rag.index                     # casos, ~1 min en GPU
python scripts\extraer_guias.py         # texto de los manuales
python -m rag.index --fuente guias      # manuales, ~30 s en GPU
```

Si se regenera el corpus, volver a correr `python scripts\patrones_registro.py`
(conteos que muestra la app).

## 5. Comprobar y abrir

Desde `rag_pipeline\`:

```powershell
python -m pytest tests/ -q              # sin Ollama ni índices
python scripts\preparar_demo.py         # debe terminar en "Todo listo"
streamlit run app.py                    # siempre desde rag_pipeline\
```

La app abre en `http://localhost:8501` y precarga los modelos (~45 s la
primera vez). Después, una respuesta tarda entre 5 y 10 s.

La configuración se cambia con variables de entorno; todas están
documentadas en `rag_pipeline/rag/config.py`.

## 6. Problemas conocidos

- **Qdrant local admite un solo proceso.** Con la app abierta fallan
  `rag.index` y `preparar_demo.py` (`already accessed by another instance`).
  Cerrar Streamlit, o apuntar `INDICE_DIR` a una copia de `indices\` sin el
  archivo `.lock`.
- **Lanzar Streamlit desde `rag_pipeline\`.** Solo así lee
  `.streamlit/config.toml`, que evita que recargue `rag/` y abra un segundo
  cliente de Qdrant. Al editar `rag/*.py`, reiniciar Streamlit.
- **HTTP 500 del chatbot**: casi siempre falta de memoria en Ollama.
  Probar `$env:NUM_GPU = "-1"`. Log en `%LOCALAPPDATA%\Ollama\server.log`.
- **La app se queda en "Cargando modelos…" y termina en `TimeoutError`**: la
  caché de prompts de Ollama llenó la RAM. Se evita con
  `setx LLAMA_ARG_CACHE_RAM 0` y reiniciando Ollama.
- **Ollama se usa por `127.0.0.1`, no `localhost`**: en Windows, `localhost`
  prueba primero IPv6 y cada llamada pierde 2 s.
- **Las tarjetas citan secciones por su identificador.** Si se vuelven a
  extraer los manuales con otra versión de `pymupdf` y cambia un identificador,
  `tests/test_tarjetas.py` lo avisa. La versión está fijada en
  `rag_pipeline/requirements.txt`.
