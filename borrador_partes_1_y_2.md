# Rediseño de RESPIA News con Jev: borrador de las partes 1 y 2

Borrador de trabajo. Caso de estudio hipotético: no se integra Jev ni se cambia el proyecto real.
Leyenda: **[V]** hecho verificado en la fuente oficial · **[P]** afirmación del proveedor · **[S]** supuesto nuestro · **[?]** falta verificar.

## Flujo elegido

**Publicación de una noticia desde el portal administrativo.** El enunciado del Proyecto 2 pide que el admin registre la procedencia de la noticia, que se pueda determinar su relevancia para distintos usuarios y que el sistema reduzca la publicación de noticias falsas y comunique la incertidumbre. La tarea habla de "un flujo relevante, como ..." (selección, feed o consulta), así que este flujo es una elección válida.

Se descartó el feed porque obligaría a llamar a un modelo cada vez que una persona lo recarga. En la publicación hay una sola llamada por noticia, una vez, y el resultado se guarda.

---

## Parte 1. Investigación

### 1.1 ¿Qué tareas puede resolver Jev y cuáles son sus límites frente a un LLM?

**Qué es.** Un modelo "System One" de TypeSafe que devuelve decisiones tipadas con probabilidades en lugar de texto. [V: documentación]

**Qué hace.** Tres tipos de pregunta, que se pueden combinar en una llamada [V]:
- `choice`: elige una opción de hasta 255 y devuelve la distribución de probabilidades y una confianza.
- `score`: puntúa en una rúbrica de 2 a 10 niveles.
- `noul`: probabilidad (0 a 1) de que una afirmación sea cierta.

**Cómo se usa.** Se le pasa un `state` (texto o JSON) y preguntas con `instructions` en lenguaje natural y `criteria`. No hay system prompt aparte. [V]

**Límites.**

| Límite | Fuente |
|---|---|
| No genera texto (no resume, no explica, no redacta) | [V] blog de TypeSafe y guía de Valyu |
| Solo entrada de texto, 64k tokens por solicitud | [V] página de modelos |
| El inglés es el idioma principal y donde mejor funciona; en otros idiomas hay menos precisión y piden probar antes | [V] página de modelos |
| En español la precisión baja entre 3 y 6 puntos porcentuales en 4 benchmarks públicos (por ejemplo, 85.0% a 78.6% en XNLI); la calibración empeora en tareas difíciles; el texto en español gasta entre 17% y 38% más tokens que en inglés. Los propios autores avisan de que no generaliza a casos reales y de que no cubrieron tareas `score` | [V] auditoría pre-registrada de un tercero (jev-acento, GitHub), no de TypeSafe. No usa noticias |
| No puede abstenerse: fuerza una decisión aunque la confianza sea baja | [V] medición de Langfuse / Good Start Labs (tercero) |
| Calibración desigual: en una prueba independiente, las respuestas `noul` salieron subconfiadas y las `choice` y `score` sobreconfiadas, así que `confidence` no se puede tomar como probabilidad real sin calibrar | [V] prueba de un tercero (jev-phishing-bench, resumida por Beri). Es una tarea distinta a la nuestra |
| "No es robusto ante contenido adversarial" | [V] resumen de Layer3Labs (tercero) |
| "Lee literal": responde la pregunta escrita, no la que se quiso hacer. Las preguntas con negaciones rinden peor | [V] guía de Valyu (tercero) |
| No da justificación auditable ni razonamiento complejo, ni aritmética | [V] guía de Valyu (tercero) |
| No trata el `state` como hostil: si contiene texto controlado por usuarios, es riesgo del integrador | [V] guía de Valyu (tercero) |
| La salida respeta el esquema, pero eso no significa que la decisión sea correcta | [S] lectura nuestra; el blog dice que "0% de alucinaciones" no es empírico |
| No verifica si algo es verdadero en el mundo: solo analiza el texto recibido | [S] consecuencia de lo anterior |

Lo que **sí** encaja: clasificar, enrutar, puntuar, filtrar antes de un paso costoso y comprobar guardrails, con decisiones pequeñas e independientes. [V]

### 1.2 ¿Qué datos enviaría el proyecto y qué dicen las políticas?

**Qué se enviaría [S]:** solo título y texto de la noticia (o un extracto) y, si hace falta, la región declarada. No se enviaría identidad del admin, correo, perfil de lectores ni historial. Las noticias son contenido ya destinado a publicarse, pero el borrador aún no lo está.

**Qué dicen las políticas.**

| Tema | Qué dice | Fuente |
|---|---|---|
| Entrenamiento | No entrenan ni afinan modelos con las entradas del cliente | [V] política de privacidad |
| Compartir | No divulgan las entradas a terceros salvo proveedores de servicio | [V] política de privacidad |
| Telemetría | Recogen datos de uso e interacción, cookies y analítica (incluye Google Analytics) | [V] política de privacidad |
| Retención | "El tiempo razonablemente necesario"; sin plazo fijo. Retención cero solo para clientes enterprise | [V] política y DPA |
| Borrado | El Master Customer Agreement (secc. 10.3) dice que TypeSafe "no tiene obligación de almacenar ni retener" los datos del cliente y "puede borrarlos en cualquier momento a su sola discreción". No hay un proceso de borrado a petición descrito | [V] MCA |
| Ubicación | Servidores en EE. UU. | [V] política de privacidad |
| Transferencias | Cláusulas Contractuales Estándar de la UE; ley de Irlanda | [V] DPA |
| Incidentes | Aviso en un máximo de 72 horas | [V] DPA |
| Logs de peticiones | El MCA solo menciona "technical logs" dentro de la telemetría, sin plazo ni detalle. El DPA no habla de logs | [V] parcial; el detalle sigue [?] |
| Subprocesadores | El DPA los autoriza con aviso previo y remite a una lista en su Trust Center, que no se consultó | No requerido por la tarea; se cita solo que existe la lista |
| Si el DPA aplica a cuentas gratuitas o de prueba | El DPA forma parte del MCA (secc. 4.4). El MCA solo habla de créditos comprados y promocionales y no menciona cuentas gratuitas. Es razonable suponer que quien use créditos de bienvenida acepta el MCA y por tanto el DPA | [S] sin confirmar |

### 1.3 Evidencia de velocidad, costo y recursos

| Dato | Valor | Estado |
|---|---|---|
| Precio de Jev | $0.042 por millón de tokens de entrada; salida gratis | [V] página de modelos |
| Límite de tasa | 80 solicitudes/s y 100K tokens/s, con ajuste dinámico | [V] página de modelos |
| Latencia | 70 a 500 ms de punta a punta | [P] blog de TypeSafe; medida por ellos desde la Costa Oeste |
| "40x a 200x más rápido y 193.6x más barato que modelos de frontera" | | [P] los propios autores admiten sesgo (benchmarks hechos por el equipo del modelo; referencia basada en modelos de OpenAI y Anthropic) y que son el extremo alto |
| Clasificar 1,018 papers: $0.08 con Jev frente a $3.99 por resumirlos con un LLM; mediana de 256 ms | | [P] ejemplo de un tercero (Valyu), con una carga distinta a la nuestra |
| Phishing, 1,000 correos de prueba: $0.038 con Jev frente a $0.462 con Claude Haiku (una pregunta), por 1,000 correos | | [V] medición de un tercero (jev-phishing-bench); costo calculado por ellos, sobre otra tarea |
| Acuerdo con un modelo de referencia: Jev coincidió con Claude Fable 5.1 en el 91.5% de 6,003 comprobaciones de rúbrica, a $160 por millón de respuestas frente a $33,000 de Fable 5.1 | | [V] medición de Good Start Labs, citada por Langfuse (tercero); mide acuerdo, no exactitud |
| Costo para RESPIA | Con ~1,000 tokens por noticia, el costo de entrada sería ~$0.00004 por noticia. Si el texto está en español, hasta 38% más de tokens | [S] cálculo nuestro a partir del precio [V], un tamaño supuesto y el dato de tokens de la auditoría en español |

**Qué cuenta como "recursos".** El PDF de la tarea pide "consumo de recursos" y "recursos" en la matriz, sin definirlos. Los medimos con lo que sí es observable para quien integra el servicio: tokens consumidos (la respuesta trae `usage` con tokens de entrada y salida [V]), límites de tasa y de contexto [V], y el cómputo que se queda en nuestro servidor. Sobre el consumo físico de los centros de datos (energía o agua) no hay datos de TypeSafe ni de terceros; lo declaramos como no disponible y no lo usamos para decidir.

**Evidencia independiente sobre exactitud (terceros, tareas distintas a las noticias).**

| Dato | Resultado | Fuente |
|---|---|---|
| Phishing, una sola pregunta | Jev 62.6% frente a 81.3% de Claude Haiku | [V] jev-phishing-bench, vía Beri |
| Phishing, cinco preguntas estrechas con regresión sobre 1,000 ejemplos etiquetados | Jev 95.0% frente a 93.2% de Haiku (diferencia no significativa) | [V] misma fuente. El autor advierte: "el 95% no es Jev, es Jev más tus datos etiquetados más una regresión" |
| Calibración fuera de distribución | Error de calibración esperado de 0.107, unas 4.4 veces el piso de ruido | [V] misma fuente |
| Exactitud agregada en el panel interno de TypeSafe | 67.8%, frente a 74.1% de un comparador no identificado | [P] según el resumen de Layer3Labs; no es independiente |

**Implicaciones para nuestro diseño** [S]: (1) las decisiones pequeñas y estrechas, como proponemos, rinden mejor que una pregunta amplia; (2) los umbrales de confianza no se pueden fijar a ojo: hay que calibrarlos con ejemplos etiquetados por humanos antes de confiar en ellos; (3) por eso la salida de Jev solo sugiere y el admin decide.

---

## Parte 2. Aplicar los temas de clase al mismo flujo

### 2.1 Design Thinking

- **Persona principal:** la persona administradora que publica noticias en el portal. Secundaria: quien lee la noticia en la app.
- **Flujo actual [S]:** el enunciado no define el flujo. Suponemos que el admin escribe el texto y rellena a mano tema, región y procedencia, y no recibe ninguna ayuda ni alerta antes de publicar.
- **Necesidad:** publicar rápido sin dejar pasar errores de clasificación, contenido sensible ni noticias sin fuente clara. Quien lee necesita que la noticia esté bien etiquetada y que le digan cuándo algo es incierto.
- **Punto a mejorar:** el momento justo antes de publicar.
- **Qué cambia en la experiencia:** el admin ve etiquetas sugeridas con su nivel de confianza y alertas claras, y decide. Escribe menos y se equivoca menos.
- **Cómo se mantiene el valor:** el admin conserva el control. La IA sugiere, no publica ni bloquea sola.

### 2.2 Lean Thinking

Llamadas a modelos que aportan valor:
- Clasificación al publicar (Jev): **sí**. Una llamada por noticia, una vez. Más barata y rápida que usar un LLM para lo mismo [P].
- Resumen para el feed (LLM): **sí**. Es generación y Jev no puede hacerlo.
- Imagen (modelo generativo de imágenes): **sí**, solo cuando la noticia no trae imagen. Jev no ve imágenes.
- Llamadas que se pueden **evitar**: no clasificar en cada carga del feed, y no usar un LLM para validar campos, comparar regiones ni aplicar umbrales (eso es código).

| Criterio | LLM generalista | Jev | Código |
|---|---|---|---|
| Velocidad | Segundos [P] | 70 a 500 ms [P] | Inmediata |
| Costo | $0.20 a $10 por millón de tokens de entrada, y la salida cuesta más [P] | $0.042 por millón de entrada, salida gratis [V] | Sin costo por llamada |
| Recursos (tokens, límites, cómputo propio) | Muchos tokens de salida; límites de contexto y de tasa según el proveedor | Tokens de entrada solo; salida casi nula; 64k de contexto y 80 solicitudes/s [V] | Cómputo propio del servidor, mínimo |
| Funcionalidad | Genera, explica, razona | Clasifica y puntúa con confianza; no genera, no verifica, no explica | Reglas exactas |
| Lo que se gana | Flexibilidad | Velocidad, costo y una confianza numérica | Previsibilidad y auditabilidad |
| Lo que se pierde | Costo y latencia | Todo lo que sea texto libre; idiomas distintos al inglés [S] | No entiende lenguaje natural |

### 2.3 Responsible AI

**Datos mínimos.** Solo título y texto de la noticia, y la región declarada si hace falta. Nada de datos de lectores. Esto también acota la exposición si el servicio guarda las peticiones [?].

**Cómo se comunican límites e incertidumbre.**
- Las sugerencias se muestran como "sugerencia de IA", con la confianza y no solo la etiqueta.
- La alerta de contenido sensible y la de lenguaje incierto indican qué detectó el modelo, no si la noticia es verdadera.
- Si la confianza es baja, el sistema lo dice y pide revisar. Para el lector, una noticia con lenguaje incierto aparece marcada como "en desarrollo o sin confirmar" según el campo que fija el admin.

**Qué pasa ante una decisión incorrecta.** El admin puede cambiar cualquier etiqueta antes de publicar. Se guarda la sugerencia de la IA junto con la decisión final, para medir con qué frecuencia el humano corrige. Si una noticia ya publicada se etiquetó mal, el admin puede corregirla y la regla de relevancia se recalcula con la etiqueta nueva.

**Riesgo de sesgo o exclusión.** Jev rinde mejor en inglés [V]. Una noticia en español, local o de una región poco representada podría recibir menos confianza o una etiqueta peor, y eso puede ocultar justamente la información local que el enunciado pide no ocultar. **Medida:** si la confianza es baja o el idioma no es inglés, la revisión humana es obligatoria y la noticia nunca se descarta automáticamente por la clasificación. Una auditoría externa en español muestra una caída de precisión de 3 a 6 puntos y peor calibración en tareas difíciles (sección 1.1), pero no usa noticias ni cubre `score`. **Pendiente de verificar:** medir el acuerdo entre Jev y las etiquetas humanas, por idioma y región, con una muestra de noticias reales.

**Otros riesgos.**
- Un texto pegado desde una fuente externa podría contener instrucciones que desvíen la clasificación. Como Jev no trata el `state` como hostil [V], la decisión final queda siempre en una persona.
- Jev no justifica sus decisiones [V]. Mostramos qué decidió y con qué confianza, pero no por qué. Debe quedar dicho en el board.

### 2.4 Matriz: LLM / JEV / CODE por función

| Función | Elección | Justificación breve | Impacto en la experiencia |
|---|---|---|---|
| Tema de la noticia | JEV | Elegir una categoría entre pocas; decisión pequeña | El admin parte de una sugerencia en lugar de una lista vacía |
| Alcance (local, nacional, internacional) | JEV | Clasificación de una frase | Menos errores de región; el código lo compara con la región declarada |
| Tipo: hecho reportado u opinión | JEV | Clasificación simple [S] | Aviso antes de publicar una opinión como hecho |
| Uso de lenguaje incierto ("según fuentes no confirmadas") | JEV (`noul`) | Detecta señales en el texto. No sabe si algo está confirmado | Alerta al admin; no es un veredicto |
| Contenido sensible (violencia gráfica, autolesión) | JEV (`noul` por tipo) | Guardrail rápido sobre el texto | Alerta antes de publicar; la decisión es humana |
| Estado de confirmación (confirmada o en desarrollo) | CODE | Es un campo que fija el admin | El lector ve la marca de incertidumbre |
| Verificar si la noticia es verdadera | Decisión humana | Ningún modelo debe tratarse como prueba (enunciado) | El flujo exige que el admin confirme la fuente |
| Fuente presente, fuente nueva, campos vacíos | CODE | Reglas exactas | Bloquea publicar sin procedencia |
| Umbrales de confianza y ruta a revisión | CODE | Regla simple y auditable | El admin ve el motivo del aviso |
| Resumen para el feed | LLM | Es generación de texto | Texto breve y legible |
| Imagen cuando falta | LLM (modelo generativo de imágenes, distinto de Jev) | Jev no ve ni genera imágenes | Imagen con la etiqueta "generada por IA", puesta por el código |
| Chat de noticias con fuentes (fuera del flujo elegido) | LLM | Requiere generar explicaciones | Sin cambios |

---

## Pendientes de verificación

Son supuestos del caso hipotético: no se implementa nada, así que no se verifican, solo se declaran.
- Que Jev clasifique bien noticias reales (las pruebas de terceros son de otras tareas).
- Que los umbrales de confianza se calibren con ejemplos etiquetados por humanos.
- Que el DPA cubra a quien use créditos de bienvenida.
- Que las noticias se clasifiquen en inglés o se traduzcan antes.

Vacíos en la información pública del proveedor (se citan como "no especificado", no se investigan más):
- Plazo y contenido de los logs de peticiones: las políticas solo mencionan "technical logs".
- Consumo físico de los centros de datos (energía, agua): no hay datos.

## Fuentes

- Documentación de Jev: https://docs.typesafe.ai/introduction
- Modelos: https://docs.typesafe.ai/models.md
- API: https://docs.typesafe.ai/api.md
- Blog de presentación: https://typesafe.ai/blog/introducing-system-one-models-and-jev
- Data Processing Agreement: https://typesafe.ai/legal/data-processing
- Política de privacidad: https://typesafe.ai/privacy
- Guía de Valyu (tercero): https://dev.to/valyuai/how-to-use-jev-a-practical-guide-to-typesafes-system-one-model-g5e
- DataCamp (tercero): https://www.datacamp.com/blog/system-one-models-jev
- Master Customer Agreement: https://typesafe.ai/legal/mca
- Auditoría en español (tercero): https://github.com/marcosmartinez/jev-acento
- jev-phishing-bench, vía Beri (tercero): https://www.beri.net/article/typesafe-jev-typed-decision-model-calibration-decomposition-shadow-eval
- Layer3Labs, benchmarks (tercero): https://www.layer3labs.io/guides/jev-benchmarks
- Langfuse, Jev para evaluaciones (tercero): https://langfuse.com/blog/2026-09-18-using-typesafes-jev-for-evals.md
