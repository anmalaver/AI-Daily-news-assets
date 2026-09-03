# Rutina — Deep Dive de IA (una noticia, 5 actos)

Produce **un video vertical en inglés** (~90 segundos) que explica **en
profundidad la noticia de IA más importante** del momento, en 5 actos: qué pasó,
por qué importa, el detalle clave, las implicaciones, y qué viene. Al terminar,
**súbelo a YouTube** y repórtame la URL.

Es un formato hermano del noticiero, pero distinto: el noticiero da 5 noticias
superficiales; esto toma **una sola** y la desarrolla. Reusa la misma
infraestructura (setup, Pexels, edge-tts, Ken Burns, tarjetas en dos capas,
cama musical, subida), con el mismo estilo visual del canal salvo la tipografía
de titulares, que cambia para señalar que es un formato "profundo".

Es **independiente del noticiero**: busca su propia noticia. Puede coincidir con
una de las 5 del día; no importa, el tratamiento es completamente distinto.

---

## 0. Setup

Corre `setup.sh` (compartido por las tres rutinas). Ya incluye **Space Grotesk**
para los titulares de este formato, además de todo lo habitual.

Carga `.env`: necesita `PEXELS_API_KEY` y las credenciales de YouTube
(`YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`, `YT_PRIVACY_STATUS`).

Paleta del día como el noticiero: `IDX=$(( 10#$(date +%j) % 5 ))`, misma tabla
de 5 pares BASE/ACCENT.

---

## 1. Selección de la noticia

Busca en la web la noticia de IA **más importante** de las últimas 24-48h. Una
sola, la de mayor peso: un lanzamiento grande, un acuerdo de miles de millones,
un avance técnico real, un movimiento regulatorio de fondo. Debe tener
suficiente sustancia para 5 actos de desarrollo — descarta lo anecdótico.

Criterios: hecho verificable y de fuente primaria; relevancia amplia (no solo
para investigadores); densidad suficiente para explicar causas e implicaciones.

Muéstrame la noticia elegida con su fuente antes de escribir. Sigue sin esperar
confirmación.

---

## 2. Verificación de hechos (ANTES de escribir)

Profundizar significa afirmar más: causas, implicaciones, quién gana y pierde.
Eso exige rigor.

1. Reúne los hechos verificables de fuentes primarias (blog oficial, paper,
   comunicado, medio establecido). Guárdalos con sus URLs.
2. **Separa hecho de análisis.** "Pasó X" es hecho; "X hará que Y" es análisis y
   debe presentarse como tal ("analysts expect…", "this could mean…"), nunca
   como certeza.
3. **Vigila la causalidad.** El acto de implicaciones es el más resbaladizo: no
   conectes hechos con relaciones causa-efecto que las fuentes no respaldan.
4. Las cifras, exactas y atribuidas. Si una fuente dice "hasta $500B", no lo
   conviertas en "$500B" a secas.

Repórtame los hechos con fuentes al final, para verificar.

---

## 3. Guion — 5 actos (arco explicativo)

Redacta primero la historia completa, luego trocéala en 5 actos. El arco:

| Acto | Función | Etiqueta |
|---|---|---|
| 1 | **Qué pasó** — el hecho, directo y claro | THE NEWS |
| 2 | **Por qué importa** — el contexto que lo hace grande | WHY IT MATTERS |
| 3 | **El detalle clave** — el dato o mecanismo que pocos notaron | THE DETAIL |
| 4 | **Implicaciones** — quién gana, quién pierde, qué cambia | WHAT IT MEANS |
| 5 | **Qué viene** — el futuro cercano y la pregunta abierta | WHAT'S NEXT |

Cada acto en el manifiesto:

```json
{
  "fecha": "<FECHA>",
  "paleta": { "base": "...", "accent": "..." },
  "voz": "en-US-AvaMultilingualNeural",
  "titulo_video": "...",
  "descripcion": "...",
  "tags": ["..."],
  "actos": [
    {
      "n": 1,
      "etiqueta": "THE NEWS",
      "titulo": ["Nvidia Backs", "OpenAI's $500B", "Ohio Buildout"],
      "subtitulo": "A quarter-trillion-dollar guarantee",
      "cuerpo": [
        "Nvidia is weighing a $250 billion",
        "guarantee for OpenAI's lease of a",
        "10-gigawatt data center. The total",
        "buildout could top $500 billion."
      ],
      "guion": "Narración en voz alta de este acto. Inglés. 30-38 palabras.",
      "conceptos": ["data center servers", "server rack close up", "abstract blue network"]
    }
  ]
}
```

**Límites (por acto):**

| Campo | Límite |
|---|---|
| `etiqueta` | fija por acto (ver tabla) + contador `· N/5` se añade al render |
| `titulo` | 2-3 líneas, **máx 18 caracteres por línea** (Space Grotesk es más ancha) |
| `subtitulo` | máx 42 caracteres, una frase con gancho |
| `cuerpo` | 4-5 líneas, **máx 42 caracteres por línea** (a 46px; 5 líneas es el máximo absoluto — termina en y≈1310, muy cerca de la safe zone 1320) |
| `guion` | 30-38 palabras (más largo que el noticiero: es explicativo) |
| total narración | ~180 palabras (≈90s) |

**Tono:** explicativo, claro, con autoridad. Como un buen analista que te
explica algo importante sin condescendencia. El acto 3 (el detalle) es el gancho
de "esto no lo viste en otros lados".

**El cierre del acto 5 debe ser filoso, no genérico.** Evita preguntas retóricas
vagas ("Will Washington act in time?") que podrían cerrar cualquier video. En su
lugar, cierra con **un dato que reencuadra** — una afirmación verificable, ya
mencionada o derivable de los hechos del video, que deje al espectador viéndolo
distinto. Tres formas que funcionan:

- **El dato que pone escala:** una cifra o hecho real que dimensione lo que
  viene ("that single training run will cost more than [algo conocido]").
- **El contraste temporal:** dónde estaba esto antes vs dónde va ("five years
  ago this was research; the next version ships this year").
- **La tensión sin resolver:** nombrar el conflicto real que queda abierto, no
  preguntarlo ("the technology is ready; the rules aren't — that's the story").

Regla dura: el cierre es una afirmación **anclada en un hecho del video**, no una
especulación nueva ni una predicción que las fuentes no respalden. Reencuadra lo
ya dicho; no adivines el futuro. Puede terminar invitando a comentar, pero desde
un dato, no desde una pregunta vacía.

Los saltos de línea de `titulo` y `cuerpo` los defines tú (SVG no hace wrap).
Escapa `&`, `<`, `>`.

### `titulo_video` y `descripcion` (SEO)

**`titulo_video` (máx 90 caracteres, idealmente 60-75 con los hashtags).** Es lo
que decide el clic y lo que indexa el buscador. Estructura de **dos partes**:

**Parte 1 — título legible con las dos marcas más importantes.** Una frase
natural y clara sobre la noticia que incluya los **dos nombres propios de mayor
arrastre** (empresas, modelos o productos: OpenAI, Nvidia, Google, Anthropic,
Sony, etc.). Legible para un humano, con gancho de curiosidad sin clickbait.

**Parte 2 — los dos tags más importantes, como hashtags al final.** Después del
título, anexa **dos hashtags** con los tags de mayor peso del video (pueden ser
marcas o términos de tema — lo que más se busque). YouTube los hace clicables y
cuentan para descubrimiento.

Formato, siguiendo el patrón `Título con dos marcas #Tag1 #Tag2`:
- "Nvidia bets $250B on OpenAI — the biggest AI deal yet #Nvidia #OpenAI"
- "Anthropic vs Sony: the AI music lawsuit explained #Anthropic #AImusic"
- "Google's Gemini 3 just shifted the AI race #Gemini #AI"

Reglas:
- Las dos marcas van en la **parte legible**, al principio o bien integradas.
- Los dos hashtags son los **tags más fuertes** (de `TAGS_BASE` o los dinámicos
  del día, los de más búsqueda). Sin espacios dentro del hashtag (`#AImusic`, no
  `#AI music`). No más de dos — tres o más hashtags YouTube los ignora.
- Vigila el límite de 90 caracteres contando los hashtags.
- Prohibido: TODO EN MAYÚSCULAS, más de un "!", promesas falsas, sufijos de
  relleno tipo "· AI news" (los hashtags reemplazan eso).

**`descripcion`.** Primeras 1-2 líneas visibles con gancho + keyword + marca
(es lo que se indexa y se lee antes del "…more"). Luego un resumen del análisis,
llamado a suscribirse ("Deep dives on the AI that matters. Subscribe."), créditos
de Pexels, y 3-5 hashtags relevantes (`#AI #artificialintelligence #technews`).

---

## 4. Fondos

Igual que el noticiero: catálogo de conceptos por tema (las 14 categorías),
User-Agent en Pexels, filtro de co-ocurrencia en `alt`, descarta generativas
salvo que no haya alternativa, una foto por acto sin repetir. Cada acto elige su
concepto según su contenido (el acto de implicaciones regulatorias usa
gobierno/mercados; el técnico usa hardware/infra, etc.). Descarga
`fit=crop&w=1080&h=1920`. Guarda `photographer` para créditos. Reporta solo lo
descartado.

---

## 5. Audio

**Voz:** `en-US-AvaMultilingualNeural` (femenina, la misma del noticiero, para
probar el impacto de una voz consistente en el canal), `--rate=+13%`,
`--pitch=+0Hz`. Ritmo ágil pero explicativo — este formato es de análisis, no de
suspenso.

> **Nota:** esto es una prueba. Antes la voz era `en-US-AndrewMultilingualNeural`
> (masculina). Si al comparar prefieres volver a la masculina para diferenciar el
> deep dive del noticiero, es un solo cambio.

> **Alternativas si quieres comparar de oído:** `en-US-AndrewMultilingualNeural`
> o `en-US-BrianMultilingualNeural` (masculinas) a los mismos parámetros.

Por acto: sintetiza `guion`, pad de 0.8s al final:

```bash
edge-tts --voice "$VOZ" --rate=+13% --pitch=+0Hz \
  --file acto_N.txt --write-media raw_N.mp3 --write-subtitles sub_N.srt
ffmpeg -y -i raw_N.mp3 -af "apad=pad_dur=0.8" -c:a libmp3lame -b:a 192k voz_N.mp3
DUR_N=$(ffprobe -v error -show_entries format=duration -of csv=p=0 voz_N.mp3)
```

Si un acto pasa de 16s, acorta y regenera. Total entre 80 y 95s.

**Cama musical:** la del día (`bed_(día%5)_norm.mp3`), tendida sobre el video
concatenado en una sola pasada, **a −14 dB**, con fade in 1.5s y fade out 2s.
Igual que el noticiero pero **sin cabezote de intro** (este formato no tiene
intro separada; el acto 1 abre directo). La música corre desde el inicio.

```bash
DUR_TOTAL=$(ffprobe -v error -show_entries format=duration -of csv=p=0 video_mudo.mp4)
FADE_OUT=$(python3 -c "print(round($DUR_TOTAL-2,3))")
ffmpeg -y -i video_mudo.mp4 -stream_loop -1 -i "audio/bed_${IDX}_norm.mp3" \
  -filter_complex "[1:a]atrim=0:$DUR_TOTAL,volume=-14dB,afade=t=in:st=0:d=1.5,afade=t=out:st=$FADE_OUT:d=2[m];[0:a][m]amix=inputs=2:duration=first:normalize=0[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -movflags +faststart "$NOMBRE"
```

---

## 6. Tarjetas (formato profundo, dos capas)

Como el noticiero anima el fondo, cada acto va en dos capas: `bg_N.jpg` (foto,
se anima) y `over_N.png` (overlay + texto, transparente, quieto).

**Diferencias con el noticiero:** titulares en **Space Grotesk** (no Anton), y
**más texto** — lleva subtítulo y cuerpo de apoyo además del título.

Paleta del día (rotativa, como el noticiero). Overlay `linearGradient`:
0.45 / 0.72 / 0.96.

Composición:

- **Ícono del acto + etiqueta con contador**, en la misma línea, y≈460:
  un ícono geométrico simple en color ACCENT a la izquierda (x≈112), seguido de
  `<etiqueta> · N/5` en Barlow Bold 34px ACCENT, `letter-spacing="6"`, empezando
  en x=148. El ícono da identidad visual a cada acto en 2 segundos de scroll.
  Íconos por acto (paths SVG de una sola forma, trazo de 4px, ~26px de alto —
  **no emojis**, que rsvg renderiza inconsistente):
  - Acto 1 THE NEWS → punto con dos arcos de emisión (broadcast/señal)
  - Acto 2 WHY IT MATTERS → triángulo de alerta con signo
  - Acto 3 THE DETAIL → lupa
  - Acto 4 WHAT IT MEANS → balanza
  - Acto 5 WHAT'S NEXT → flecha hacia adelante
  El ícono va en la capa `over` (transparente), alineado verticalmente con la
  etiqueta, sin invadir el margen de x=90.
- **Título:** Space Grotesk peso 700, ~104px, blanco, primera línea ~y=600,
  interlineado ~112. **Tamaño auto-calculado** midiendo el ancho real (método de
  la intro): si la línea más ancha no cabe en 900px, baja de a 4px. Máx 3 líneas.
- **Regla:** rect ACCENT 150×6px, bajo el título.
- **Subtítulo:** Barlow Bold 40px, color ACCENT, bajo la regla.
- **Cuerpo:** Barlow 46px `#D6DEE8`, **4-5 líneas**, interlineado 64, bajo el
  subtítulo (subido de 40px para mejor legibilidad y para llenar el espacio
  inferior). **5 líneas es el máximo absoluto:** termina en y≈1310, a solo ~10px
  de la safe zone (1320). Al usar 5 líneas, vigila que la última no tenga
  descendentes (j, g, p, y) ni acentos altos que empujen más abajo; si el texto
  roza el borde, usa 4 líneas.

Las posiciones se calculan según cuántas líneas tenga el título. Verifica que
todo quede entre y=250 y y=1320. La imagen va embebida base64 en `bg`; `over` es
SVG transparente.

Verifica que la capa foto no esté vacía antes de seguir.

### Paths de los íconos (referencia, color = ACCENT del día)

Usa estos grupos SVG, trasladados a la posición del ícono (x≈112, y≈455). El
`stroke` es el ACCENT del día:

```svg
<!-- 1 THE NEWS: broadcast -->
<circle cx="0" cy="0" r="8" fill="ACCENT"/>
<path d="M -18 -18 A 25 25 0 0 1 -18 18" fill="none" stroke="ACCENT" stroke-width="4"/>
<path d="M 18 -18 A 25 25 0 0 0 18 18" fill="none" stroke="ACCENT" stroke-width="4"/>
<!-- 2 WHY IT MATTERS: alerta -->
<path d="M 0 -22 L 20 18 L -20 18 Z" fill="none" stroke="ACCENT" stroke-width="4" stroke-linejoin="round"/>
<line x1="0" y1="-6" x2="0" y2="6" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<circle cx="0" cy="12" r="2.5" fill="ACCENT"/>
<!-- 3 THE DETAIL: lupa -->
<circle cx="-4" cy="-4" r="15" fill="none" stroke="ACCENT" stroke-width="4"/>
<line x1="7" y1="7" x2="20" y2="20" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<!-- 4 WHAT IT MEANS: balanza -->
<line x1="0" y1="-20" x2="0" y2="18" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<line x1="-20" y1="-14" x2="20" y2="-14" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<path d="M -20 -14 L -27 2 L -13 2 Z" fill="none" stroke="ACCENT" stroke-width="3"/>
<path d="M 20 -14 L 13 2 L 27 2 Z" fill="none" stroke="ACCENT" stroke-width="3"/>
<line x1="-12" y1="18" x2="12" y2="18" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<!-- 5 WHAT'S NEXT: flecha -->
<line x1="-20" y1="0" x2="18" y2="0" stroke="ACCENT" stroke-width="4" stroke-linecap="round"/>
<path d="M 6 -12 L 20 0 L 6 12" fill="none" stroke="ACCENT" stroke-width="4" stroke-linecap="round" stroke-linejoin="round"/>
```

---

## 7. Ensamblaje

**CRÍTICO: el texto NO se mueve con el fondo.** Cada acto son dos capas — la
foto (`bg_N.jpg`) que se anima con Ken Burns, y el overlay+texto (`over_N.png`)
que va **fijo encima**. El error a evitar: aplicar el zoom al conjunto ya
fusionado, o meter el texto en la capa que se anima. El `zoompan` se aplica
**solo a la imagen**; el overlay se superpone después, sin animar.

Fondo animado por acto (Ken Burns), método `d=1` + `-frames:v`, escala 2×,
efecto alternado (1 zoom-in, 2 zoom-out, 3 pan, 4 zoom-in, 5 zoom-out), a **0.18
de intensidad**. El zoom va de 1.0 a 1.18 a lo largo del acto. (Si se siente
demasiado, baja a 0.15.)

```bash
FPS=30
FRAMES=$(python3 -c "print(round($DUR_N * $FPS))")

# expresión de zoom según el efecto:
#   zoom-in  : z='min(1+0.18*on/FRAMES,1.18)'  x,y centrados
#   zoom-out : z='max(1.18-0.18*on/FRAMES,1.0)' x,y centrados
#   pan      : z='1.18'  x='(iw-iw/zoom)*(on/FRAMES)'  y='(ih-ih/zoom)*(on/FRAMES)'

# ejemplo zoom-in — el zoompan va SOLO en [0:v] (la foto). El texto [1:v] se
# superpone encima con overlay, SIN animar, por eso queda fijo:
ffmpeg -y -loop 1 -framerate $FPS -t "$DUR_N" -i bg_N.jpg \
       -loop 1 -framerate $FPS -t "$DUR_N" -i over_N.png \
       -i voz_N.mp3 \
  -filter_complex "\
    [0:v]scale=2160:3840,zoompan=z='min(1+0.18*on/$FRAMES\,1.18)':d=1:\
x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1080x1920:fps=$FPS,setsar=1[bg];\
    [bg][1:v]overlay=0:0[v]" \
  -map "[v]" -map 2:a \
  -frames:v $FRAMES \
  -c:v libx264 -profile:v high -preset medium -crf 20 -pix_fmt yuv420p \
  -r $FPS -g 60 -c:a aac -b:a 192k -ar 48000 \
  -movflags +faststart clip_N.mp4
```

La razón de que el texto quede fijo: `[0:v]…zoompan…[bg]` anima **solo la foto**;
`[bg][1:v]overlay=0:0` pega el `over_N.png` (texto, transparente) encima en
posición estática. Si el texto se moviera, es que el zoompan se aplicó después
del overlay o a la capa equivocada — el orden importa: **primero animar la foto,
después superponer el texto.**

Concatena con `-f concat -c copy` en `video_mudo.mp4`. Luego la cama musical (paso 5).

**Nombre final:** `deepdive_YYYYMMDD.mp4`.

---

## 8. Subida a YouTube (programada a las 4am NY)

`videos.insert` con las credenciales del `.env`, `categoryId: 28`, subida
`resumable`. **Igual que el noticiero, el video se programa** para publicarse a
las **4:00am hora de Nueva York**, no de inmediato: sube ahora como `private`
con `publishAt`, y YouTube lo hace público a esa hora.

```python
import os, json
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

creds = Credentials(None,
    refresh_token=os.environ["YT_REFRESH_TOKEN"],
    client_id=os.environ["YT_CLIENT_ID"],
    client_secret=os.environ["YT_CLIENT_SECRET"],
    token_uri="https://oauth2.googleapis.com/token")
yt = build("youtube", "v3", credentials=creds)

# 4:00am NY fija; zoneinfo maneja EDT/EST solo
NY = ZoneInfo("America/New_York")
ahora = datetime.now(NY)
objetivo = ahora.replace(hour=4, minute=0, second=0, microsecond=0)
if objetivo <= ahora:
    objetivo += timedelta(days=1)
publish_at = objetivo.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

meta = json.load(open("meta_en.json"))
body = {
    "snippet": {"title": meta["titulo_video"], "description": meta["descripcion"],
                "tags": meta["tags"], "categoryId": "28"},
    "status": {"privacyStatus": "private", "publishAt": publish_at,
               "selfDeclaredMadeForKids": False,
               "containsSyntheticMedia": False},  # "No" a divulgación A/S: voz IA + stock real no lo requiere
}
resp = yt.videos().insert(part="snippet,status", body=body,
    media_body=MediaFileUpload(os.environ["NOMBRE"], resumable=True)).execute()
vid = resp["id"]
print(f"VIDEO_URL=https://www.youtube.com/watch?v={vid}")
print(f"STUDIO_URL=https://studio.youtube.com/video/{vid}/edit")
print(f"PUBLISH_AT={publish_at}  (4:00am America/New_York)")
```

- **`publishAt` exige `privacyStatus: private`** (el código lo fija explícito).
  Si el `.env` estuviera en `public`, se publicaría de inmediato.
- Captura `VIDEO_URL` y la hora programada. Si falla, reporta el error exacto y
  deja el mp4. Ante `invalid_grant`, el token expiró: avísame.

**Tags — llenar hasta ~490 de los 500 caracteres disponibles.** Este formato
apunta a maximizar alcance, así que aprovecha todo el espacio de tags.

**Bloque fijo (idéntico al del noticiero — los mismos tags "cabeza" en ambos
formatos, para un alcance base consistente):**

```python
TAGS_BASE = [
    "artificial intelligence", "AI news", "AI news today", "AI", "tech news",
    "technology", "machine learning", "ChatGPT", "OpenAI", "AI tools",
]  # ~110 caracteres, 10 tags
```

**Luego añade dinámicos** hasta ~490 caracteres, sacados de la noticia del video:
- Nombres propios: la empresa, producto, persona o modelo del deep dive.
- De formato: `AI explained`, `AI deep dive`, `AI analysis`, `tech explained`,
  `generative AI`, `future of AI`.
- Long-tail del tema concreto (frases de 2-3 palabras que alguien buscaría).

Regla de llenado: empieza por `TAGS_BASE`, agrega dinámicos midiendo el total
(`sum(len(t) for t in tags) + len(tags) - 1`), y para en ~490. Cuenta de verdad,
no adivines. Sin tags engañosos de temas que no aparecen — YouTube penaliza el
tag-stuffing; llenar los 500 es usar más tags *relevantes*, no basura.

---

## 9. Entrega

1. **La URL del video** bien visible al principio (y la de Studio).
2. El `deepdive_YYYYMMDD.mp4`.
3. El `manifiesto.json`.
4. Resumen: título y descripción publicados, duración total, **los hechos con
   sus fuentes** para verificar el fact-check, y conceptos de fondo descartados.

---

## Reglas de fallo

- Si la noticia no tiene sustancia para 5 actos, elige otra más densa.
- Si un hecho no se verifica en fuente primaria, no lo uses.
- Si el arco introduce causalidad sin respaldo, reescribe el acto.
- Si un título excede el ancho aun al mínimo, acórtalo.
- Si la subida falla, reporta el error exacto y deja el mp4. Ante `invalid_grant`,
  el token expiró: avísame.
- Si algo falla irrecuperable, entrega lo que alcanzaste y di en qué acto/paso.

---

## Al terminar

Dame tu **top 3 de sugerencias** priorizadas para mejorar el formato. Enfócate
en lo que más movería la aguja: claridad de la explicación, gancho, o ritmo.

## Reporte de medición (para optimización de tokens)

Al final, además del top 3, dame un **reporte de ejecución** para medir el costo
de la corrida:

- **Número de llamadas a herramientas** que hiciste en total (bash, ffmpeg,
  ffprobe, rsvg-convert, python, descargas, etc.).
- **Tamaño aproximado de salida** que esas herramientas devolvieron (suma de
  caracteres/líneas de los logs que tuviste que leer), aunque sea una estimación.
- Las 2-3 fases que más llamadas consumieron (p.ej. "fondos: 12 llamadas",
  "ensamblaje: 18 llamadas").

Esto es una línea base para comparar antes/después de consolidar la producción
en un solo script. No cambia el video; es solo instrumentación.
