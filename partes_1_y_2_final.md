# Rediseño de RESPIA News con Jev: investigación y aplicación de los temas de clase

Caso de estudio hipotético: no se integra Jev ni se modifica el proyecto real. Esta versión reúne el contenido de las partes 1 y 2 de la tarea y alimenta el board y el prototipo.

**Cómo leer las marcas**

| Marca | Significa |
|---|---|
| **[V]** | Hecho verificado en una fuente oficial de TypeSafe que leímos (documentación, DPA, MCA, política de privacidad) |
| **[T]** | Medición o afirmación de un tercero; leímos su artículo o repositorio original |
| **[R]** | Dato de un tercero que solo leímos en resumen; no se verificó en el original |
| **[P]** | Afirmación del proveedor, sin verificación independiente |
| **[S]** | Supuesto nuestro |
| **[N/E]** | Las fuentes no lo especifican |

---

## Flujo elegido

**Publicación de una noticia desde el portal administrativo.**

Por qué: el enunciado del Proyecto 2 exige que el admin registre la procedencia, que se pueda determinar la relevancia de la noticia para distintos usuarios, que el sistema reduzca la publicación de noticias falsas y que comunique la incertidumbre. La tarea pide "un flujo relevante, como" selección, feed o consulta. Esos son ejemplos, así que este flujo es válido.

Se descartó el feed porque llamaría a un modelo cada vez que alguien lo recarga. En la publicación hay una llamada por noticia, una sola vez, y el resultado se guarda.

---

## Parte 1. Investigación

### 1.1 ¿Qué tareas puede resolver Jev y cuáles son sus límites frente a un LLM?

**Qué es.** Un modelo "System One" de TypeSafe: devuelve decisiones tipadas con probabilidades, no texto. [V]

**Qué hace.** Tres tipos de pregunta, combinables en una llamada [V]:
- `choice`: elige una opción (hasta 255) y devuelve la distribución de probabilidades y una confianza.
- `score`: puntúa en una rúbrica de 2 a 10 niveles.
- `noul`: probabilidad (0 a 1) de que una afirmación sea cierta.

**Cómo se usa.** Se envía un `state` (texto o JSON) y preguntas con `instructions` en lenguaje natural y `criteria`. No hay system prompt aparte. [V] El patrón que muestra la guía de Valyu es una llamada por decisión pequeña, con pocas preguntas cerradas y un `state` corto. [T]

**Límites.**

| Límite | Fuente |
|---|---|
| No genera texto: no resume, no explica, no redacta | [V] blog y documentación de TypeSafe |
| Solo texto, hasta 64k tokens por solicitud (32k para el estado más la pregunta más larga) | [V] página de modelos |
| El inglés es el idioma principal y donde la precisión es mejor; en otros idiomas hay menos precisión y recomiendan probar antes | [V] página de modelos |
| En español, la precisión baja de 3.0 a 6.4 puntos porcentuales en cuatro benchmarks públicos (XNLI 85.0% a 78.6%, PAWS-X 83.4% a 77.2%, MASSIVE 84.5% a 80.8%, Belebele 98.2% a 95.2%). El error de calibración casi se duplica en XNLI y PAWS-X. El texto en español usa entre 17% y 38% más tokens de entrada. Escribir las instrucciones en español en vez de inglés no cambió los resultados de forma apreciable. Los autores advierten: una sola corrida, una sola redacción, datos en es-ES, `score` no cubierto y "cuatro benchmarks académicos no son tu carga de trabajo" | [T] auditoría pre-registrada `jev-acento` (jev-1.13.0, 19,200 llamadas). No usa noticias |
| "Lee literal": responde la pregunta escrita, no la que se quiso hacer. Las preguntas con negaciones rinden peor. No es una calculadora. No da justificación auditable. No trata el `state` como hostil: si contiene texto controlado por usuarios, es riesgo del integrador | [T] guía de Valyu |
| Calibración desigual: en tickets de soporte (fuera de la distribución de entrenamiento), el error de calibración esperado fue 0.107, unas 4.4 veces el piso de 0.024. Las respuestas sí/no salieron subconfiadas y las de `choice` y `score` sobreconfiadas. En tareas que Jev no podía responder razonablemente, acertó 44.7% con una confianza media de 0.74 | [T] artículo de Rajesh Beri sobre `jev-phishing-bench` |
| No puede abstenerse: fuerza una decisión aunque la confianza sea baja | [R] Langfuse |
| "No es robusto ante contenido adversarial" | [R] Layer3Labs |
| La salida respeta el esquema, pero eso no implica que la decisión sea correcta. El blog admite que el "0% de alucinaciones" no es empírico | [S] lectura nuestra a partir de [V] |
| No verifica si algo es verdadero en el mundo: solo analiza el texto que recibe | [S] consecuencia de lo anterior |

**Lo que sí encaja:** clasificar, enrutar, puntuar, filtrar antes de un paso costoso y comprobar guardrails, con decisiones pequeñas e independientes. [V][T]

### 1.2 ¿Qué datos enviaría el proyecto y qué dicen las políticas?

**Qué se enviaría [S]:** el título y el texto de la noticia. Nada más. No saldrían la identidad ni el correo del admin, ni datos de lectores, ni el historial. La fuente, la región y la imagen se evalúan con código, dentro de nuestro sistema.

**Qué dicen las políticas** (documentos leídos: política de privacidad, DPA y Master Customer Agreement):

| Tema | Qué dice | Fuente |
|---|---|---|
| Entrenamiento | No entrenan ni afinan modelos con las entradas del cliente | [V] política de privacidad |
| Compartir | No divulgan las entradas a terceros salvo proveedores de servicio | [V] política de privacidad |
| Telemetría | Recogen datos de uso e interacción, cookies y analítica (incluye Google Analytics). El MCA menciona "technical logs" dentro de la telemetría | [V] política de privacidad y MCA |
| Retención | "El tiempo razonablemente necesario", sin plazo fijo. Retención cero (ZDR) solo para clientes enterprise | [V] política de privacidad, DPA y página legal |
| Conservación y borrado | TypeSafe "no tiene obligación de almacenar ni retener" los datos del cliente y puede borrarlos "en cualquier momento a su sola discreción" (MCA, secc. 10.3). No hay un proceso de borrado a petición descrito | [V] MCA |
| Ubicación | Servidores en EE. UU. | [V] política de privacidad |
| Transferencias | Cláusulas Contractuales Estándar de la UE; ley de Irlanda | [V] DPA |
| Incidentes | Aviso sin demora indebida y en un máximo de 72 horas | [V] DPA |
| Subprocesadores | Autorizados con aviso previo; hay una lista en su Trust Center | [V] DPA (la lista no se consultó) |
| Plazo y contenido de los logs de peticiones | No se especifican | [N/E] |
| Si el DPA cubre a quien usa créditos de bienvenida | El DPA forma parte del MCA (secc. 4.4), y el MCA contempla créditos comprados y promocionales. Suponemos que ambos aplican | [S] |

### 1.3 Evidencia de velocidad, costo y consumo de recursos

**Qué cuenta como "recursos".** La tarea pide "consumo de recursos" sin definirlo. Lo medimos con lo que observa quien integra el servicio: tokens consumidos (la respuesta trae `usage` con tokens de entrada y de salida [V]), límites de tasa y de contexto [V] y el cómputo que queda en nuestro servidor. No hay datos sobre el consumo físico (energía, agua) de los centros de datos, ni de TypeSafe ni de terceros, y no lo usamos para decidir.

**Hechos verificados y datos de terceros leídos en el original.**

| Dato | Valor | Fuente |
|---|---|---|
| Precio | $0.042 por millón de tokens de entrada; salida gratis | [V] página de modelos |
| Límites de tasa | 80 solicitudes/s y 100K tokens/s, con ajuste dinámico | [V] página de modelos |
| Phishing, 1,000 correos de prueba, costo por 1,000 correos | Jev $0.038; Claude Haiku 4.5 $0.462 (una pregunta) y $1.02 (cinco señales) | [T] `jev-phishing-bench` vía Beri. Costos calculados por el autor |
| Clasificar 1,018 papers en 24 temas | $0.08 con Jev frente a $3.99 por resumirlos con otro modelo; mediana de 256 ms por paper | [T] guía de Valyu. Es una carga distinta a la nuestra |
| Costo de tokens en español | Entre 17% y 38% más tokens que en inglés | [T] auditoría `jev-acento` |

**Afirmaciones del proveedor y datos no verificados.**

| Dato | Valor | Fuente |
|---|---|---|
| Latencia | 70 a 500 ms de punta a punta; LLM de 3 a 329 s | [P] blog de TypeSafe, medido desde laptops en la Costa Oeste |
| "40x a 200x más rápido y 193.6x más barato que modelos de frontera" | | [P] el propio autor admite sesgo (benchmarks hechos por el equipo del modelo; referencia basada en modelos de OpenAI y Anthropic) y que son "el extremo superior de las ganancias reales" |
| Acuerdo con un modelo de referencia | Jev coincidió con Claude Fable 5.1 en el 91.5% de 6,003 comprobaciones de rúbrica; costo de $160 frente a $33,000 por millón de respuestas | [R] Good Start Labs, vía Langfuse. Mide acuerdo, no exactitud |
| Exactitud agregada en el panel interno de TypeSafe | 67.8% frente a 74.1% de un comparador no identificado | [R] Layer3Labs; no es independiente |
| Costo para RESPIA | Con ~1,000 tokens por noticia, ~$0.00004 de entrada por noticia (hasta 38% más en español) | [S] cálculo nuestro con el precio [V] |

**Evidencia sobre exactitud (terceros, tareas distintas a las noticias).**

| Condición | Jev | Claude Haiku 4.5 | Fuente |
|---|---|---|---|
| Phishing, una sola pregunta | 62.6% | 81.3% (p < 0.0001) | [T] Beri |
| Phishing, cinco preguntas estrechas con regresión sobre 1,000 ejemplos etiquetados | 95.0% | 93.2% (p = 0.063, no significativo) | [T] Beri |

El autor aclara: "El 95% no es Jev. Es Jev más tus datos etiquetados más una regresión que mantienes tú". En ese benchmark los cuerpos de los correos son sintéticos y las etiquetas salen de listas de reputación de URL. [T]

**Implicaciones para nuestro diseño [S]:**
1. Las decisiones pequeñas y estrechas, como proponemos, rinden mejor que una pregunta amplia.
2. Los umbrales de confianza no se fijan a ojo: habría que calibrarlos con ejemplos etiquetados por humanos.
3. Por eso Jev solo sugiere y el admin decide.

---

## Parte 2. Aplicar los temas de clase al mismo flujo

### 2.1 Design Thinking

- **Persona principal:** la persona administradora que publica noticias. Secundaria: quien lee la noticia en la app.
- **Flujo actual [S]:** el enunciado no define el flujo. Suponemos que el admin escribe el texto y rellena a mano tema, región y procedencia, sin ninguna ayuda ni alerta antes de publicar.
- **Necesidad:** publicar con rapidez sin dejar pasar errores de clasificación, contenido sensible ni noticias sin fuente clara. Quien lee necesita noticias bien etiquetadas y que le digan cuándo algo es incierto.
- **Punto a mejorar:** el momento justo antes de publicar.
- **Qué cambia en la experiencia:** el admin ve etiquetas sugeridas con su confianza y alertas claras, y decide. Teclea menos y revisa mejor.
- **Cómo se mantiene el valor:** el admin conserva el control. La IA sugiere; no publica ni bloquea por sí sola.

### 2.2 Lean Thinking

**Llamadas a modelos que aportan valor y las que se evitan**
- Clasificación al publicar (Jev): **sí**. Una llamada por noticia, una sola vez.
- Resumen para el feed (LLM): **sí**. Es generación; Jev no puede hacerlo.
- Imagen cuando falta (modelo generativo de imágenes): **sí**, solo cuando la noticia no trae una. Jev no ve imágenes.
- **Se evitan:** clasificar en cada carga del feed, y usar un LLM para validar campos, comparar regiones o aplicar umbrales. Eso es código.

| Criterio | LLM generalista | Jev | Código |
|---|---|---|---|
| Velocidad | De 3 a 329 s [P] | De 70 a 500 ms [P] | Inmediata |
| Costo | $0.20 a $10 por millón de tokens de entrada, salida más cara [P] | $0.042 por millón de entrada, salida gratis [V] | Sin costo por llamada |
| Recursos (tokens, límites, cómputo propio) | Muchos tokens de salida; límites según el proveedor | Solo tokens de entrada, salida casi nula; 64k de contexto y 80 solicitudes/s [V] | Cómputo propio, mínimo |
| Funcionalidad | Genera, explica, razona | Clasifica y puntúa con confianza; no genera, no verifica, no explica | Reglas exactas |
| Qué se gana | Flexibilidad | Velocidad, costo y una confianza numérica | Previsibilidad y auditabilidad |
| Qué se pierde | Costo y latencia | Todo lo que sea texto libre; menor precisión en español [T] | No entiende lenguaje natural |

### 2.3 Responsible AI

**Datos mínimos que se compartirían.** Solo título y texto de la noticia. La fuente, la región y la imagen se procesan en nuestro sistema. Nada de datos de lectores ni del admin. Eso también limita la exposición si el proveedor conserva las peticiones, algo que sus políticas no acotan (sección 1.2).

**Cómo se comunican los límites y la incertidumbre**
- Las sugerencias se muestran como "sugerencia de IA", con su confianza, no solo con la etiqueta.
- Las alertas de contenido sensible y de lenguaje incierto dicen qué detectó el modelo, no si la noticia es verdadera.
- Si la confianza es baja, el sistema lo dice y pide revisar. Como Jev no puede abstenerse [R], esa baja confianza la detecta nuestro código con un umbral, no el modelo.
- Para el lector, el estado de confirmación sale de un campo que fija el admin, no de Jev.

**Qué ocurre ante una decisión incorrecta.** El admin puede cambiar cualquier etiqueta antes de publicar. Se guarda la sugerencia de la IA junto con la decisión final, para saber con qué frecuencia el humano corrige. Si una noticia ya publicada quedó mal etiquetada, el admin corrige la etiqueta y la relevancia se recalcula con la nueva.

**Riesgo de sesgo o exclusión.** Jev rinde mejor en inglés [V] y en español la precisión baja de 3 a 6 puntos y la calibración empeora [T]. Una noticia en español, local o de una región poco representada podría recibir una etiqueta peor o una confianza mal estimada, y eso puede ocultar justo la información local que el enunciado pide no ocultar.
**Medida:** si el idioma no es inglés o la confianza es baja, la revisión humana es obligatoria, y ninguna noticia se descarta ni se degrada automáticamente por la clasificación. Esa medida está en el diseño; no se pone a prueba porque el caso es hipotético.

**Otros riesgos**
- Un texto pegado desde una fuente externa podría traer instrucciones que desvíen la clasificación [T]. La decisión final es siempre de una persona.
- Jev no justifica sus decisiones [T]. Mostramos qué decidió y con qué confianza, pero no el porqué, y el board lo dice.
- Una etiqueta de "sin riesgo" no significa que la noticia sea verdadera: una noticia falsa bien escrita puede pasar. La verificación sigue siendo humana.

### 2.4 Matriz: LLM / JEV / CODE por función

Los temas del portal son Tech, Economía y Finanzas. Las celdas de velocidad y costo de JEV y LLM son las del proveedor [P] o cálculos nuestros [S] (secciones 1.3 y 2.2). Las preguntas de JEV se envían en una sola llamada por noticia, así que su costo y su velocidad se comparten.

| Función | Elección | Justificación | Impacto en UX | Privacidad | Velocidad | Costo | Recursos | Funcionalidad |
|---|---|---|---|---|---|---|---|---|
| Tema (Tech, Economía o Finanzas) | JEV | Elegir una categoría entre pocas; decisión pequeña | El admin parte de una sugerencia, no de una lista vacía | Sale título y texto | 70 a 500 ms [P] | ~$0.00004 por noticia [S] | Tokens de entrada | Etiqueta con confianza |
| Alcance (local, nacional, internacional) | JEV | Clasificación de una frase | Menos errores de región; el código lo compara con la región declarada | Sale título y texto | Misma llamada | Misma llamada | Misma llamada | Etiqueta con confianza |
| Tipo: hecho reportado u opinión | JEV | Clasificación simple [S] | Aviso antes de publicar una opinión como hecho | Sale título y texto | Misma llamada | Misma llamada | Misma llamada | Etiqueta con confianza |
| Uso de lenguaje incierto ("según fuentes no confirmadas") | JEV (`noul`) | Detecta señales en el texto; no sabe si algo está confirmado | Alerta al admin; no es un veredicto | Sale título y texto | Misma llamada | Misma llamada | Misma llamada | Probabilidad de 0 a 1 |
| Contenido sensible (violencia gráfica, autolesión) | JEV (`noul` por tipo) | Guardrail rápido sobre el texto | Alerta antes de publicar; la decisión es humana | Sale título y texto | Misma llamada | Misma llamada | Misma llamada | Probabilidad de 0 a 1 |
| Estado de confirmación (confirmada o en desarrollo) | CODE | Es un campo que fija el admin | El lector ve la marca de incertidumbre | No sale nada | Inmediata | Sin costo | Servidor propio | Regla exacta |
| Verificar si la noticia es verdadera | Decisión humana | Ningún modelo debe tratarse como prueba (enunciado) | El flujo exige que el admin confirme la fuente | No sale nada | Tiempo de una persona | Tiempo de una persona | Una persona | Juicio y contraste de fuentes |
| Fuente presente, fuente nueva, campos vacíos | CODE | Reglas exactas | Impide publicar sin procedencia | No sale nada | Inmediata | Sin costo | Servidor propio | Regla exacta |
| Umbrales de confianza y ruta a revisión | CODE | Regla simple y auditable | El admin ve el motivo del aviso | No sale nada | Inmediata | Sin costo | Servidor propio | Regla exacta |
| Resumen para el feed | LLM | Es generación de texto | Texto breve y legible | Sale el texto al proveedor del LLM (por definir) [S] | De 3 a 329 s [P] | Tokens de entrada y de salida [P] | Muchos tokens de salida | Genera texto |
| Imagen cuando falta | LLM (modelo generativo de imágenes, distinto de Jev) | Jev no ve ni genera imágenes | Imagen con la etiqueta "generada por IA", que pone el código | Sale un prompt al proveedor de imágenes (por definir) [S] | Segundos [S] | Por imagen (por definir) [S] | Cómputo del proveedor | Genera imágenes |
| Chat de noticias con fuentes (fuera del flujo elegido) | LLM | Requiere generar explicaciones | Sin cambios | Sale la consulta y el contexto al proveedor del LLM [S] | De 3 a 329 s [P] | Tokens de entrada y de salida [P] | Muchos tokens de salida | Genera texto con fuentes |

### 2.5 Flujo propuesto

1. El admin escribe o pega la noticia y completa fuente y región.
2. El sistema valida con código: campos obligatorios, fuente, región y longitud.
3. El sistema envía a Jev solo el título y el texto, con preguntas cortas e independientes: tema, alcance, tipo, lenguaje incierto y contenido sensible. Si falta imagen, aparte, se pide una a un modelo generativo.
4. El código aplica los umbrales de confianza y decide si hay que pedir revisión.
5. El admin ve las sugerencias con su confianza y las alertas, corrige lo que haga falta y publica.
6. Se guardan las etiquetas finales y la sugerencia de la IA.

**Estado de incertidumbre en el prototipo:** confianza baja, idioma distinto del inglés o alerta de contenido sensible. La pantalla lo explica en una frase y ofrece una salida clara: revisar y corregir, enviar a revisión o publicar sabiendo el aviso.

---

## Conclusión sobre la arquitectura propuesta

Un sistema híbrido: **Jev** resuelve las decisiones cortas y cerradas de clasificación y alerta al publicar, **código** aplica reglas exactas, umbrales y validaciones, y un **LLM** genera lo que requiere texto libre (resúmenes, chat) junto a un modelo de imágenes. La verificación de la verdad y la decisión final de publicar siguen en una persona.

Lo que se gana, según las fuentes: llamadas más rápidas y baratas para las decisiones pequeñas [P][T] y datos mínimos enviados al proveedor. Lo que se pierde o queda en riesgo: justificaciones, soporte parejo para el español [T] y una confianza que hay que calibrar [T].

## Supuestos y lo que faltaría verificar

Es un caso hipotético: no se verifican, se declaran.
- Que Jev clasifique bien noticias reales; las pruebas de terceros son de otras tareas.
- Que los umbrales de confianza se calibren con ejemplos etiquetados por humanos.
- Que las noticias se clasifiquen en inglés o se traduzcan antes, y cuánto cuesta traducir.
- Que el DPA cubra a quien use créditos de bienvenida.
- Plazo y contenido de los logs de peticiones; lista de subprocesadores: el proveedor no los especifica en los documentos leídos.
- Los datos marcados [P] y [R] provienen del proveedor o de resúmenes de terceros y no se verificaron en el original.

## Fuentes

Del proveedor (leídas):
- Documentación de Jev: https://docs.typesafe.ai/introduction
- Modelos: https://docs.typesafe.ai/models.md
- API: https://docs.typesafe.ai/api.md
- Blog de presentación: https://typesafe.ai/blog/introducing-system-one-models-and-jev
- Data Processing Agreement: https://typesafe.ai/legal/data-processing
- Master Customer Agreement: https://typesafe.ai/legal/mca
- Política de privacidad: https://typesafe.ai/privacy

De terceros, leídas en el original:
- Guía de Valyu: https://dev.to/valyuai/how-to-use-jev-a-practical-guide-to-typesafes-system-one-model-g5e
- Auditoría en español `jev-acento`: https://github.com/marcosmartinez/jev-acento
- Artículo de Rajesh Beri sobre `jev-phishing-bench`: https://www.beri.net/article/typesafe-jev-typed-decision-model-calibration-decomposition-shadow-eval

De terceros, solo en resumen:
- Layer3Labs: https://www.layer3labs.io/guides/jev-benchmarks
- Langfuse: https://langfuse.com/blog/2026-09-18-using-typesafes-jev-for-evals.md
- DataCamp: https://www.datacamp.com/blog/system-one-models-jev
