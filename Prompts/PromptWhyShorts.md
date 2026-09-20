# Rutina — Canal "Why" (una pregunta, 6 actos, shorts diarios)

Produce **un short vertical en inglés** (~70-85 segundos) que responde **una
pregunta popular de psicología o sociología** con fuentes académicas reales y un
gráfico de datos, en 6 actos. Al terminar, **súbelo a YouTube** y repórtame la URL.

Formato hermano del deep dive, pero de nicho evergreen: no depende de noticias,
el archivo acumula views por años, y el gancho no es novedad sino
**contraintuición** — la respuesta popular es casi siempre la equivocada.

---

## 0. Cómo usar este documento

Este prompt está escrito como **harness de ejecución**: casi todas las decisiones
visuales, tipográficas y de estructura ya están tomadas y fijadas aquí. Tu trabajo
no es diseñar — es **investigar, escribir y ensamblar** siguiendo las constantes.

Tres reglas que gobiernan todo lo demás:

1. **No inventes parámetros visuales.** Colores, fuentes, coordenadas, duraciones
   y proporciones están fijados en la sección 6. Si algo no está especificado,
   es porque no debe variar entre videos.
2. **Consolida tool calls.** Cada fase tiene un script único (`fase_N.py` o
   `fase_N.sh`). Escribe el script completo, córrelo una vez, lee la salida.
   No hagas 15 llamadas a bash para lo que cabe en una. El objetivo explícito
   de este harness es minimizar tokens.
3. **Los gates son no-negociables.** Las secciones marcadas `GATE` abortan la
   corrida si fallan. No los interpretes con flexibilidad, no los saltes "porque
   casi pasa". Es preferible no publicar hoy que publicar algo sin respaldo.

---

## 1. Setup

Corre `setup.sh`. Además de lo habitual, este formato necesita:

**Fuentes (instancias estáticas — NO la variable).** `rsvg-convert` y PIL no
resuelven ejes variables de forma confiable: si pasas la variable, todo sale en
peso regular y se pierde el contraste del sistema. Descarga de Google Fonts:

```
fonts/Fraunces_9pt-Bold.ttf
fonts/Fraunces_9pt-MediumItalic.ttf
fonts/IBMPlexMono-SemiBold.ttf
fonts/IBMPlexMono-Regular.ttf
```

**Python:** `pillow`, `matplotlib` (`pip install --break-system-packages`).

**`.env`:** `PEXELS_API_KEY`, credenciales de YouTube (`YT_CLIENT_ID`,
`YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`), `GITHUB_TOKEN` (historial).
Opcionales: `TEMA_FIJO`, `CREATE_THUMBNAIL`.

**Sin paleta rotativa.** A diferencia del noticiero y el deep dive, este canal
tiene **una sola paleta fija** (sección 6). La identidad se construye por
repetición, no por variación.

La cama musical sí rota (sección 7), reusando las `bed_0..4_norm.mp3` que ya
existen. Es la única variación permitida entre videos.

---

## 2. Selección de la pregunta

### 2.1. Ramificación por `TEMA_FIJO`

```bash
TEMA_FIJO="${TEMA_FIJO:-NONE}"
```

- **`NONE`** → flujo de cola (2.2).
- **Cualquier otro valor** → esa es la pregunta. Salta a verificación (sección 3).
  No cambies de tema aunque encuentres algo "mejor".

### 2.2. Flujo de cola

La cola vive en el repo `anmalaver/AI-Daily-news-assets`, en
`topics-history/why-queue.json`. **Es la fuente de verdad de qué se publica cada
día** — la rutina no inventa preguntas mientras haya ideas `ready` en la cola.

Descárgala con **dos métodos en orden**, porque `raw.githubusercontent.com` ha
devuelto 404 para este repo:

```bash
# 1) intento directo
curl -fsSL -o why-queue.json \
  "https://raw.githubusercontent.com/anmalaver/AI-Daily-news-assets/main/topics-history/why-queue.json" \
|| {
  # 2) fallback por tarball (el método confiable para este repo)
  curl -fsSL "https://codeload.github.com/anmalaver/AI-Daily-news-assets/tar.gz/refs/heads/main" \
    | tar -xz --wildcards --strip-components=1 -O \
      "*/topics-history/why-queue.json" > why-queue.json
}
```

Si ambos fallan o el archivo no existe, asume `{"ideas": []}` y salta a 2.3.

### 2.2.1. Verificación cruzada contra el historial (obligatoria)

**No confíes solo en el campo `status`.** El `status` pasa a `published` únicamente
si el push al repo funciona al final de la corrida, y ese push puede fallar por
red, token o permisos — casos en los que el prompt te indica publicar igual y
reportar. La consecuencia: una idea ya publicada sigue diciendo `ready` y mañana
se produce otra vez, entera.

Por eso, antes de aceptar una idea de la cola, descarga también el historial y
compara por `slug`:

```bash
curl -fsSL -o why-history.json \
  "https://raw.githubusercontent.com/anmalaver/AI-Daily-news-assets/main/topics-history/why-history.json" \
|| {
  curl -fsSL "https://codeload.github.com/anmalaver/AI-Daily-news-assets/tar.gz/refs/heads/main" \
    | tar -xz --wildcards --strip-components=1 -O \
      "*/topics-history/why-history.json" > why-history.json
} || echo '{"topics": []}' > why-history.json
```

```python
"""Pick the next queue idea, skipping anything already published."""
import json

queue = json.load(open("why-queue.json")).get("ideas", [])
history = json.load(open("why-history.json")).get("topics", [])
published = {t["slug"] for t in history}

candidata = None
saltadas = []
for idea in queue:
    if idea.get("status") != "ready":
        continue
    if idea["publish_date"] > HOY:          # HOY = 'YYYY-MM-DD' en America/New_York
        continue
    if idea["slug"] in published:
        saltadas.append(idea["slug"])       # ready en cola pero ya en historial
        continue
    candidata = idea
    break

if saltadas:
    print(f"QUEUE_DESYNC={saltadas}  — status quedó 'ready' tras un push fallido")
```

**Reporta siempre `QUEUE_DESYNC` en la entrega.** Cada slug en esa lista es una
idea cuyo `status` no se actualizó: el video ya existe pero la cola no lo sabe.
Son las que hay que corregir a mano en el repo.

**Regla de precedencia:** el historial manda sobre la cola. Si un slug aparece en
ambos, se salta — es preferible saltar un día que publicar un duplicado, porque
el duplicado dispara la política de contenido inauténtico de YouTube y el daño
es a nivel de canal, no de video.

Si tras el filtro no queda ninguna candidata, salta a 2.3.

---

Estructura de cada idea (ya pre-investigada, con fuente y chart definidos):

```json
{
  "id": "why-0007",
  "slug": "time-speeds-up-with-age",
  "question_en": "Why does time speed up as you age?",
  "myth": "A year is 20% of your life at five. Only 2% at fifty.",
  "myth_attribution": "Janet, 1897. Repeated ever since.",
  "verdict": "It's not your age. It's your calendar.",
  "takeaway": "What would you take off this week?",
  "sources": [
    {"cite": "Friedman & Janssen, Acta Psychologica, 2010", "n": 1865,
     "doi": "10.1016/j.actpsy.2010.01.004", "year": 2010}
  ],
  "chart_spec": {
    "type": "line",
    "x_label": "age 20 → 80",
    "series": [
      {"name": "last 10 years", "values": [44,55,66,78,86,88,88], "color": "punch"},
      {"name": "week / month / year", "values": [72,71,73,72,71,72,73], "color": "mute"}
    ],
    "caption": "perceived speed of time · n = 1,865"
  },
  "stimulus_queries": ["vintage alarm clocks", "antique wall clocks",
                       "crowd silhouette sunset", "watchmaker repairing",
                       "traffic light trails night"],
  "status": "ready",
  "publish_date": "2026-10-07"
}
```

### 2.3. Minería en vivo (fallback)

Solo si la cola está vacía. Busca candidatas en:

- Google autocomplete / People Also Ask con semillas `why do people…`,
  `why do we…`, `why does everyone…`
- r/AskSocialScience, r/askpsychology, r/explainlikeimfive — ordenar por top/all-time
- YouTube autocomplete con las mismas semillas

**Descarta contra el historial** (`why-history.json`, mismos 20 últimos slugs).

Arma 3 candidatas, elige una al azar con RNG real:

```bash
IDX=$(python3 -c "import random; print(random.randrange(3))")
```

Muéstrame las 3 y cuál quedó. Sigue sin esperar confirmación.

---

## 3. GATE de evidencia (ANTES de escribir)

Este es el gate que define si el canal tiene credibilidad o es pop-psychology
más. **Aplica los cuatro cortes en orden. Si falla cualquiera, descarta la
pregunta y toma la siguiente de la cola.**

### 3.1. Corte de fuente

La respuesta debe apoyarse en **al menos un meta-análisis, revisión sistemática
o estudio con n ≥ 500**, publicado en revista revisada por pares. Guarda cita
completa y DOI.

No sirven: artículos de divulgación, libros de autoayuda, TED talks, blogs,
notas de prensa universitarias sin el paper detrás.

### 3.2. Corte de replicación

**Cualquier hallazgo de psicología social anterior a 2015 debe verificarse
contra la crisis de replicación antes de usarse.** Busca explícitamente
`"<hallazgo>" replication failed` o `"<hallazgo>" meta-analysis`.

Prohibidos salvo que el video sea precisamente sobre su caída: social priming,
power posing, ego depletion, facial feedback, el efecto Macbeth, la mayoría de
los efectos de "priming" conductual.

Si el hallazgo central del video está en esa familia y no encuentras una
replicación preregistrada posterior a 2015 que lo sostenga, **descarta la
pregunta**.

### 3.3. Corte de graficabilidad

Debe existir un dato que se pueda convertir en **una línea, barra o distribución
que se lea en 4 segundos en pantalla vertical**. Si el hallazgo solo se puede
expresar en prosa, no es video de este canal.

Máximo 2 series, máximo 9 puntos por serie. Si necesitas más, el dato es
demasiado complejo para el formato.

### 3.4. Corte de contraintuición

La respuesta correcta debe contradecir la respuesta que el espectador daría.
Prueba binaria: **¿el acto 05 sorprende a alguien que ya leyó el acto 02?**

Si la respuesta es "sí, básicamente confirma lo que uno pensaba", descarta.
Sin giro no hay short.

### 3.5. Corte de temperatura política

Descarta preguntas donde la respuesta honesta requiera tomar partido en un
debate político activo (voto, inmigración, aborto, armas, identidad de género,
conflictos geopolíticos). El canal explica comportamiento humano; no hace
comentario político.

**Reporta al final los cuatro cortes con su veredicto**, para auditoría.

---

## 4. Guion — 6 actos

Escribe la historia completa primero, luego trocéala. El arco es fijo:

| Acto | Función | Statement | Línea de apoyo | Narración |
|---|---|---|---|---|
| 1 | **El gancho** | la tensión, no la pregunta | la pregunta literal | 14-20 palabras |
| 2 | **El mito** | la creencia entrecomillada | su origen y por qué pegó | 24-32 palabras |
| 3 | **El dato** | la cifra que lo rompe | qué muestra el gráfico | 30-40 palabras |
| 4 | **El mecanismo** | por qué pasa | cómo opera en la práctica | 30-40 palabras |
| 5 | **El veredicto** | la frase memorizable | la consecuencia | 16-22 palabras |
| 6 | **El cierre** | la pregunta de vuelta | — | 10-14 palabras |

**Total narración: 125-165 palabras (≈70-85s a rate +8%).**

La v3 apuntaba a 90-115 palabras y 50-60s. Quedó corto: el video se veía bien
pero se terminaba sin haber enseñado gran cosa. Un espectador que llega por una
pregunta de psicología quiere **entender el mecanismo**, no solo saber que su
intuición estaba mal. Subir a 70-85s es barato en retención y caro no hacerlo.

### 4.0.1. El acto 4 cambió de función

Era "el matiz" — la variable que sí predice el fenómeno, en una frase. Ahora es
**el mecanismo**: cómo funciona realmente la cosa, con suficiente detalle para
que el espectador pueda explicárselo a alguien más al día siguiente.

Es el acto con más peso informativo del video y el que justifica su existencia.
Si al terminar de escribirlo el espectador no puede responder "¿y por qué pasa
eso?", el acto no está hecho.

Estructura interna del acto 4, en las 30-40 palabras:
1. El mecanismo en una frase.
2. Cómo se manifiesta en un caso concreto y reconocible.
3. Qué predice que la intuición popular no predice.

### 4.1. Reglas de escritura por acto

**Acto 1 — el gancho.** Cambio importante respecto a la v3.

La v3 ponía la pregunta literal en pantalla como statement. Es un gancho débil:
una pregunta sin tensión no detiene el scroll, y el espectador ya sabe que el
video va a responderla.

Ahora el statement es **la tensión**, y la pregunta baja a la línea de apoyo:

| Débil (v3) | Fuerte (v4) |
|---|---|
| Do opposites really attract? | Everyone believes this. 79,000 couples say otherwise. |
| Why does time speed up as you age? | The explanation everyone repeats has never been measured. |

El gancho nombra el conflicto entre lo que se cree y lo que se midió. La
pregunta literal sigue siendo el **título del video** — ahí sí va textual,
porque es la query de búsqueda. En pantalla va la tensión.

Máximo 4 líneas de 20 caracteres.

**Acto 2 — el mito.** La creencia popular entrecomillada, más su atribución y
**por qué pegó**. Ese "por qué pegó" es nuevo: casi siempre hay una razón (suena
bien, confirma algo que queremos creer, viene de un estudio mal leído), y
nombrarla ya es información.

**Acto 3 — el dato.** El estudio, la n y el hallazgo. La cifra que rompe el mito
va como statement en grande. Si el estudio tiene un segundo hallazgo relevante,
métrelo en la línea de apoyo — no lo desperdicies.

**Acto 4 — el mecanismo.** Ver 4.0.1.

**Acto 5 — el veredicto.** Una afirmación, máximo 4 líneas de 20 caracteres.
Memorizable y citable. Reglas duras:
- Afirmación, nunca pregunta.
- Anclada en el dato del acto 3, no en especulación nueva.
- Sin "quizás", "podría ser", "los expertos sugieren".
- Sin moraleja. "You married your mirror." sí. "Love is complex." no.

La línea de apoyo del acto 5 lleva **la consecuencia**: qué cambia si esto es
cierto. Es lo que convierte el veredicto en algo utilizable.

**Acto 6 — el cierre.** Una pregunta corta que el espectador pueda contestarse a
sí mismo hoy. Prohibido: "comment below", "subscribe for more", "what do you
think?".

### 4.1.1. La línea de apoyo — densidad de lectura

Todos los actos salvo el 6 llevan una **segunda línea** bajo el statement, en
Fraunces MediumItalic 42px. No es decoración: en `why_003` los frames tenían una
sola frase corta y mucho aire, y el espectador terminaba de leer en 2 segundos
sobre un frame de 8. Ese hueco es donde se pierde retención.

La línea de apoyo **no repite el statement con otras palabras**. Aporta lo que el
statement da por supuesto:

| Acto | Statement | Línea de apoyo |
|---|---|---|
| 1 | Everyone believes this. 79,000 couples say otherwise. | Do opposites really attract? |
| 2 | *"Opposites attract."* | Folk wisdom — and its exact opposite, repeated just as often. |
| 3 | Correlated on almost everything. | 133 traits measured. Not one came out negative. |
| 4 | It's the funnel, not chemistry. | You only ever meet people your schooling, city and job already filtered. |
| 5 | You married your mirror. | Which means compatibility was decided before you met. |

Reglas:
- **15-25 palabras**, máximo 4 líneas de 34 caracteres.
- Va en el manifiesto como campo `support` del acto.
- **No se narra.** La voz lleva el guion, los ojos leen esto. Son dos canales;
  llenarlos con lo mismo desperdicia uno.
- El acto 6 no la lleva: el cierre funciona por quedar colgando.

### 4.2. Manifiesto

```json
{
  "fecha": "<FECHA>",
  "id": "why-0007",
  "voz": "en-US-AvaMultilingualNeural",
  "titulo_video": "...",
  "descripcion": "...",
  "tags": ["..."],
  "actos": [
    {
      "n": 1,
      "etiqueta": "the question",
      "layout": "photo_full",
      "texto": ["Why does time", "speed up as", "you age?"],
      "support": null,
      "estilo": "q",
      "guion": "Why does time speed up as you get older?",
      "stimulus": "vintage alarm clocks"
    }
  ],
  "chart_spec": { "...": "..." },
  "sources": [ { "...": "..." } ]
}
```

### 4.3. Tono

Ensayo editorial, no divulgación entusiasta. El narrador sabe algo que el
espectador no y lo dice sin celebrarlo. Cercano a un artículo de The Atlantic
leído en voz alta; lejos de "¡dato curioso!".

Prohibido: signos de admiración, "increíble", "te va a volar la cabeza",
"la ciencia dice", "los científicos descubrieron que" (di qué estudio).

---

## 5. Imágenes (Pexels)

Una foto por acto, **excepto el acto 3**, que no lleva foto — el gráfico es la
imagen. Son 5 descargas.

Usa las `stimulus_queries` de la cola. Si la cola no trae, derívalas del
contenido de cada acto.

```python
import os, urllib.request, json
HDR = {"Authorization": os.environ["PEXELS_API_KEY"],
       "User-Agent": "why-pipeline/1.0"}

def fetch(query, out):
    url = ("https://api.pexels.com/v1/search?"
           f"query={urllib.parse.quote(query)}&orientation=portrait"
           "&per_page=15&size=large")
    req = urllib.request.Request(url, headers=HDR)
    data = json.load(urllib.request.urlopen(req))
    for p in data["photos"]:
        if p["width"] >= 1080 and p["height"] >= 1350:
            src = p["src"]["large2x"]
            urllib.request.urlretrieve(src, out)
            return p["photographer"]
    return None
```

**Criterios de descarte** (aplica sin preguntar):
- Fotos con texto legible, logos o marcas visibles.
- Fotos claramente generadas por IA (manos raras, texturas plásticas, simetría
  imposible). Pexels las marca a veces; si dudas, descarta.
- Retratos frontales de una sola persona mirando a cámara — leen como stock.
  Prefiere manos, siluetas, escenas, objetos, multitudes de espaldas.
- La misma foto en dos actos del mismo video.

Guarda `photographer` de cada una para los créditos.

---

## 6. CONSTANTES VISUALES (no modificar)

Todo lo de esta sección es fijo entre videos. No lo re-decidas, no lo "mejores",
no lo adaptes al tema. La identidad del canal vive aquí.

### 6.1. Paleta

```python
PAPER       = "#FBFAF5"   # fondo único
INK         = "#0F0F0E"   # texto principal sobre papel
MUTED       = "#6E6E68"   # labels, metadata, citas
PUNCH       = "#EF3E36"   # SOLO: signo "?" final, punto del veredicto, comilla de
                          # apertura, líneas. NUNCA una palabra ni una frase.
PUNCH_DEEP  = "#C42820"   # punch cuando es texto ≤20px
AMBER       = "#F5B800"   # SOLO fills: barras, bloques, subrayados
AMBER_DEEP  = "#B07C00"   # ámbar cuando es texto
VERDICT     = "#00A67E"   # veredicto, display grande
VERDICT_DEEP= "#00805F"   # verde cuando es texto pequeño
```

**Dos reglas gobiernan la paleta:**

1. **Los colores saturados pintan formas; sus versiones `_DEEP` pintan letras.**
   Nunca `AMBER` como color de texto. Nunca `PUNCH` en texto menor a 20px.
2. **El fondo decide el color del texto, no el rol.** Ver la tabla obligatoria
   de 6.3.1 — es la regla que más se rompe al ejecutar y la que produjo el peor
   fallo de la primera versión: la pregunta entera en rojo sobre una foto
   rojiza, y texto tinta sobre foto oscura.

`PUNCH` y `VERDICT` **jamás pintan una frase**. Solo signos de puntuación
aislados y líneas. El énfasis del canal lo carga el peso tipográfico de
Fraunces Bold, no el color.

### 6.2. Tipografía

| Rol | Fuente | Tamaño (en canvas 1080×1920) |
|---|---|---|
| Pregunta / veredicto (`estilo: "q"`) | Fraunces Bold 700 | 96px, interlineado 100 |
| Mito / matiz / cierre (`estilo: "myth"`) | Fraunces MediumItalic 500 | 62px, interlineado 78 |
| Números y correlaciones | IBM Plex Mono SemiBold 600, tabular | 104px |
| Etiqueta de acto, metadata | IBM Plex Mono Regular 400 | 26px, letter-spacing 3.8 |
| **Línea de apoyo** (`support`) | Fraunces MediumItalic 500 | 44px, interlineado 56 |
| Pie de gráfico, `caption` | IBM Plex Mono Regular 400 | 34px |
| Cita académica | IBM Plex Mono Regular 400 | 30px, interlineado 42 |

Ninguna otra fuente, ningún otro peso.

**Los tamaños de metadata subieron respecto a la v2** (cita 22→30px, caption
26→34px). A 22px sobre un canvas de 1920 el texto es ilegible en un teléfono: en
la corrida `why_003` la cita del paper quedó imposible de leer. La regla práctica:
**nada por debajo de 30px en este canvas.**

### 6.3. Layout del canvas (1080×1920)

Medido sobre una captura real del reproductor de Shorts, no estimado. La UI de
YouTube ocupa **mucho más de lo que parece**, y no solo abajo: el riel de
botones (me gusta, comentar, guardar, compartir) come la franja derecha en toda
la mitad inferior.

```
y = 0      ┌──────────────────────────────┐
           │   zona segura superior       │  260px — sin nada
y = 260    ├──────────────────────────────┤
           │                              │
           │   ZONA A — ancho completo    │  x 82 → 998
           │   statement principal        │  y 260 → 880
           │                              │
y = 880    ├───────────────────┬──────────┤
           │  ZONA B           │  RIEL DE │  x 82 → 860
           │  texto secundario │  BOTONES │  y 880 → 1400
           │  apoyo, caption,  │  (UI de  │
           │  cita             │  YouTube)│
y = 1400   ├───────────────────┴──────────┤
           │   MUERTA — nombre de canal,  │  520px
           │   título, controles          │  NADA aquí, nunca
y = 1920   └──────────────────────────────┘
```

| Zona | x | y | Qué va |
|---|---|---|---|
| A | 82 → 998 | 260 → 880 | Statement principal, gráfico |
| B | 82 → **860** | 880 → 1400 | Apoyo, caption, cita, números |
| Muerta | — | 1400 → 1920 | **Nada** |

**El límite de x=860 en la zona B no es negociable.** En la corrida `why_003`
el caption y la línea de apoyo se extendían hasta x=998 y quedaron literalmente
debajo de los botones "Me gusta" y "Guardar" — texto que el espectador nunca
puede leer.

**La zona muerta son 520px, no 420.** La fila con el nombre del canal y el
título del video empieza en y≈1426 en coordenadas de video.

```python
ZONE_A = (82, 260, 998, 880)     # statement, gráfico
ZONE_B = (82, 880, 860, 1400)    # apoyo, caption, cita
# y > 1400: prohibido
```

**Identificador `why · #NNN`**: arriba a la derecha, dentro de la zona segura
superior pero **por debajo de y=200** — en la captura quedó pisado por la hora
y la batería del teléfono. Mejor: muévelo al pie de la zona B, alineado a la
izquierda, mono 26px. Ahí nada lo tapa.

`NNN` sale del `id` de la idea (`why-0003` → `003`) y es **constante en los seis
frames**.

```python
IDEA_NUM = idea["id"].split("-")[-1]
BADGE = f"why · #{IDEA_NUM[-3:]}"      # igual en los 6 frames
```

**Zona de reposo.** Con foto, el scrim adaptativo (6.3.2) ya cubre y>1400. **No
apliques degradado adicional sobre foto** — produce la franja gris con borde
duro. En el frame de papel (solo acto 3), degradado de `#FBFAF5` a `#CFC8BA`
con easing cuadrático, empezando en y=1400.

```python
"""Bottom readability gradient — paper frame ONLY."""
from PIL import ImageDraw

def rest_zone_gradient(canvas):
    top, bottom = (251, 250, 245), (207, 200, 186)
    d = ImageDraw.Draw(canvas)
    for i in range(520):
        k = (i / 519) ** 2
        rgb = tuple(int(top[c] + (bottom[c] - top[c]) * k) for c in range(3))
        d.line([(0, 1400 + i), (1080, 1400 + i)], fill=rgb)
    return canvas
```

### 6.3.-1. Medida de línea — comprimir, no encoger

**Cuando el texto no cabe, acorta la línea y añade líneas. Nunca bajes la
fuente.** El tamaño es lo que hace el canal legible en un teléfono; la medida
de línea es lo que lo hace caber.

| Rol | Máx. caracteres por línea | Máx. líneas |
|---|---|---|
| Statement (`q`) | **20** | 4 |
| Mito / matiz / cierre (`myth`) | **26** | 4 |
| Línea de apoyo (`support`) | **34** | 4 |
| Caption de gráfico | **38** | 2 |
| Cita académica | **42** | 3 |

Interlineados más cerrados que la v3, porque con más líneas el bloque necesita
compactarse:

| Rol | Tamaño | Interlineado |
|---|---|---|
| Statement | 96px | **98** |
| Myth | 62px | **72** |
| Support | 42px | **52** |
| Caption | 34px | 44 |
| Cita | 30px | 40 |

El resultado es un bloque más denso y más alto: exactamente lo que se busca.
Un statement de 4 líneas cortas se lee más rápido que uno de 2 líneas largas, y
llena la zona A sin necesidad de agrandar la fuente.

### 6.3.0. El papel es la excepción, no el default

**Solo el acto 3 tiene fondo de papel.** Los otros cinco van con foto a sangre
completa, texto encima con scrim.

La v2 permitía `photo_band` en el acto 2: foto arriba al 34%, texto sobre papel
abajo. Se ve flojo — parte el frame en dos, desperdicia la foto y el bloque
blanco inferior lee como diapositiva de presentación, no como el canal.

Regla: **si el frame no lleva gráfico, lleva foto a sangre.** El papel existe
porque un gráfico necesita fondo claro para ser legible; fuera de ese caso no
tiene justificación.

### 6.3.1. Color de texto por fondo — TABLA OBLIGATORIA

**Esta es la regla que más se rompe al ejecutar.** La sección 6.1 define los
colores por rol semántico, pero el rol no dice nada sobre el fondo. Sin este
binding el modelo elige por su cuenta y produce tinta oscura sobre foto oscura,
o la pregunta entera en rojo sobre una foto rojiza.

**El fondo manda sobre el rol. Siempre.**

| Elemento | Sobre papel (acto 3) | Sobre foto (actos 1,2,4,5,6) |
|---|---|---|
| Pregunta / veredicto | `INK` #0F0F0E | `PAPER` #FBFAF5 |
| Mito / matiz / cierre | `INK` #0F0F0E | `PAPER` #FBFAF5 |
| Etiqueta de acto, `why · #NNN` | `MUTED` #6E6E68 | `PAPER` al 62% de opacidad |
| Cita académica | `MUTED` #6E6E68 | `PAPER` al 62% de opacidad |
| Números grandes | `PUNCH_DEEP` / `AMBER_DEEP` | `#FF6B60` / `#F5B800` |
| Cita textual (comillas `"`) | `PUNCH` #EF3E36 | `PUNCH` #EF3E36 |

**El acento nunca pinta una frase completa.** `PUNCH` y `VERDICT` se usan
exclusivamente en:
- el signo final de la pregunta (`?`),
- el punto final del veredicto (`.`),
- la comilla de apertura del mito,
- una línea o subrayado.

Nunca en las palabras. Una pregunta completa en rojo sobre foto es ilegible y
además grita — el contraste del canal viene del peso tipográfico, no del color.

**El texto sobre foto lleva sombra siempre:** offset (0,3), blur 18,
`rgba(15,15,14,0.55)`. Es lo que sostiene la legibilidad cuando el scrim no
alcanza.

### 6.3.2. Scrim adaptativo — MEDIDO, no fijo

Un scrim de opacidad fija falla, porque las fotos de Pexels varían de luminancia
entre sí y entre zonas. **Mide la región exacta bajo el texto y ajusta.**

```python
"""Adaptive scrim: darken only as much as the photo under the text requires."""
from PIL import Image, ImageDraw, ImageStat

TEXT_BOX = (82, 380, 998, 1500)   # la zona de acción
TARGET_L = 78                     # luminancia máxima admitida bajo texto crema

def needed_alpha(img):
    """Return the scrim alpha (0-255) that brings the text region to TARGET_L."""
    region = img.crop(TEXT_BOX).convert("L")
    mean = ImageStat.Stat(region).mean[0]
    if mean <= TARGET_L:
        return 90                      # piso: siempre algo de scrim
    alpha = int(255 * (mean - TARGET_L) / mean)
    return max(90, min(215, alpha))    # techo para no matar la foto

def apply_scrim(img):
    a = needed_alpha(img)
    ov = Image.new("RGBA", img.size, (0, 0, 0, 0))
    d = ImageDraw.Draw(ov)
    # degradado vertical: más oscuro arriba y abajo, más claro al centro
    for y in range(img.size[1]):
        t = y / img.size[1]
        edge = max(0.0, 1 - abs(t - 0.5) * 2)      # 0 en bordes, 1 al centro
        val = int(a * (1 - 0.45 * edge))
        d.line([(0, y), (img.size[0], y)], fill=(11, 10, 9, val))
    return Image.alpha_composite(img.convert("RGBA"), ov).convert("RGB")
```

**Verificación obligatoria por frame:** tras componer, calcula el contraste real
entre el color del texto y la media de la región que ocupa. **Mínimo 4.5:1.** Si
no llega, sube el alpha del scrim en pasos de 20 y recompón. Si a 215 sigue sin
llegar, **descarta esa foto y baja la siguiente** de `stimulus_queries` — la foto
está mal, no el scrim.

```python
def contrast_ratio(rgb_a, rgb_b):
    """WCAG contrast ratio between two RGB colors."""
    def lum(c):
        s = [v / 255 for v in c]
        s = [v / 12.92 if v <= 0.03928 else ((v + 0.055) / 1.055) ** 2.4 for v in s]
        return 0.2126 * s[0] + 0.7152 * s[1] + 0.0722 * s[2]
    la, lb = sorted([lum(rgb_a), lum(rgb_b)], reverse=True)
    return (la + 0.05) / (lb + 0.05)
```

### 6.3.3. Llenado vertical

El texto debe ocupar **entre 70% y 90%** de la zona de acción (y 380→1500). Por
debajo de 60% el frame se ve vacío y el texto diminuto; por encima de 90% se
siente apretado.

Si un acto queda corto, **sube el tamaño de fuente** hasta llenar — no dejes el
tamaño base con aire muerto arriba y abajo. Los tamaños de la sección 6.2 son el
punto de partida, y pueden crecer hasta un 25% para llenar. Nunca encoger por
debajo del 85% del valor base.

### 6.4. Rotación de layout por acto

Seis frames idénticos en estructura leen como plantilla — el patrón que penaliza
la política de contenido inauténtico de YouTube. La rotación es fija:

| Acto | `layout` | Descripción |
|---|---|---|
| 1 | `photo_full` | Foto a sangre + scrim + pregunta centrada |
| 2 | `photo_full` | Foto a sangre + mito entrecomillado, alineado a la izquierda |
| 3 | `chart_only` | **Único frame de papel.** Gráfico + cifra + cita |
| 4 | `photo_duo` | Foto a sangre, texto en dos bloques: dato arriba, lectura abajo |
| 5 | `photo_mirror` | Foto partida en dos mitades, la derecha volteada |
| 6 | `photo_full` | Foto a sangre + cierre centrado |

La variación ya no viene de alternar papel y foto — viene de **dónde vive el
texto dentro del frame**: centrado, a la izquierda, en dos bloques, sobre un
espejo. Es más sutil y se ve mucho mejor que partir frames en bandas.

`photo_mirror` es literal cuando el veredicto habla de simetría o reflejo. Si no
aplica al tema, usa `photo_full` con el texto desplazado hacia abajo y anótalo
en el reporte.

### 6.5. Tratamiento de foto (duotono)

Las fotos de Pexels **nunca** se usan crudas. El duotono es lo que impide que el
canal parezca un slideshow de stock. Script fijo:

```python
"""Apply the channel duotone treatment to a stock photo."""
import random
from PIL import Image, ImageEnhance, ImageOps

PAPER_RGB = (251, 250, 245)
INK_RGB   = (26, 18, 16)      # tinta cálida, NO negro neutro
TARGET    = (1080, 1920)

MIDTONES = {          # el midtone da la temperatura; rota por acto
    1: (178, 80, 52), 2: (184, 112, 40), 4: (168, 76, 56),
    5: (150, 100, 74), 6: (186, 124, 34),
}

def ramp(shadow, highlight, mid):
    out = []
    for i in range(256):
        t = i / 255.0
        if t < 0.5:
            k = t * 2
            rgb = [shadow[c] + (mid[c] - shadow[c]) * k for c in range(3)]
        else:
            k = (t - 0.5) * 2
            rgb = [mid[c] + (highlight[c] - mid[c]) * k for c in range(3)]
        out.append(tuple(int(v) for v in rgb))
    return out

def duotone(src, dst, act):
    img = Image.open(src).convert("RGB")
    img = ImageOps.fit(img, TARGET, Image.LANCZOS, centering=(0.5, 0.42))
    img = ImageEnhance.Contrast(img.convert("L")).enhance(1.25)
    table = ramp(INK_RGB, PAPER_RGB, MIDTONES[act])
    r = img.point([c[0] for c in table])
    g = img.point([c[1] for c in table])
    b = img.point([c[2] for c in table])
    img = Image.merge("RGB", (r, g, b))
    grain = Image.new("L", img.size)
    rnd = random.Random(7)                       # seed fija = grano reproducible
    grain.putdata([rnd.randint(110, 145) for _ in range(img.size[0] * img.size[1])])
    img = Image.blend(img, Image.merge("RGB", (grain, grain, grain)), 0.06)
    img.save(dst, "JPEG", quality=88)
```

**Verificación obligatoria:** después de tratar, abre una imagen y confirma que
tiene temperatura cálida visible. Si sale gris neutro, el midtone no se aplicó
— revisa que estés pasando `mid` y no `None`.

### 6.6. Gráfico (acto 3)

matplotlib, sin estilo por defecto. Parámetros fijos:

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
from matplotlib import font_manager

font_manager.fontManager.addfont("fonts/IBMPlexMono-Regular.ttf")
plt.rcParams.update({
    "font.family": "IBM Plex Mono",
    "figure.facecolor": "#FBFAF5",
    "axes.facecolor": "#FBFAF5",
    "axes.edgecolor": "#0F0F0E",
    "axes.linewidth": 1.8,
    "text.color": "#0F0F0E",
    "xtick.labelsize": 30,
    "xtick.color": "#6E6E68",
    "ytick.color": "#6E6E68",
    "font.size": 30,
})

fig, ax = plt.subplots(figsize=(9.16, 5.2), dpi=100)
ax.spines[["top", "right", "left"]].set_visible(False)
ax.set_yticks([])
# serie principal en PUNCH (#EF3E36), lw=4
# series de contraste en #C9C4BA, lw=2.4
# sin grid, sin leyenda de matplotlib — las etiquetas van como texto al final
# de cada línea, en IBM Plex Mono 18px
fig.savefig("chart.png", transparent=True, bbox_inches="tight", pad_inches=0.3)
```

**Guarda el PNG con `transparent=True` y pégalo sobre el papel del frame.** Si
lo guardas con fondo opaco, el `#FBFAF5` de matplotlib no coincide exactamente
con el del canvas y aparece un rectángulo gris visible alrededor del gráfico
— pasó en la primera corrida.

Reglas del gráfico:
- Sin título dentro del gráfico (el título va como texto del frame).
- Sin eje Y numerado. La forma es el mensaje, no los valores exactos.
- Etiquetas de serie al final de cada línea, no en leyenda, **mínimo 30px**.
- Máximo 2 series.
- **Valores sobre las barras/puntos, mono SemiBold 32px.** Son el dato; tienen
  que leerse sin esfuerzo.
- **Etiquetas del eje X obligatorias y legibles (30px).** Un gráfico de barras
  sin saber qué mide cada barra no informa nada. Si los nombres no caben,
  reduce el número de barras — no el tamaño de la fuente.

**El acto 3 debe llenar el frame.** En `why_003` el gráfico ocupó la franja
central y dejó ~350px muertos arriba y ~300px abajo. Composición del frame de
papel, de arriba a abajo dentro de la zona de acción (y 280→1500):

| Elemento | Altura aprox. |
|---|---|
| Statement (la cifra que rompe el mito), Fraunces Bold | 200px |
| Gráfico | 620px |
| `caption` bajo el gráfico, mono 34px | 60px |
| Línea de apoyo, Fraunces italic 44px | 120px |
| Cita académica, mono 30px, 2-3 líneas | 130px |

Suma ≈1130px de 1220 disponibles: 93% de llenado. Si sobra más de 150px, sube
el alto del gráfico hasta consumirlo.

### 6.7. Prohibiciones visuales

- Fondos oscuros como superficie de diseño. El único oscurecimiento permitido es
  el scrim sobre foto (6.3.2) y el degradado de reposo del frame de papel (6.3).
- **Bandas o barras sólidas de color.** No hay franja inferior, ni cabecera, ni
  bloques de color detrás del texto. El scrim es degradado, nunca un borde duro.
- Texto por debajo de y=1400, o a la derecha de x=860 entre y 880 y 1400.
- Bajar el tamaño de fuente para que quepa el texto. Se acorta la línea, no la letra.
- **Etiquetas de acto** ("the question", "the data"). Eliminadas en v3.
- **Fondo de papel fuera del acto 3.** Si no hay gráfico, hay foto a sangre.
- **Degradado de reposo sobre frames con foto.** El scrim ya lo cubre; añadir
  otro produce la franja gris con borde duro de la v2.
- Texto por debajo de 30px en el canvas de 1920.
- Cualquier fuente que no sea Fraunces o IBM Plex Mono.
- Emojis, iconos decorativos, flechas de clip-art.
- Gradientes de más de dos paradas.
- Texto sobre foto sin scrim.
- Fotos crudas sin duotono.
- Sombras paralelas, bordes redondeados en el canvas del video.
- Animaciones de texto que entren volando o rebotando.

---

## 7. Audio

### 7.1. Voz

**`en-US-AvaMultilingualNeural`, `--rate=+8%`, `--pitch=+0Hz`.**

Misma voz que el noticiero y el deep dive — decisión deliberada, no inercia. Los
tres canales comparten voz para que el archivo completo suene como una casa
editorial y no como tres productos sueltos. Lo que diferencia este formato es el
**ritmo**, no el timbre:

| Formato | Rate | Registro |
|---|---|---|
| Noticiero | +13% | urgente, denso |
| Deep dive | +13% | explicativo |
| **Why** | **+8%** | **reflexivo — la pausa es parte del argumento** |

A +8% cada frase aterriza antes de que empiece la siguiente. En un video donde
el acto 5 tiene que sonar a conclusión y no a dato más, esa respiración es la
mitad del efecto.

**Si quieres comparar de oído** (un solo cambio de parámetro):
`en-US-EmmaMultilingualNeural` (más grave, más cercana al registro de audiolibro)
o `en-US-AndrewMultilingualNeural` (masculina, si el canal llega a tener voz
propia). No cambies sin correr al menos 5 videos con cada una — la voz es de las
decisiones que solo se evalúan con retención, no con gusto.

### 7.2. Síntesis

```bash
edge-tts --voice "en-US-AvaMultilingualNeural" --rate=+8% --pitch=+0Hz \
  --file acto_N.txt --write-media raw_N.mp3 --write-subtitles sub_N.srt
ffmpeg -y -i raw_N.mp3 -af "apad=pad_dur=$PAD_N" -c:a libmp3lame -b:a 192k voz_N.mp3
DUR_N=$(ffprobe -v error -show_entries format=duration -of csv=p=0 voz_N.mp3)
```

**Pads diferenciados por acto** — la respiración es narrativa, no técnica:

| Acto | Pad | Por qué |
|---|---|---|
| 1 | 0.9s | la pregunta necesita quedar colgando |
| 2, 3, 4 | 0.5s | el cuerpo del argumento fluye |
| 5 | 0.9s | el veredicto necesita aire antes del cierre |
| 6 | 1.2s | el cierre respira antes del corte |

**Total: 70-85s.** Si pasa de 85s, acorta el acto más largo y regenera. Si baja
de 68s, alarga los actos 3 y 4 — nunca el 1 ni el 5, que dependen de ser breves.

### 7.3. Cama musical

Reusa las camas existentes del canal (`bed_0..4_norm.mp3`), con
la misma rotación por día que el noticiero y el deep dive:

```bash
IDX=$(( 10#$(date +%j) % 5 ))
```

La diferencia está en el **volumen: −17dB**, más bajo que los otros formatos
(−14dB). El silencio entre frases es parte del tono editorial; la cama sostiene,
no acompaña. Fade in 1.2s, fade out 2s.

La rotación de cama es la única variación permitida entre videos de este canal
— todo lo visual permanece fijo. La música cambia lo justo para que el archivo
no se sienta idéntico al oído, sin tocar la identidad visual.

```bash
DUR_TOTAL=$(ffprobe -v error -show_entries format=duration -of csv=p=0 video_mudo.mp4)
FADE_OUT=$(python3 -c "print(round($DUR_TOTAL-2,3))")
ffmpeg -y -i video_mudo.mp4 -stream_loop -1 -i "audio/bed_${IDX}_norm.mp3" \
  -filter_complex "[1:a]atrim=0:$DUR_TOTAL,volume=-17dB,afade=t=in:st=0:d=1.2,afade=t=out:st=$FADE_OUT:d=2[m];[0:a][m]amix=inputs=2:duration=first:normalize=0[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -movflags +faststart "$NOMBRE"
```

---

## 8. Ensamblaje

**Dos capas por acto, siempre.** `bg_N.jpg` (foto tratada o papel+gráfico) se
anima con Ken Burns; `over_N.png` (solo texto, fondo transparente) va
**fijo encima**. El error clásico es aplicar el zoom al conjunto fusionado — el
texto tiembla y el video se ve amateur.

**Ken Burns suave: intensidad 0.10.** Mucho más contenido que el deep dive
(0.18). El movimiento agresivo pelea con el registro editorial.

Efecto por acto (fijo): 1 zoom-in · 2 estático · 3 estático · 4 zoom-in ·
5 zoom-out · 6 zoom-out lento.

Los actos 2 y 3 no se animan: el 2 porque la foto es solo una banda, el 3 porque
un gráfico en movimiento es ilegible.

```bash
FPS=30
FRAMES=$(python3 -c "print(round($DUR_N * $FPS))")

ffmpeg -y -loop 1 -framerate $FPS -t "$DUR_N" -i bg_N.jpg \
       -loop 1 -framerate $FPS -t "$DUR_N" -i over_N.png \
       -i voz_N.mp3 \
  -filter_complex "\
    [0:v]scale=2160:3840,zoompan=z='min(1+0.10*on/$FRAMES\,1.10)':d=1:\
x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1080x1920:fps=$FPS,setsar=1[bg];\
    [bg][1:v]overlay=0:0[v]" \
  -map "[v]" -map 2:a -frames:v $FRAMES \
  -c:v libx264 -profile:v high -preset medium -crf 20 -pix_fmt yuv420p \
  -r $FPS -g 60 -c:a aac -b:a 192k -ar 48000 \
  -movflags +faststart clip_N.mp4
```

Para actos estáticos (2 y 3), omite el `zoompan` y usa `scale=1080:1920` directo.

**Letterbox:** todo shot lleva
`force_original_aspect_ratio=increase,crop=1080:1920`. Sin excepción.

Concatena con `-f concat -c copy` → `video_mudo.mp4`. Luego la cama (sección 7).

**Nombre final:** `why_NNN_YYYYMMDD.mp4` (NNN = id de la idea).

---

## 9. GATE de render

Antes de subir, verifica en un solo script y **aborta si algo falla**:

```python
"""Pre-upload validation. Halts the pipeline on any failure."""
import subprocess, sys
from PIL import Image, ImageStat

def probe(path, key):
    return subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", key, "-of", "csv=p=0", path],
        capture_output=True, text=True).stdout.strip()

fails = []

# 1. Duración total en rango
dur = float(probe(NOMBRE, "format=duration"))
if not 68 <= dur <= 85:
    fails.append(f"duration {dur:.1f}s outside 68-85s")

# 2. Resolución exacta
res = probe(NOMBRE, "stream=width,height").replace("\n", "x")
if res != "1080x1920":
    fails.append(f"resolution {res} != 1080x1920")

# 3. Pista de audio presente y no silenciosa
if not probe(NOMBRE, "stream=codec_type").count("audio"):
    fails.append("no audio stream")

# 4. Cada overlay tiene píxeles no transparentes (el texto se renderizó)
for n in range(1, 7):
    im = Image.open(f"over_{n}.png").convert("RGBA")
    if im.getextrema()[3][1] == 0:
        fails.append(f"over_{n}.png is fully transparent — text failed to render")

# 5. Contraste real texto/fondo en cada frame — el check que atrapa el fallo
#    más caro: texto ilegible. Compara el color del texto contra la media
#    de la región que ocupa en el frame YA compuesto.
for n in range(1, 7):
    frame = Image.open(f"frame_{n}.png").convert("RGB")
    region = frame.crop((82, 380, 998, 1500))
    mean = tuple(int(v) for v in ImageStat.Stat(region).mean)
    ratio = contrast_ratio(TEXT_COLOR[n], mean)
    if ratio < 4.5:
        fails.append(f"act {n}: contrast {ratio:.1f}:1 below 4.5:1 minimum")

# 6. Ninguna banda sólida al pie (regresión de la v1)
for n in range(1, 7):
    frame = Image.open(f"frame_{n}.png").convert("RGB")
    strip = frame.crop((0, 1700, 1080, 1900)).convert("L")
    st = ImageStat.Stat(strip)
    if st.mean[0] < 25 and st.stddev[0] < 6:
        fails.append(f"act {n}: solid dark band at bottom — banda eliminada en v2")

# 7. Llenado y respeto de zonas seguras
for n in range(1, 7):
    ov = Image.open(f"over_{n}.png").convert("RGBA")
    alpha = ov.crop((82, 260, 998, 1400)).getchannel("A")
    rows = [y for y in range(alpha.height)
            if alpha.crop((0, y, alpha.width, y + 1)).getextrema()[1] > 0]
    if rows:
        fill = (rows[-1] - rows[0]) / alpha.height
        if not 0.70 <= fill <= 0.90:
            fails.append(f"act {n}: vertical fill {fill:.0%} outside 70-90%")

# 8. El badge es idéntico en los seis frames (bug de la v2: usaba el nº de acto)
badges = {badge_text(n) for n in range(1, 7)}
if len(badges) != 1:
    fails.append(f"badge differs across acts: {sorted(badges)} — debe ser el id de la idea")

# 9. Solo el acto 3 tiene fondo de papel
for n in range(1, 7):
    frame = Image.open(f"frame_{n}.png").convert("RGB")
    corner = ImageStat.Stat(frame.crop((0, 300, 200, 500)).convert("L")).mean[0]
    is_paper = corner > 215
    if is_paper and n != 3:
        fails.append(f"act {n}: paper background — solo el acto 3 lo lleva")
    if not is_paper and n == 3:
        fails.append("act 3: debe tener fondo de papel para el gráfico")

# 10. Nada de texto en la zona muerta (y > 1400) ni bajo el riel de botones
for n in range(1, 7):
    ov = Image.open(f"over_{n}.png").convert("RGBA")
    dead = ov.crop((0, 1400, 1080, 1920)).getchannel("A")
    if dead.getextrema()[1] > 0:
        fails.append(f"act {n}: texto en la zona muerta (y>1400)")
    rail = ov.crop((860, 880, 1080, 1400)).getchannel("A")
    if rail.getextrema()[1] > 0:
        fails.append(f"act {n}: texto bajo el riel de botones (x>860, y 880-1400)")

# 11. Medida de línea — ninguna línea supera su máximo de caracteres
for n, lines in RENDERED_LINES.items():          # dict acto -> {rol: [str, ...]}
    for rol, ls in lines.items():
        cap = {"q": 20, "myth": 26, "support": 34, "caption": 38, "cite": 42}[rol]
        for line in ls:
            if len(line) > cap:
                fails.append(f"act {n} {rol}: linea de {len(line)} chars (max {cap})")

if fails:
    print("RENDER_GATE=FAILED")
    for f in fails:
        print(f"  - {f}")
    sys.exit(1)
print("RENDER_GATE=OK")
```

**Si el gate falla, no subas.** Arregla y vuelve a correr, o entrega el mp4
diciendo exactamente qué check falló. Nunca publiques saltando el gate.

---

## 10. SEO

### `titulo_video`

**La pregunta literal, sin adornos, más dos hashtags.** No reformules. La
pregunta es la query que va a traer views durante años; cambiarla por algo
"más atractivo" destruye el match de búsqueda.

```
Why does time speed up as you age? #psychology #science
```

Reglas:
- Máximo 90 caracteres con los hashtags.
- Dos hashtags máximo (tres o más los ignora YouTube).
- Sin mayúsculas sostenidas, sin "!", sin "you won't believe".
- Sin sufijos de relleno tipo "· explained".

### `descripcion`

Primeras dos líneas con el gancho y el veredicto (es lo visible antes del
"…more"). Luego: resumen en 3-4 líneas, **las citas académicas completas con
DOI**, créditos de Pexels, y 3-5 hashtags.

Las citas en la descripción no son adorno — son la prueba pública de que el
canal no inventa. Fórmalas así:

```
Source: Friedman, W. J., & Janssen, S. M. J. (2010). Aging and the speed of
time. Acta Psychologica, 134(2), 130-141. doi:10.1016/j.actpsy.2010.01.004
```

### `tags`

Llena hasta ~490 de los 500 caracteres.

```python
TAGS_BASE = [
    "psychology", "human behavior", "social science", "why do we",
    "psychology facts", "behavioral science", "science explained",
    "cognitive science", "sociology", "research",
]
```

Luego dinámicos del tema concreto: el fenómeno, el nombre del efecto, términos
long-tail que alguien buscaría. Mide de verdad:
`sum(len(t) for t in tags) + len(tags) - 1`.

---

## 11. Subtítulos (en/es/fr)

Tres pistas por video. Se activan solo si el viewer prende CC, pero **indexan
en los tres idiomas** — en un canal cuyo tráfico viene de búsqueda, eso importa
más que en un formato de feed.

### 11.1. Requisito de scope

El `YT_REFRESH_TOKEN` necesita `https://www.googleapis.com/auth/youtube.force-ssl`
además de `youtube.upload`. Sin ese scope, `captions().insert` falla con
`insufficientPermissions`. Si falta, sáltate la sección y avísalo en la entrega
— no abortes el video por esto.

### 11.2. `subs_en.srt`

edge-tts ya generó un `sub_N.srt` por acto en la sección 7. Cada uno arranca en
00:00:00, así que hay que concatenarlos aplicando el **offset acumulado** de las
duraciones reales medidas con ffprobe (`DUR_N`), no de las estimadas.

Usa la misma función `parse_srt` + `_fmt` del deep dive, iterando actos 1..6
(no 1..5).

**Cuidado con los pads diferenciados.** Los actos 1, 5 y 6 llevan pads más
largos (0.9s, 0.9s, 1.2s). El offset debe usar `DUR_N` medido *después* del pad,
o los subtítulos se desfasan acumulativamente y el error llega a ~2s al final.

### 11.3. `subs_es.srt` y `subs_fr.srt`

Traduce tú mismo, cue por cue, **sin llamada externa y sin costo**. Reglas:

- **Timestamps idénticos.** Nunca los modifiques: el audio y el video son los
  mismos en las tres pistas.
- **Nunca traduzcas fuentes académicas.** `Acta Psychologica` se queda igual en
  las tres pistas. Igual los apellidos de autores, nombres de revistas, DOIs y
  nombres de efectos con término técnico establecido.
- **Términos con traducción canónica sí se traducen:** `time pressure` →
  "presión de tiempo" / "pression temporelle"; `assortative mating` →
  "emparejamiento selectivo" / "appariement assortatif". Si dudas, busca cómo lo
  nombra la literatura académica en ese idioma, no cómo suena mejor.
- **Números en formato local:** `1,865` (en) → `1.865` (es) → `1 865` (fr).
  Decimales: `0.79` (en) → `0,79` (es/fr).
- **Registro editorial, no doblaje.** Debe leerse como prosa de revista seria en
  ese idioma. Traducción natural, no literal.
- **Velocidad de lectura.** Español y francés corren ~20% más largos que el
  inglés. Si un cue queda denso, acorta sin cambiar el sentido. **Nunca partas un
  cue en dos** — rompe la sincronía.

### 11.4. Por qué tres idiomas y no más

Inglés es el audio. Español y francés son los dos mercados donde el nicho de
divulgación psicológica tiene demanda alta y competencia baja, y son los dos
idiomas de doblaje ya contemplados para los otros canales — mismo esfuerzo de
traducción, infraestructura compartida. Añadir más idiomas antes de validar
retención es trabajo sin señal.

---

## 12. Subida

### 12.1. Modo piloto (actual)

Mientras el canal "Why" no exista como canal propio, **los videos se suben al
canal de AI Daily News** con las mismas credenciales del `.env`
(`YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`), y **quedan privados
de forma permanente** — sin `publishAt`, sin programación.

Esto es deliberado: el objetivo del piloto es validar render, tono y ritmo sin
contaminar el feed ni el historial de recomendaciones de un canal que ya tiene
audiencia de otro nicho. Un video de psicología publicado en un canal de noticias
de IA le enseña algoritmos equivocados a ambos.

```python
body = {
    "snippet": {
        "title": meta["titulo_video"],
        "description": meta["descripcion"],
        "tags": meta["tags"],
        "categoryId": "27",            # Education
    },
    "status": {
        "privacyStatus": "private",    # permanente, NO temporal
        "selfDeclaredMadeForKids": False,
        "containsSyntheticMedia": False,
    },
}
resp = yt.videos().insert(
    part="snippet,status", body=body,
    media_body=MediaFileUpload(os.environ["NOMBRE"], resumable=True)).execute()
vid = resp["id"]
print(f"VIDEO_URL=https://www.youtube.com/watch?v={vid}")
print(f"STUDIO_URL=https://studio.youtube.com/video/{vid}/edit")
print("MODO=piloto privado en canal AI Daily News")
```

**No pases `publishAt` en modo piloto.** Si lo pasas, YouTube programa la
publicación y el video se hace público solo — exactamente lo que no queremos.

`containsSyntheticMedia: False` — voz sintética sobre stock real no requiere
divulgación; no hay material realista alterado.

### 12.2. Modo canal propio (cuando exista)

Cuando el canal "Why" tenga sus propias credenciales, cambia dos cosas y nada más:

- `privacyStatus: "private"` + `publishAt` a las **7:00am hora de Nueva York**.
  El público de este formato consume en el commute matinal, no de madrugada.
- Variables de entorno `YT_WHY_CLIENT_ID` / `YT_WHY_CLIENT_SECRET` /
  `YT_WHY_REFRESH_TOKEN` en lugar de las genéricas.

```python
NY = ZoneInfo("America/New_York")
ahora = datetime.now(NY)
objetivo = ahora.replace(hour=7, minute=0, second=0, microsecond=0)
if objetivo <= ahora:
    objetivo += timedelta(days=1)
publish_at = objetivo.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
```

### 12.3. Captions

Sube los tres tracks con `captions().insert` (ver sección 11). **Funcionan
igual en modo piloto:** un video privado acepta captions, y así se valida el
pipeline completo antes de tener canal propio.

---

## 13. Historial

Tras la subida exitosa, actualiza dos archivos en
`anmalaver/AI-Daily-news-assets`. **El orden importa:** escribe primero el
historial, después la cola.

El historial es lo que la rutina de mañana consulta para no repetir (sección
2.2.1). Si solo uno de los dos pushes alcanza a pasar, tiene que ser ese.

### 13.1. `topics-history/why-history.json` (primero)

Inserta la entrada al inicio, filtra duplicados por `slug`, trunca a 20. Misma
mecánica de `sha` + PUT que el deep dive.

```json
{
  "fecha": "2026-10-07",
  "id": "why-0007",
  "slug": "time-speeds-up-with-age",
  "question": "Why does time speed up as you age?",
  "verdict": "It's not your age. It's your calendar.",
  "source_doi": "10.1016/j.actpsy.2010.01.004",
  "video_url": "https://www.youtube.com/watch?v=..."
}
```

La inserción es idempotente: filtrar por `slug` antes de insertar significa que
correr la rutina dos veces sobre el mismo tema no ensucia el archivo.

### 13.2. `topics-history/why-queue.json` (después)

Marca la idea usada como `"status": "published"` y añade `"video_url"`.

### 13.3. Si algo falla

**No bajes el video.** Ya está subido y en modo piloto está privado, así que no
hay urgencia. Reporta en la entrega:

- Cuál de los dos pushes falló y con qué error.
- Las entradas JSON completas para pegar a mano.

Un fallo en el push de la cola es recuperable solo: el cruce contra el historial
de la sección 2.2.1 atrapa la idea mañana y la salta, reportando `QUEUE_DESYNC`.
Un fallo en el push del historial es el que sí puede producir un duplicado — si
ese falla, dilo de forma destacada en la entrega.

---

## 14. Entrega

1. **La URL del video** al principio, y la de Studio.
2. El mp4 y el `manifiesto.json`.
3. **Los cuatro cortes del GATE de evidencia** con su veredicto — es lo que
   permite auditar que el canal no está publicando pop-psychology.
4. Las fuentes con DOI.
5. Resultado del `RENDER_GATE`.
6. Estado de captions y de los dos archivos de historial. Si el push del
   **historial** falló, dilo de forma destacada — es el que puede causar un
   duplicado mañana.
7. **`QUEUE_DESYNC`**: los slugs que estaban `ready` en la cola pero ya
   aparecían en el historial. Cada uno es una idea publicada cuyo `status` no se
   actualizó; hay que corregirla a mano en el repo.
8. Créditos de Pexels por acto.

---

## 15. Reglas de fallo

- GATE de evidencia falla → descarta la pregunta, toma la siguiente de la cola.
  Si la cola se agota, aborta y avisa. **Nunca publiques sin respaldo.**
- GATE de render falla → no subas. Arregla o entrega el mp4 explicando el check.
- Duotono sale gris → el midtone no se aplicó. Revisa antes de seguir.
- Duración fuera de 68-85s → ajusta actos 3 y 4, nunca el 1 o el 5.
- Fuente no verificable en fuente primaria → no la uses.
- Subida falla → reporta el error exacto y deja el mp4. `invalid_grant` significa
  token expirado: avísame.
- Push de historial falla → el video ya está publicado, no lo bajes. Pega el JSON.
  **Márcalo destacado en la entrega**: sin esa entrada, la corrida de mañana puede
  producir el mismo tema otra vez.
- Push de cola falla → recuperable solo. El cruce contra historial (2.2.1) la
  salta mañana y reporta `QUEUE_DESYNC`. Pega el JSON igual.
- Un slug aparece en cola como `ready` y también en el historial → sáltalo
  siempre. Prefiere no publicar hoy antes que publicar un duplicado: la política
  de contenido inauténtico penaliza a nivel de canal, no de video.
- Algo irrecuperable → entrega lo que alcanzaste y di en qué sección paraste.

---

## 16. Reporte de ejecución

Además de la entrega, dame:

- **Número total de llamadas a herramientas** (bash, python, ffmpeg, ffprobe,
  descargas, búsquedas web).
- **Las 3 fases que más llamadas consumieron.**
- **Cuántas preguntas descartó el GATE de evidencia** antes de encontrar una
  viable, y en qué corte cayó cada una.
- Si tuviste que **re-renderizar** algún acto y por qué.

Esto es instrumentación para optimizar el harness. No cambia el video.

---

## Anexo A — Diferencias con el deep dive

Para quien venga del otro prompt y asuma continuidad:

| | Deep dive | Why |
|---|---|---|
| Actos | 5 | 6 |
| Duración | ~90s | **70-85s** |
| Paleta | rotativa (5 pares) | **fija, una sola** |
| Fondo | oscuro | **papel casi blanco** |
| Fuentes | Space Grotesk + Barlow | **Fraunces + IBM Plex Mono** |
| Ken Burns | 0.18 | **0.10** |
| Voz rate | +13% | **+8%** |
| Cama musical | −14dB | **−17dB** (mismas camas rotativas) |
| Category ID | 28 (Sci & Tech) | **27 (Education)** |
| Publicación | 4:00am NY | **7:00am NY** |
| Disparador | noticia del día | **cola pre-investigada** |
| Gancho | novedad | **contraintuición** |
| Vida útil | días | **años** |

## Anexo B — Por qué el harness está escrito así

Notas de diseño, para quien lo edite después.

**Las constantes visuales están congeladas** porque la variación entre videos no
aporta identidad, la destruye. El deep dive rota paleta para que cada video se
sienta fresco; este canal hace lo contrario — la repetición exacta es lo que hace
que el espectador reconozca un video del canal en medio del scroll.

**Los gates existen porque el modo de fallo de este formato es silencioso.** Un
deep dive con un dato flojo se nota. Un video de psicología con un hallazgo no
replicado se ve idéntico a uno sólido, y el daño reputacional llega meses
después. Por eso el corte de replicación es explícito y nombra los efectos
prohibidos: es más barato codificar la lista que confiar en que el modelo la
recuerde.

**La rotación de layout está fijada en tabla** y no se deja a criterio, porque
es la defensa concreta contra la política de contenido inauténtico de YouTube
(jul-2025): seis frames idénticos con distinto texto es exactamente el patrón
que desmonetiza. La rotación tiene que ser estructural, no opcional.

**Los scripts van completos en el prompt** en lugar de descritos, porque cada
ida y vuelta de "escribe el script / córrelo / corrige" cuesta tokens y
reintroduce variabilidad. El modelo que ejecuta no debe estar diseñando el
duotono; debe estar corriéndolo.

**El título es la pregunta literal** porque este canal no compite por el feed,
compite por la búsqueda. Un título ingenioso gana el scroll de hoy; la pregunta
exacta gana la query de los próximos tres años.


---

## Anexo C — Changelog

### v2 (2026-09-19) — calibración tras la primera corrida real

La corrida `why_live0919` produjo un video correcto en estructura, audio y
ritmo, pero con dos fallos visuales. Ambos eran fallos **de este documento**, no
del modelo que lo ejecutó:

**1. Banda inferior negra — eliminada.**
La v1 pedía una franja `INK` sólida de 365px al pie de todos los frames, para
que la UI blanca de Shorts se distinguiera sobre papel casi blanco. El problema
era real; la solución era desproporcionada: 19% del frame en negro plano,
cortando la foto en seco y sin llevar información. Sustituida por la zona de
reposo (6.3): 420px sin texto, con el scrim de la foto o un degradado cálido en
el frame de papel. El identificador `why · #NNN` subió junto a la etiqueta de
acto.

**2. Contraste insuficiente — causa raíz identificada.**
La v1 definía los colores por rol semántico ("PUNCH: pregunta, comillas") sin
decir nunca qué color de texto corresponde a qué tipo de fondo. El modelo tuvo
que inferirlo y lo resolvió distinto en cada acto: la pregunta completa en
`PUNCH` sobre una foto rojiza (ilegible), y el matiz en `INK` sobre una foto
oscura (ilegible). Peor, el documento literalmente listaba "pregunta" como uso
de `PUNCH`, así que el modelo hizo lo que decía.

Correcciones:
- Tabla obligatoria 6.3.1 que vincula color de texto a tipo de fondo. El fondo
  manda sobre el rol.
- `PUNCH` y `VERDICT` restringidos a signos de puntuación aislados y líneas.
  Nunca una palabra, nunca una frase.
- Scrim adaptativo medido (6.3.2) en vez de opacidad fija, con descarte de la
  foto si ni al máximo alcanza el contraste.
- Sombra obligatoria en todo texto sobre foto.
- Check de contraste WCAG ≥4.5:1 por frame en el gate de render, que aborta.

**3. Costura del gráfico.** El PNG de matplotlib se guardaba con fondo opaco y
su `#FBFAF5` no coincidía exactamente con el del canvas, dejando un rectángulo
gris visible. Ahora `transparent=True`.

**4. Llenado vertical.** El acto 3 dejaba ~400px muertos entre el pie del
gráfico y la cita. Añadida la regla 6.3.3 (60-85% de la zona de acción) y su
check en el gate.

### Lección para futuras ediciones

Los tres fallos comparten forma: **el documento especificaba el *qué* sin
especificar el *cuándo*.** "PUNCH es para la pregunta" es una regla de rol;
"sobre foto el texto es PAPER" es una regla de contexto. Un harness para un
modelo ejecutor necesita las dos, y cuando falta la de contexto el modelo
rellena el hueco con una decisión razonable que se ve mal.

Al agregar cualquier regla nueva a este prompt, escríbela como binding
condicional —  *si el fondo es X, entonces Y* — y no como atributo suelto.

### v3 (2026-09-20) — segunda calibración

La corrida `why_003` resolvió el contraste: el texto crema sobre foto con scrim
adaptativo funcionó bien en los cinco frames con imagen. Los fallos restantes
eran de composición y densidad.

**1. Etiquetas de acto — eliminadas.** `the question`, `the myth`, `the data`
arriba a la izquierda. No informaban nada: el espectador no necesita saber en
qué parte de la estructura va, y ocupaban la franja superior de la zona de
acción.

**2. Fondos de papel fuera de lugar.** El layout `photo_band` del acto 2 (foto
al 34% arriba, texto sobre blanco abajo) partía el frame en dos y leía como
diapositiva. Eliminado. Ahora **solo el acto 3 tiene papel**, porque un gráfico
lo necesita; los otros cinco van a foto a sangre. La variación entre frames la
da ahora la posición del texto (centrado / izquierda / dos bloques / espejo),
no el tipo de fondo.

**3. Densidad de lectura.** Cada frame tenía una sola frase corta y mucho aire:
el espectador terminaba de leer en 2s sobre un frame de 8s, y ese hueco es donde
se pierde retención. Añadida la **línea de apoyo** (sección 4.0): 15-22 palabras
en Fraunces italic 44px bajo el statement, en los actos 2-5. No se narra — la voz
lleva el guion y los ojos leen otra cosa.

**4. Tipografía de gráfico ilegible.** La cita académica a 22px y el caption a
26px sobre un canvas de 1920 no se leen en un teléfono. Subidos a 30px y 34px, y
`font.size` de matplotlib de 20 a 30. Regla nueva: **nada por debajo de 30px**.
Añadidos valores sobre las barras y etiquetas de eje X obligatorias — un gráfico
de barras sin saber qué mide cada barra no informa.

**5. Bug del identificador.** `why · #NNN` se imprimía con el número de acto, así
que un mismo video mostraba `#001`, `#002`, `#003` y `#004` en frames distintos.
Debe ser el id de la idea, constante en los seis. Añadido check en el gate.

**6. Franja gris reintroducida.** El degradado de la zona de reposo se aplicaba
también sobre frames con foto, encima del scrim, produciendo una banda con borde
duro visible — la misma banda que la v2 había eliminado, por otra vía. Ahora el
degradado es exclusivo del frame de papel.

**7. Llenado vertical.** Subido de 60-85% a 70-90%, y añadida la tabla de
composición del acto 3, que en `why_003` dejó ~650px muertos entre el gráfico y
los bordes de la zona de acción.

### v4 (2026-09-20) — zonas seguras reales y profundidad de contenido

Calibrado contra una captura del reproductor de Shorts, no contra el mp4 suelto.
Ver el video fuera del reproductor oculta el fallo más caro.

**1. Las zonas seguras estaban mal medidas.** El prompt asumía que la UI de
YouTube solo comía los últimos 420px. En realidad:
- El **riel de botones** (me gusta, comentar, guardar, compartir) ocupa desde
  x≈905 en toda la franja y 880→1400. En `why_003` el caption y la línea de
  apoyo se extendían hasta x=998 y quedaron literalmente debajo de "Me gusta" y
  "Guardar" — texto que nadie puede leer.
- La fila con el nombre del canal y el título empieza en y≈1426, no en 1500.

Nueva geometría en dos zonas: **A** (ancho completo, y 260→880) para el
statement y el gráfico; **B** (x hasta 860, y 880→1400) para apoyo, caption y
cita; y zona muerta de 520px. Dos checks nuevos en el gate abortan si hay
píxeles de texto en cualquiera de las dos regiones prohibidas.

**2. Comprimir en vez de encoger.** Al reducirse el ancho útil, la tentación es
bajar la fuente — que es exactamente lo que rompió la legibilidad en la v2.
Regla nueva (6.3.-1): **el tamaño no se toca; se acorta la medida de línea y se
añaden líneas.** Statement a 20 caracteres por línea, apoyo a 34, con
interlineados más cerrados. Un statement de 4 líneas cortas se lee más rápido
que uno de 2 largas y llena mejor la zona A.

**3. El video no enseñaba lo suficiente.** A 50-60s el formato se veía bien pero
se terminaba sin haber explicado nada: el espectador salía sabiendo que su
intuición estaba mal, sin entender por qué. Tres cambios:

- **Duración a 70-85s** (125-165 palabras de narración).
- **El acto 4 pasó de "el matiz" a "el mecanismo"**: cómo funciona realmente la
  cosa, con detalle suficiente para que el espectador pueda explicárselo a
  alguien al día siguiente. Es el acto que justifica el video.
- **El acto 5 gana una consecuencia** en su línea de apoyo: qué cambia si esto
  es cierto.

**4. El hook era débil.** El acto 1 mostraba la pregunta literal, que no genera
tensión — el espectador ya sabe que el video va a responderla. Ahora el
statement es **el conflicto** ("Everyone believes this. 79,000 couples say
otherwise.") y la pregunta literal baja a la línea de apoyo. El título del video
sigue siendo la pregunta textual, porque ahí sí es la query de búsqueda.

**5. El identificador se movió al pie de la zona B.** Arriba a la derecha
quedaba pisado por la hora y la batería del teléfono.

### Lección acumulada

Las tres calibraciones (v2, v3, v4) fallaron en lo mismo: **especificar el
artefacto sin especificar el contexto donde se consume.** v2 definía colores sin
decir sobre qué fondo; v3 definía una zona de reposo sin medir la UI real; v4
corrige ambas midiendo una captura del reproductor.

Para la próxima: **cualquier regla de layout debe validarse contra una captura
del reproductor de Shorts con la UI encima, no contra el mp4 abierto en un
visor.** El mp4 se ve perfecto en los dos casos donde el video real falla.
