# Chatbot — búsqueda, generación y app

Es la segunda parte del sistema. Responde consultas sobre pacientes de trauma
desde dos fuentes, con cita en cada afirmación:

- **Casos del registro**: los 8.198 casos que deja
  `../anonimizacion/outputs/corpus_indexable.jsonl` (`[caso: ID]`).
- **Manuales**: ATLS 11.ª ed. (inglés), ATLS 10.ª ed. (traducción no oficial
  al español) y el manual del HUV (`[guía: ID]`).

Todo corre en local: los embeddings y el reranker con `torch`, el generador
(`qwen3:4b-instruct`) con Ollama y los índices en un Qdrant embebido, sin
servicio. La instalación y la puesta en marcha están en el `README.md` de la
raíz; aquí se explica cómo funciona.

## Uso

Siempre desde `rag_pipeline\`:

```powershell
python scripts\revisar_entorno.py       # VRAM, dependencias, índices y Ollama
python -m rag.index                     # índices de los casos
python scripts\extraer_guias.py         # texto de los manuales (desde los PDF)
python -m rag.index --fuente guias      # índice de los manuales
python -m pytest tests/ -q              # sin Ollama ni índices
python scripts\preparar_demo.py         # el sistema completo: debe terminar en "Todo listo"
streamlit run app.py
```

`rag.index` acepta `--fuente casos|guias|todas`; reindexar una fuente no borra
la otra. Si se regenera el corpus, correr también
`python scripts\patrones_registro.py`.

## Recorrido de una consulta

1. **Ficha** (`rag/ficha.py`). Reglas, sin el modelo, deciden si el mensaje
   presenta a un paciente nuevo o sigue con el mismo, y acumulan mecanismo,
   regiones, hallazgos afirmados y negados, signos vitales, grupo de edad y
   estado hemodinámico. Un paciente nuevo solo empieza si el mensaje cuenta un
   trauma.
2. **Casos** (`rag/retrieve.py`). BM25 y búsqueda densa (bge-m3) fusionadas con
   RRF; el reranker (bge-reranker-v2-m3) reordena los 30 primeros y quedan 5.
   Dos guardas deciden si se responde: el coseno del mejor caso
   (`UMBRAL_ABSTENCION`) y al menos `MIN_CASOS` casos sobre `UMBRAL_RERANK`.
3. **Manuales** (`rag/consulta.py`, `rag/retrieve.py`). La consulta se busca
   con las abreviaturas expandidas y el nombre del mecanismo; para el ATLS 11
   se traduce al inglés con el mismo generador. Guarda por el mejor fragmento
   (`UMBRAL_GUIAS`) y cuota igual por libro dentro de `PRESUPUESTO_GUIAS`
   tokens.
4. **Señal de trauma** (`rag/senal.py`). Si la consulta no nombra un
   mecanismo, una lesión ni un procedimiento de trauma, las dos guardas piden
   más. Así el sistema se abstiene ante consultas ajenas al trauma.
5. **Protocolo** (`rag/tarjetas.py`). Para peatón, ocupante, caída, arma de
   fuego y cortopunzante, el apartado de los manuales es una tarjeta escrita
   de antemano desde los pasajes, ajustada a la ficha: lo que los datos del
   paciente vuelven prioritario sube a "Con estos datos". Encima de los casos
   va la línea del registro ("En el registro hay 324 casos de peatón…").
6. **Generación** (`rag/build_prompt.py`, `rag/generate.py`). El modelo
   escribe en JSON con esquema, por letras de la revisión primaria (xABCDE).
   La letra de cada término y de cada pasaje la decide `rag/prioridades.py`.
7. **Verificación** (`rag/verificar.py`). Cada oración se revisa contra lo que
   cita: que la cita exista, que las cifras y los términos clínicos estén en la
   fuente y que oriente en vez de ordenar.
8. **Preguntas** (`rag/preguntas.py`). Hasta dos preguntas con botones sobre lo
   que falta y cambiaría el manejo. Un clic suma el dato a la ficha y responde
   de nuevo.

Escribir solo un número (`40887`) recupera y resume ese caso.

## Estructura

```
rag_pipeline/
├── app.py                    el chatbot (Streamlit)
├── requirements.txt
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
│   └── preparar_demo.py      prueba el sistema completo
├── eval/
│   ├── consultas.yaml        consultas de prueba que usa preparar_demo.py
│   └── consultas_guias.yaml  ídem, para los manuales
├── tests/                    sin Ollama ni índices
└── indices/                  ⟵ fuera de git: BM25 y Qdrant (06_indices en Drive)
```

## Configuración

Todo se cambia con variables de entorno (`$env:NOMBRE = "valor"` antes de
lanzar la app). Cada una está documentada, con sus mediciones, en
`rag/config.py`. Las principales:

| Variable | Por defecto | Qué hace |
|---|---|---|
| `MODELO_LLM` | `qwen3:4b-instruct` | generador en Ollama. No `qwen3:4b`, que es la variante que razona |
| `OLLAMA_URL` | `http://127.0.0.1:11434` | con `localhost`, Windows pierde 2 s por llamada |
| `NUM_CTX` | `6144` (`3072` sin manuales) | contexto del generador |
| `NUM_GPU` | `99` | capas en GPU; `-1` deja decidir a Ollama |
| `KEEP_ALIVE` | `2h` | el modelo no se descarga entre preguntas |
| `RUTA` | `auto` | `rerank` con ≥ 7 GB de VRAM; si no, `rrf` |
| `DISPOSITIVO_RERANKER` | `auto` | `cuda` con ≥ 7 GB; en CPU tarda ~31 s por consulta |
| `TOP_K` | `5` | casos que van al prompt |
| `UMBRAL_ABSTENCION` | `0.52` | coseno mínimo del mejor caso |
| `UMBRAL_RERANK` | `0.2` | probabilidad mínima por caso |
| `GUIAS` | `1` | los manuales como segunda fuente |
| `ATLS10` | `1` | el ATLS 10 como tercer manual; `0` vuelve a dos |
| `TRADUCIR_ATLS11` | `1` | el ATLS 11 se busca con la consulta en inglés |
| `UMBRAL_GUIAS` | `0.34` | probabilidad mínima del mejor fragmento |
| `PRESUPUESTO_GUIAS` | `1800` | tokens de los manuales en el prompt |
| `SENAL_TRAUMA` | `1` | guardas más exigentes sin señal de trauma |
| `CONVERSACION` | `ficha` | `llm` = el modelo reescribe cada seguimiento |
| `PREGUNTAS` | `1` | preguntas con botones |
| `ORDEN_ATLS` | `1` | respuesta por letras xABCDE |
| `TARJETAS` | `1` | protocolo por mecanismo |
| `PROSA_SIEMPRE` | `1` | muestra la prosa aunque no pase la verificación (ver abajo) |
| `INDICE_DIR` | `indices/` | apuntar a una copia para correr scripts con la app abierta |
| `CORPUS_JSONL` | `../anonimizacion/outputs/corpus_indexable.jsonl` | el corpus que se indexa |
| `LIBROS_DIR` | `../anonimizacion/data` | los PDF de los manuales y sus transcripciones |

## Decisiones de diseño

**El modelo redacta, el código verifica.** La ficha, las preguntas, la letra
xABCDE, el protocolo y la línea del registro los pone el código. El modelo
solo redacta la prosa, y `rag/verificar.py` la revisa oración por oración: que
cada cita exista, que las cifras y los términos clínicos estén en la fuente
citada y que oriente ("se describe", "se documentó") en vez de ordenar. El
vocabulario de redacción permitido incluye lo escrito en `build_prompt.py`: lo
que se agregue ahí, aunque sea un comentario, deja de contar como inventado.

**`PROSA_SIEMPRE=1`: la prosa se muestra aunque no pase.** Se eligió para la
demostración, para que todas las respuestas se lean de corrido. La
verificación corre igual y sus motivos quedan en `Respuesta.motivos`. Con
`PROSA_SIEMPRE=0`, lo que no pasa se muestra armado desde la fuente y una nota
arriba de la respuesta dice por qué. Es la configuración coherente con "nada
inventado" y conviene volver a ella fuera de la demostración.

**Abstenerse antes que responder con casos que no corresponden.** Dos guardas
para los casos y una para los manuales; se responde si pasa cualquiera de las
fuentes. Sin señal de trauma, las guardas piden más: con las de hoy, ninguna de
las 20 consultas ajenas al trauma de `eval/` recibe respuesta y las 25 de
trauma sí.

**Casos y manuales no se mezclan.** Van en apartados separados, con el mismo
peso, y el sistema no elige entre ellos: si no coinciden, lo dice el código,
no el modelo.

**El ATLS 11 se busca en inglés.** El reranker puntúa bajo un pasaje en inglés
ante una consulta en español, así que el ATLS 11 casi nunca llegaba al prompt.
El generador traduce la consulta (una línea, con descarte si no parece
traducción) y el ATLS 11 se busca y reordena con ella; los manuales en español,
con la original.

**Las tarjetas citan secciones por su identificador.** Cada viñeta tiene que
pasar el verificador contra lo que cita de cada libro por separado
(`tests/test_tarjetas.py`). Si se vuelven a extraer los manuales y cambia un
identificador o su texto, ese test lo avisa. Por eso `pymupdf` está fijado en
`requirements.txt`. Las viñetas se muestran antes de que las revise el médico;
el encabezado de `tarjetas.yaml` señala las tres en que el ATLS 11 dice otra
cosa.

## Pruebas

`python -m pytest tests/ -q` corre sin Ollama ni índices: el Qdrant y el
generador se reemplazan por dobles. Tres tests se saltan si no están el texto
extraído de los manuales o las transcripciones, que no van en git.

`scripts/preparar_demo.py` es la prueba del sistema completo, con los modelos
y los índices reales: GPU, Ollama, las consultas de `eval/` marcadas como
botones, una conversación de tres turnos y el peatón con su tarjeta. Debe
correrse con la app cerrada, porque Qdrant local admite un solo proceso.
