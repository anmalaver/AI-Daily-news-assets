# Rutina diaria — Shorts de noticias de IA

Produce **un video vertical en inglés** con las 5 noticias más importantes de
inteligencia artificial de las últimas 24 horas, y **súbelo a YouTube** con las
credenciales del `.env`. Al terminar, repórtame la URL del video para revisarlo.

> **Idioma: solo inglés por ahora.** El canal se está madurando en inglés antes
> de escalar a otros idiomas. La arquitectura multi-idioma sigue descrita más
> abajo (paleta por offset, reescritura por idioma, etc.) y queda lista para
> cuando se reactive, pero **por ahora genera únicamente `en`**. Ignora es, pt y
> fr hasta nueva orden.

La búsqueda de noticias, la selección y los fondos se hacen una vez. La config
multi-idioma se conserva para el futuro:

```
IDIOMAS = [
  { code: "es", voz: "es-MX-DaliaNeural",            offset: 0 },
  { code: "en", voz: "en-US-AvaMultilingualNeural",  offset: 1 },   # ← único activo
  { code: "pt", voz: "pt-BR-FranciscaNeural",        offset: 2 },
  { code: "fr", voz: "fr-FR-DeniseNeural",           offset: 3 },
]
```

Inglés usa `offset: 1`, así que su paleta es la de `(día+1)%5` y su foto el
índice 1 del pool — se conserva tal cual del diseño multi-idioma, no cambia
nada. La voz usa `--rate=+5%`, `--pitch=+2Hz`.

---

## 0. Setup

**Primero verifica, y solo instala si algo falta.** Un solo comando:

```bash
falta=()
for t in ffmpeg ffprobe rsvg-convert jq edge-tts python3; do
  command -v "$t" >/dev/null || falta+=("$t")
done
fc-list 2>/dev/null | grep -qi anton  || falta+=("fuentes")
fc-list 2>/dev/null | grep -qi barlow || falta+=("fuentes-cuerpo")
fc-list 2>/dev/null | grep -qi "dm serif" || falta+=("fuente-intro")
cb=$(python3 -c 'import certifi;print(certifi.where())' 2>/dev/null)
[ -n "$cb" ] && [ -f "$cb.orig" ]     || falta+=("certificados")
for loc in es_ES en_US pt_BR fr_FR; do
  locale -a 2>/dev/null | grep -qi "$loc" || { falta+=("locales"); break; }
done
[ -n "$PEXELS_API_KEY" ]              || falta+=("PEXELS_API_KEY")
[ ${#falta[@]} -eq 0 ] && echo "ENTORNO OK" || echo "FALTA: ${falta[*]}"
```

- Si imprime `ENTORNO OK`, **no instales nada** y pasa directo al paso 1.
- Si imprime `FALTA: …`, corre `setup.sh` una vez y vuelve a verificar.
  `setup.sh` ya es idempotente: instala solo lo ausente.
- Si falta `locales`, `apt-get install -y locales-all` habilita es/en/pt/fr.
- Si tras correrlo sigue faltando algo, detente y dime qué.

`PEXELS_API_KEY` puede venir del entorno preconfigurado y no de un archivo:
**se valida la variable, no la existencia de `.env`**. Si existe `.env`, cárgalo
con `set -a; . ./.env; set +a`; si no existe pero la variable está, sigue.

Define el índice de paleta del día y deriva el de cada idioma con su `offset`:

```bash
DIA=$(( 10#$(date +%j) ))
# por idioma: IDX_lang = (DIA + offset) % 5
# es→+0   en→+1   pt→+2   fr→+3
```

| IDX | BASE | ACCENT |
|-----|---------|---------|
| 0 | #0B1B2B | #FFB020 |
| 1 | #24101A | #FF7A6B |
| 2 | #0C1F17 | #5FE08A |
| 3 | #1A1024 | #C77DFF |
| 4 | #101820 | #38BDF8 |

Así los cuatro videos del mismo día usan paletas distintas entre sí, y cada
idioma rota día a día. El español lleva la paleta del día; inglés, portugués y
francés las de los días siguientes.

---

## 1. Noticias

Busca en la web noticias de IA publicadas en las **últimas 24 horas**. Selecciona 5.

**Criterios de selección, en orden:**

1. **Ventana temporal.** Prioriza estrictamente lo publicado en las últimas 24h.
   Si no consigues 5 que la cumplan, puedes estirar hasta 48h, pero entonces
   avísame en el resumen cuáles se salieron de la ventana y por qué las dejaste.
   Nunca estires más de 48h.
2. Hechos concretos y verificables: lanzamientos, cifras, acuerdos, regulación,
   resultados de investigación. No opinión ni listas de "las mejores herramientas".
3. Fuente primaria por encima de agregador. Blog oficial de la empresa, paper,
   comunicado regulatorio, medio de tecnología establecido. Descarta blogs SEO
   que reciclan lo mismo.
4. Diversidad: no cinco noticias de la misma empresa ni del mismo subtema.
5. Interés para audiencia general amplia, no solo para investigadores. Recuerda
   que estas noticias se publicarán en cuatro idiomas: prioriza hechos con
   relevancia internacional, no solo para un país.

Si una noticia no supera los criterios, descártala y busca otra. Es preferible
buscar de más que rellenar con algo débil.

**Orden de las 5 (importa para la retención).** Una vez elegidas, ordénalas así:
- **La #1 es la más fuerte** del día — es el gancho, y de ella sale el hook de la
  intro.
- **La #5 es la segunda más fuerte**, reservada a propósito para cerrar con
  energía. Cerrar con la noticia más floja hace que la gente abandone antes del
  final; cerrar fuerte sostiene la retención hasta el cierre.
- Las #2, #3, #4 son las tres restantes, en el orden que dé mejor ritmo (evita
  dos muy técnicas seguidas).

Esto conecta con el teaser del hook: si en la intro insinúas otra noticia, que
sea real y que valga la pena, porque va a estar entre las fuertes. Nunca
prometas un cierre que la #5 no cumpla.

**Esta búsqueda y selección se hace una sola vez.** Las 5 noticias son las
mismas para los cuatro idiomas; solo cambia cómo se redactan.

Antes de continuar, muéstrame las 5 en una lista de una línea cada una,
con fuente y fecha. Sigue sin esperar mi confirmación.

---

## 2. Guion y manifiesto

**Antes de escribir nada, resuelve la fecha en los cuatro idiomas:**

```bash
FECHA=$(date +%F)                              # 2026-07-24
DIA=$(( 10#$(date +%j) ))                      # día del año, para las paletas
FECHA_ES=$(LC_TIME=es_ES.UTF-8 date "+%A %-d de %B de %Y")   # viernes 24 de julio de 2026
FECHA_EN=$(LC_TIME=en_US.UTF-8 date "+%A, %B %-d, %Y")        # Friday, July 24, 2026
FECHA_PT=$(LC_TIME=pt_BR.UTF-8 date "+%A, %-d de %B de %Y")   # sexta-feira, 24 de julho de 2026
FECHA_FR=$(LC_TIME=fr_FR.UTF-8 date "+%A %-d %B %Y")          # vendredi 24 juillet 2026
```

Si algún locale no existe, arma esa fecha a mano con los nombres de día y mes
del idioma, en minúscula y sin abreviar. El badge de la intro usa una versión
corta en mayúsculas (`24 JULIO 2026`, `JULY 24 2026`, `24 JULHO 2026`,
`24 JUILLET 2026`).

Los valores de los ejemplos son ilustrativos: reemplázalos por los reales.

### Estructura del manifiesto

El manifiesto es **un solo archivo con una sección compartida y una por idioma**.
La sección compartida se genera una vez; el bloque `idiomas` se reescribe cuatro
veces (ver la regla de reescritura abajo).

```json
{
  "fecha": "<FECHA>",
  "compartido": {
    "noticias": [
      {
        "n": 1,
        "conceptos": ["data center servers", "server rack close up", "abstract blue network"],
        "fuente": "https://..."
      }
    ]
  },
  "idiomas": {
    "es": {
      "voz": "es-MX-DaliaNeural",
      "paleta": { "base": "<BASE de (DIA+0)%5>", "accent": "<ACCENT de (DIA+0)%5>" },
      "fondo_offset": 0,
      "fecha_texto": "<FECHA_ES>",
      "badge": "24 JULIO 2026",
      "intro_guion": "Nvidia just backed OpenAI with 250 billion dollars — the biggest AI story today. Plus the one company quietly betting the other way.",
      "titulo_video": "...",
      "descripcion": "...",
      "tags": ["inteligencia artificial", "..."],
      "noticias": [
        {
          "n": 1,
          "guion": "Texto tal cual se leerá, ya adaptado fonéticamente. 26 a 30 palabras.",
          "cierre": "Frase de opinión o pregunta abierta, 8 a 12 palabras.",
          "pronunciaciones": { "Mobileye": "Mobilái" },
          "titulo": ["CLAUDE FABLE 5", "SALE AL MUNDO"],
          "cuerpo": [
            "Anthropic activó el despliegue global tras el",
            "levantamiento de las restricciones de",
            "exportación de Estados Unidos, junto a nuevas",
            "medidas de ciberseguridad."
          ]
        }
      ]
    },
    "en": { "voz": "en-US-AvaMultilingualNeural", "paleta": "...(DIA+1)%5", "fondo_offset": 1, "...": "..." },
    "pt": { "voz": "pt-BR-FranciscaNeural",       "paleta": "...(DIA+2)%5", "fondo_offset": 2, "...": "..." },
    "fr": { "voz": "fr-FR-DeniseNeural",          "paleta": "...(DIA+3)%5", "fondo_offset": 3, "...": "..." }
  }
}
```

### Reescritura por idioma, no traducción

Cada idioma se **redacta desde la noticia original**, no se traduce del español.
Una traducción literal suena acartonada, y el `cierre` de opinión sobre todo se
vuelve raro palabra por palabra. Escribe cada versión como la redactaría un
periodista nativo de ese idioma: mismo hecho, mismas cifras, fraseo natural.

El `titulo` y el `cuerpo` también se reescriben por idioma respetando sus
propios límites de caracteres (los mismos números aplican a los cuatro).
Ojo: un título que cabe en español puede desbordar en alemán o alargarse en
francés — revisa los límites en cada idioma, no asumas que por caber en uno
cabe en todos.

**Límites duros. Si no se cumplen, reescribe antes de renderizar. Aplican por
idioma:**

| Campo | Límite |
|---|---|
| `guion` de cada noticia | 30–36 palabras (ajustado: a la voz +5% más lenta, 38 palabras rozan el tope de 14s) |
| siglas técnicas deletreadas por escena | máx 2 |
| `cierre` de cada noticia | 8–12 palabras |
| `guion` de la intro | **16–24 palabras** (hook: hecho fuerte + teaser concreto, sin fecha hablada ni marca) |
| `titulo` | máx 2 líneas, **máx 16 caracteres por línea** |
| `cuerpo` | **4 o 5 líneas**, **máx 42 caracteres por línea** (llena mejor el espacio inferior; a 48px caben hasta 5 líneas dentro de la safe zone) |
| `tags` | bloque fijo `TAGS_BASE` + dinámicos hasta ~490 de los 500 caracteres |
| `titulo_video` | máx 90 caracteres |

**Siglas técnicas: el conteo de palabras engaña.** Una sigla deletreada pesa
mucho más al leerse que una palabra normal: "GB300" es "ge-be-tres-cero-cero",
cinco golpes de voz. Una escena con "AMD MI450, Helios y GB300" tiene pocas
palabras pero se atropella. Regla: **máximo 2 siglas o códigos alfanuméricos
deletreados por escena**. Si la noticia tiene más, quédate con los 1-2 más
relevantes y describe el resto ("los nuevos aceleradores", "su plataforma").
Esto trabaja junto al tope de 14s del paso 4: la regla evita el problema, el
tope lo caza si se escapa.

Los saltos de línea de `titulo` y `cuerpo` los defines tú: SVG no hace
word-wrap. Balancea las líneas, no dejes una de dos palabras y otra de seis.
La última línea del cuerpo no debe quedar con una sola palabra.

**Códigos alfanuméricos en el título.** Cadenas tipo `MI400`, `GPT-5`, `K3` o
`MI400 Y HELIOS` se vuelven ilegibles en Anton, donde mayúsculas y dígitos
tienen formas muy parecidas. Si un título los contiene: ponlo solo en su línea,
o sepáralo con `·` (`MI400 · HELIOS`), o cámbialo por una forma verbal
(`NUEVOS MI400`). Nunca dos códigos pegados en la misma línea.

**Tono:** informativo y directo. Nada de clickbait, signos de admiración,
"increíble", "revoluciona" ni preguntas retóricas en el cuerpo. El título es
sustantivo, no gancho.

### Metadatos de YouTube (título, descripción, tags)

Estos tres campos son **para la plataforma, no para el video**, y su único
trabajo es atraer clic y crecimiento. No confundir con el `titulo` de la
tarjeta, que es sobrio: **el `titulo_video` sí puede tener gancho.** Se
redactan por idioma, nativos, no traducidos.

**`titulo_video` (máx 90 caracteres, idealmente 60-70).**
Es lo que decide el clic en el feed Y lo que indexa el buscador. Estructura que
funciona: el hecho más fuerte del día + un gancho de curiosidad, sin mentir.
- Ancla en la noticia #1 o la más llamativa de las cinco, no en las cinco a la
  vez. Un título que promete todo no promete nada.
- **Estructura de dos partes: título legible con marcas + dos hashtags.**
  - **Parte legible:** una frase natural anclada en la noticia más fuerte, que
    incluya la(s) **marca(s) de mayor arrastre** (OpenAI, Nvidia, Google,
    Anthropic, Gemini, ChatGPT, etc.). Si caben dos marcas de peso sin forzar,
    mejor ("Nvidia just came for AMD"). La gente busca y hace clic por nombres que
    reconoce. Un título con "Nvidia" y "OpenAI" rinde más que "una empresa de
    chips y un laboratorio de IA".
  - **Dos hashtags al final:** anexa los **dos tags más importantes** del día como
    hashtags (`#Nvidia #OpenAI`, o un término de tema fuerte tipo `#AInews`).
    YouTube los hace clicables y cuentan para descubrimiento. Sin espacios dentro
    del hashtag; no más de dos.
  - Patrón: `Título legible con marcas #Tag1 #Tag2`.
- Permitido el número: "5 noticias de IA que…". Los listicles rinden en Shorts.
- Prohibido: TODO EN MAYÚSCULAS, más de un signo de admiración, promesas falsas
  ("esto lo cambia todo"), sufijos de relleno tipo "· IA hoy" (los hashtags
  reemplazan eso), y el emoji de flecha/fuego en exceso. Un emoji puntual está bien.
- Vigila el límite de 90 caracteres contando los hashtags.
- Ejemplos de estructura (no copiar literal):
  - es: "Anthropic libera Claude Fable 5 y sacude la IA #Anthropic #IA"
  - en: "Nvidia just came for AMD — today's 5 AI stories #Nvidia #AInews"
  - pt: "OpenAI admite falha grave no ChatGPT e mais 4 notícias #OpenAI #IA"
  - fr: "Google lance Gemini 3 et bouscule la course à l'IA #Gemini #IA"

**`descripcion`.**
Las **primeras 1-2 líneas** son lo único visible antes del "…more" y de lo que
más pesa en el buscador de YouTube, así que trátalas con la misma disciplina que
el título:
- **Primera línea:** la keyword del nicho (IA / AI news) + **la marca principal
  del día** (la empresa/modelo de la noticia #1), en una frase con gancho. Igual
  que el título, la gente busca por nombres reconocibles. Ejemplo: "Nvidia bets
  $250B on OpenAI — plus 4 more AI news stories you need today."
- **Segunda línea:** refuerza con un término buscable más y el marco del día.
Luego, más abajo (ya no tan visible, pero indexado):
- Un renglón por noticia (los 5 titulares), para retención y SEO.
- Llamado a suscribirse enfocado en el hábito diario: "Cada día, la IA sin
  rodeos. Suscríbete."
- Los créditos de Pexels (fotógrafos) al final.
- 3-5 hashtags relevantes al cierre (`#IA #inteligenciaartificial #tech`…).

**`tags` — llenar hasta ~490 de los 500 caracteres disponibles.**
YouTube permite 500 caracteres de tags y hoy se están desperdiciando (solo ~166
usados). Aprovéchalos: más tags relevantes = más superficies de descubrimiento.

**Bloque fijo (hardcodeado, va SIEMPRE, idéntico en noticiero y deep dive)** —
son los tags "cabeza" del nicho, de alto volumen de búsqueda. El primero es la
palabra clave objetivo exacta (YouTube le da peso extra al primer tag):

```python
TAGS_BASE = [
    "artificial intelligence", "AI news", "AI news today", "AI", "tech news",
    "technology", "machine learning", "ChatGPT", "OpenAI", "AI tools",
]  # ~110 caracteres, 10 tags
```

**Luego añade tags dinámicos del día** hasta acercarte a ~490 caracteres (deja
un pequeño margen para no pasarte de 500). Los dinámicos salen de las 5 noticias:
- Nombres propios del día: empresas, productos, personas, modelos (`Anthropic`,
  `Gemini 3`, `Nvidia`, `Claude`, etc.).
- Frases long-tail que alguien buscaría (2-3 palabras): `new AI model`,
  `AI chip`, `AI regulation`, según los temas reales del día.
- Variantes de descubrimiento: `AI breakthrough`, `AI update`, `generative AI`.

Regla de llenado: empieza por `TAGS_BASE`, agrega dinámicos uno a uno midiendo el
total (suma de caracteres de los tags + las comas), y para cuando llegues a ~490.
**Cuenta los caracteres de verdad** (`sum(len(t) for t in tags) + len(tags) - 1`),
no los adivines.

Sin tags engañosos de temas que no aparecen en el video: YouTube penaliza el
tag-stuffing. Llenar los 500 es usar más tags *relevantes*, no meter basura.

Cada noticia termina con una frase que invita a pensar. Se concatena al final
del `guion` al generar el audio, **y no aparece en la tarjeta**.

El objetivo es conversación, no polémica. La frase señala la tensión real del
hecho, o hace una pregunta genuinamente abierta.

**Sí:**
- "¿Alcanza con un botón de apagado, o llega tarde?"
- "La pregunta es quién decide cuándo apretarlo."
- "Medir el uso real resultó más difícil que anunciarlo."
- "Queda por ver si los clientes lo notan."

**No:**
- Preguntas con respuesta implícita: "¿No es preocupante que…?"
- Indignación fabricada: "Otra vez las grandes tecnológicas…"
- Posturas políticas o partidistas de cualquier signo
- Predicciones categóricas: "Esto va a cambiarlo todo."
- Interpelaciones vacías: "¿Tú qué opinas?" al final de las cinco
- **Cliché de periodista**, sobre todo al traducir: "enfrenta una prueba de
  credibilidad", "faces a public credibility test", "marca un antes y un
  después". Suena a teleprónter, no a alguien hablándote.

**El cierre se escribe conversacional en cada idioma, no se traduce.** Un cierre
en español que suena natural, traducido literal al inglés cae en el cliché
noticioso. Redáctalo como lo diría una persona real en ese idioma:

- **es:** "Y ahí está el detalle que nadie mencionó." · "Suena bien, pero falta
  la letra chica."
- **en:** "And nobody's really talking about that part." · "Sounds great —
  until you read the fine print."
- **pt:** "E é aí que ninguém prestou atenção." · "Parece ótimo, até ler os
  detalhes."
- **fr:** "Et c'est là que personne n'a regardé." · "Sur le papier, ça passe.
  En vrai, on verra."

La prueba: si el cierre suena a titular de telediario, reescríbelo. Si suena a
un amigo que te comenta la noticia, va bien.

Si una noticia no tiene una tensión honesta que señalar, un cierre neutro que
apunte a lo que falta por saber es mejor que forzar controversia.

### Pronunciación

edge-tts no acepta SSML personalizado, así que **no existen `<phoneme>` ni
`<sub alias>`**. La única forma de corregir la pronunciación es reescribir la
palabra fonéticamente dentro del `guion`.

Esto es seguro porque el `guion` **nunca se muestra en pantalla**: los textos
visibles son `titulo` y `cuerpo`, que conservan siempre la grafía correcta.

**La pronunciación es por idioma.** Una sigla o marca en inglés se lee distinto
según el idioma del narrador. Reescribe cada término como debería sonar leído
por un nativo de *ese* idioma, y registra los cambios en `pronunciaciones` de
cada versión.

| Término | es (Dalia) | en (Ava) | pt (Francisca) | fr (Denise) |
|---|---|---|---|---|
| AI | inteligencia artificial | AI *(tal cual)* | inteligência artificial | intelligence artificielle |
| Mobileye | Mobilái | *(tal cual)* | Mobilái | Mobilaïe |
| OpenAI | Óupen ei ái | *(tal cual)* |Ôupen ei ái | Open-A-I |
| GPT | ge pe te | *(tal cual)* | guê pê tê | gé pé té |
| Google | Gúgol | *(tal cual)* | Gúgou | Gougueul |
| startup | startap | *(tal cual)* | startâp | starteupe |

Regla general: en **inglés** casi nada necesita reescritura, porque la voz ya
está entrenada en inglés y lee las siglas y marcas correctamente — solo
interviene en palabras de otros idiomas. En **español, portugués y francés**,
reescribe las siglas y anglicismos a la fonética local. Ante la duda entre sigla
y forma larga, usa la forma larga traducida: siempre suena mejor.

Nombres propios que ya se leen bien en el idioma (Meta, Anthropic, AMD, Intel)
se dejan tal cual en los cuatro.

`concepto` debe ser uno de estos textos exactos:

```
INFRAESTRUCTURA   data center servers · server rack close up · fiber optic cables
HARDWARE          circuit board macro · silicon wafer · computer chip macro
MODELOS           abstract blue network · glowing data particles · code on screen
ROBÓTICA          humanoid robot · robotic arm factory · robot hand closeup
MERCADOS          stock exchange trading floor · financial charts screen · wall street building
REGULACIÓN UE     european parliament brussels · eu flags building · european commission
REGULACIÓN EEUU   us capitol building · white house washington · american flag government
REGULACIÓN OTRA   courthouse columns · government building · legal documents gavel
SEGURIDAD         cybersecurity lock · hacker dark room · network security screen
SALUD             doctor with tablet · medical laboratory · hospital technology
CONSUMO           person using laptop · smartphone in hand · people working office
EMPLEO            empty office desks · job interview · worker warehouse
MOVILIDAD         autonomous car sensors · self driving car road · electric vehicle
ENERGÍA/ESPACIO   satellite space earth · power plant energy · solar panels field
```

**El array `conceptos` es campo compartido** (vive en `compartido.noticias`,
no por idioma): la búsqueda de fondos se hace una sola vez. Cada escena lleva
3 opciones de su categoría, en orden de preferencia. Elige la categoría por el
tema de la noticia y ajusta la geografía cuando aplique: una noticia de
regulación europea usa REGULACIÓN UE, no el genérico.

**La intro tiene su propio pool de conceptos abstractos**, no uno fijo. Si los
cuatro idiomas usan el mismo concepto, las cuatro intros se parecen aunque
cambie la foto por offset. Usa este pool y aplícale la misma rotación por
`fondo_offset`:

```
INTRO   abstract blue network · glowing data particles · digital abstract background · technology particles
```

Todos son abstractos/tech para mantener la identidad, pero dan variedad visual
entre los cuatro idiomas del día. Busca los cuatro, arma el pool combinado y
reparte por offset igual que las noticias. El filtro de calidad ya existente
(mínimo 5 fotos, `alt` coherente) descarta solo el concepto que venga vacío, así
que si alguno de estos rinde poco en Pexels, no rompe nada: simplemente no
aporta al pool.

---

## 3. Fondos

**Esta búsqueda se hace una sola vez y sirve para los cuatro idiomas.** Lo que
cambia por idioma es cuál foto del pool se usa, vía `fondo_offset`.

**Toda llamada a Pexels debe llevar User-Agent.** Cloudflare responde 403 a
`urllib` porque su cabecera por defecto es `Python-urllib/3.12`. Sin esto la
API devuelve cero resultados y parece que todos los conceptos fallaron:

```python
req.add_header("User-Agent", "Mozilla/5.0")
req.add_header("Authorization", os.environ["PEXELS_API_KEY"])
```

Para cada escena (intro + 5 noticias), recorre su array `conceptos` **en orden**
y quédate con el primero que sirva:

1. Busca en Pexels: `orientation=portrait`, `size=large`, `per_page=15`.
2. Descarta el concepto si devuelve menos de 5 fotos.
3. Revisa el campo `alt` de las primeras 8 fotos y descarta el concepto si
   ninguna es coherente con el tema.
4. Guarda la lista completa de fotos que pasaron el filtro para esa escena
   (su pool). No elijas una sola todavía.

**El filtro `alt` exige co-ocurrencia, no palabra suelta.** Un candado de
verdad no es ciberseguridad: `lock` solo cuenta si aparece junto a `cyber`,
`network`, `digital`, `data`, `screen`, `computer` o `security`. La misma regla
para otros términos ambiguos: `chip` con `computer`/`circuit`/`silicon`,
`cloud` con `server`/`data`/`computing`, `key` con `digital`/`encryption`.

**Descarta imágenes generativas.** Pexels ya tiene bastante imagen hecha con IA
y se ve plástica, justo lo contrario de la sensación de foto real que buscas.
Baja de prioridad cualquier foto cuyo `alt` contenga `generated`, `generative`,
`AI-generated`, `AI generated` o `rendered`. **Pero no es descarte absoluto:**
si tras aplicar todos los filtros una escena se quedaría sin fotos, úsalas —
antes una imagen plástica que ningún fondo. El orden de preferencia es: foto
real temática > foto generativa temática > fallback `abstract blue network`.

Si los tres conceptos fallan, cae a `abstract blue network`.

### Foto por idioma

Cada idioma toma **una foto distinta del mismo pool**, para que los cuatro
videos del día no se vean idénticos:

```
foto_del_idioma = pool[ fondo_offset ]   # es→0  en→1  pt→2  fr→3
```

Con `per_page=15` el pool casi siempre tiene 4+ fotos y cada idioma recibe una
distinta. Pero si el filtro de calidad dejó **menos de 4 fotos**, evita repetir
a ciegas con estos pasos, en orden:

1. **Amplía el pool primero.** Vuelve a buscar con el **segundo concepto** del
   array (y el tercero si hace falta) y suma esas fotos al pool, aunque el
   primero ya hubiera dado algunas. Es preferible un fondo del concepto B
   temáticamente válido que repetir el mismo fondo en dos idiomas.
2. Si aun así el pool combinado tiene menos de 4, **asigna sin repetir hasta
   agotarlo** y solo entonces reutiliza: `pool[offset % len(pool)]`. Así, con 3
   fotos, tres idiomas van únicos y solo el cuarto repite, en vez de repetir dos.
3. Reparte la repetición en la escena menos visible. Si hay que duplicar una
   foto, que sea en una noticia distinta para cada idioma, no siempre en la
   misma, para que ningún par de videos comparta más de un fondo.

Reporta en el resumen final si alguna escena cayó en reutilización, indicando
en qué idiomas se repitió. Es señal de que ese concepto necesita más
alternativas en el catálogo.

Descarga cada foto con `fit=crop&w=1080&h=1920`. Guarda el `photographer` de
cada una: van en la descripción del video de su idioma.

Al terminar, repórtame solo qué conceptos **descartaste** y por qué. No listes
la foto elegida de cada escena; eso se ve en el video.

### Fondo en video para intro, noticia 2 y noticia 4

Tres escenas usan **video de fondo** en vez de foto, para dar movimiento real
donde más engancha: la **intro**, la **noticia 2** y la **noticia 4**. El resto
(noticias 1, 3, 5) siguen con foto. Si no se consigue un video válido para una
de esas tres, **cae a foto** con el mismo flujo de siempre — nunca se queda sin
fondo.

Usa la **API de video de Pexels** (`https://api.pexels.com/videos/search`), misma
key y mismo User-Agent obligatorio:

```python
req.add_header("User-Agent", "Mozilla/5.0")
req.add_header("Authorization", os.environ["PEXELS_API_KEY"])
# GET https://api.pexels.com/videos/search?query=<concepto>&orientation=portrait&per_page=10
```

Para cada una de las tres escenas (intro, n2, n4):

1. Busca con el **mismo `conceptos`** que ya tiene la escena, en orden.
2. Filtra los resultados: prefiere `orientation=portrait` o vertical (alto ≥
   ancho). Un video horizontal recortado a 9:16 pierde los lados; úsalo solo si
   no hay vertical.
3. De cada video, elige el archivo (`video_files`) de mayor resolución que no
   pase de 1080 de ancho, formato `.mp4` (`link` directo).
4. Descarga el clip. Si ningún concepto da un video válido (o la descarga falla),
   **marca la escena para usar foto** y sigue el flujo normal de fotos.
5. Guarda el `user.name` del video para los créditos (como el `photographer`).

**Recorte a vertical y sincronización con el audio** (el audio manda, igual que
con foto). Un solo comando cubre los dos casos —clip corto se repite, clip largo
se corta— gracias a `-stream_loop -1 -t`:

```bash
# DUR_N = duración del audio de la escena (se calcula en el paso 4)
ffmpeg -y -stream_loop -1 -t "$DUR_N" -i clip_crudo.mp4 \
  -vf "scale=-1:1920,crop=1080:1920:(iw-1080)/2:0,setsar=1,fps=30" \
  -an -c:v libx264 -preset medium -crf 20 -pix_fmt yuv420p bg_N.mp4
```

El clip queda como `bg_N.mp4` (video) en vez de `bg_N.jpg` (foto). En el paso 6,
las escenas con video **no llevan Ken Burns** (el clip ya se mueve solo); las de
foto sí. El overlay+texto (`over_N.png`) va encima igual en ambos casos.

Como el video pesa mucho más que la foto (5-30 MB por clip vs ~200 KB), estas
tres escenas hacen la corrida más lenta y pesada. Es aceptable en la corrida de
2am. Si Pexels-video falla del todo, el fallback a foto mantiene el video del
día saliendo igual.

---

## Del paso 4 al 6: repetir por idioma

**Los pasos 4, 5 y 6 se ejecutan una vez por cada idioma** (es, en, pt, fr),
usando la sección `idiomas[code]` de su manifiesto. Trabaja en un subdirectorio
por idioma (`out/es/`, `out/en/`…) para no pisar archivos entre corridas.

Lo que varía en cada pasada: la `voz`, la `paleta`, el `fondo_offset`, y los
textos (`intro_guion`, `titulo`, `cuerpo`, `guion`+`cierre`). El resto es igual.

---

## 4. Audio

Voz: la del idioma (`idiomas[code].voz`), con `--rate=+5%`, `--pitch=+2Hz`.

Para cada escena:

- **Intro — hook, no portada de marca.** El texto es `intro_guion`. Los primeros
  2 segundos deciden si el usuario se queda, así que la intro **no** abre con la
  marca ("AI today, no fluff…") — eso es una portada, no un gancho. Abre con **el
  hecho más fuerte del día, concreto y específico**, y desliza un **teaser real**
  hacia otra noticia. Reglas:
  - Arranca con el dato más impactante de la noticia #1, dicho en cifras o hechos
    concretos ("Nvidia just backed OpenAI with 250 billion dollars"), no con una
    valoración vaga ("something huge happened in AI today").
  - Añade un teaser **específico** de otra noticia sin resolverlo ("plus the one
    company quietly betting the other way"). El teaser nombra algo real; la
    curiosidad viene del dato, no de una promesa.
  - **Prohibido el clickbait de meta-promesa:** nada de "the last one will shock
    you", "you won't believe #5", "wait for the end". La gente rechaza esas
    fórmulas. El gancho es la especificidad, no la promesa de una emoción.
  - No tiene `cierre`. Corta y directa: el hecho fuerte + el teaser, y a las
    noticias. Sin fecha hablada ni preámbulo.
- **Noticias (n1–n5):** el texto es `guion` + espacio + `cierre`. El cierre se
  lee, no se muestra en la tarjeta.

```bash
edge-tts --voice "$VOZ" --rate=+5% --pitch=+2Hz \
  --file guion_N.txt --write-media raw_N.mp3 --write-subtitles sub_N.srt
ffmpeg -y -i raw_N.mp3 -af "apad=pad_dur=0.8" -c:a libmp3lame -b:a 192k aud_N.mp3
DUR_N=$(ffprobe -v error -show_entries format=duration -of csv=p=0 aud_N.mp3)
```

**La duración del audio manda sobre la del clip, nunca al revés.**

Si alguna escena supera los **14 segundos** con el silencio incluido, acorta el
guion y regenera. La suma total debe quedar entre **70 y 85 segundos**.

### Cabezote sobre la voz de la intro

La intro lleva el cabezote (`audio/intro_master.mp3`, 13.77s) fundido con la
voz. El cabezote arranca solo 1.2s, entra la voz encima, y **ambos terminan
juntos** con un fade. El audio de la intro se arma así, tras generar `raw_intro.mp3`:

```bash
V=$(ffprobe -v error -show_entries format=duration -of csv=p=0 raw_intro.mp3)
ADELAY=1.2; PAD=0.8
TOTAL=$(python3 -c "print(round($ADELAY+$V+$PAD,3))")
FADE_ST=$(python3 -c "print(round($TOTAL-0.9,3))")
MS=$(python3 -c "print(int($ADELAY*1000))")

ffmpeg -y -i raw_intro.mp3 -i audio/intro_master.mp3 \
  -filter_complex "\
    [0:a]adelay=$MS|$MS,apad=whole_dur=$TOTAL[v];\
    [1:a]atrim=0:$TOTAL,volume=-11dB,afade=t=out:st=$FADE_ST:d=0.9[m];\
    [v][m]amix=inputs=2:duration=first:normalize=0[a]" \
  -map "[a]" -c:a libmp3lame -b:a 192k aud_intro.mp3
DUR_INTRO=$(ffprobe -v error -show_entries format=duration -of csv=p=0 aud_intro.mp3)
```

- El cabezote a **−11 dB** (protagonista compartido, no cama de fondo).
- `adelay=1200`: la voz entra 1.2s después, dejando que el cabezote abra solo.
- El fade de 0.9s arranca antes del final, así música y voz mueren juntas.
- `apad=whole_dur` evita que `amix` corte la mezcla al terminar la voz.
- La cama musical de las noticias **no** se mezcla aquí; va en una sola pasada
  al final (paso 6), para que sea continua bajo las 5 escenas.

---

## 5. Tarjetas

Un SVG de 1080×1920 por escena, con la paleta **del idioma en curso**
(`idiomas[code].paleta`), no la del día global.

### La imagen va embebida, no referenciada

**librsvg bloquea toda imagen que no esté en el mismo directorio que el SVG, y
falla en silencio: no da error, simplemente no la pinta.** Verificado: ruta
absoluta falla, ruta relativa a carpeta hermana falla, `file://` falla.

Embebe el JPEG como data URI. Es la única forma que no depende de rutas:

```python
import base64
b64 = base64.b64encode(open(bg_path, "rb").read()).decode()
href = f"data:image/jpeg;base64,{b64}"
```

```xml
<image xlink:href="{href}" x="0" y="0" width="1080" height="1920"
       preserveAspectRatio="xMidYMid slice"/>
```

### Estructura de las tarjetas de noticia (n1–n5)

Las noticias llevan **fondo animado** (ver paso 6), así que su tarjeta se
renderiza en **dos capas separadas**, no en un PNG único:

- **Capa foto** (`bg_N.jpg`): la foto de Pexels ya recortada a 1080×1920, sin
  overlay ni texto. Es la que se mueve.
- **Capa overlay** (`over_N.png`): un SVG con **fondo transparente** que lleva
  solo el degradado del overlay + etiqueta + título + regla + cuerpo. Se
  renderiza con `rsvg-convert` a PNG con alfa y queda **quieta** encima de la
  foto animada.

La capa overlay **no** lleva `<image>` de fondo: empieza directamente por el
`<rect>` del degradado sobre un SVG sin color de fondo (transparente por
defecto). El resto de elementos son idénticos:

- Overlay: `linearGradient` vertical del color BASE — opacidad 0.42 arriba,
  0.62 al 35%, 0.90 al 60%, 0.97 abajo
- Etiqueta: `IA · HOY`, Anton 38px, ACCENT, `letter-spacing="14"`, en x=90 y=600
- Título: Anton 128px blanco, `letter-spacing="-1"`, primera línea en y=740,
  interlineado 120
- Regla: rect ACCENT, x=90, ancho 180, alto 8, a 52px bajo la última línea del título
- Cuerpo: Barlow 48px `#D6DEE8`, primera línea 90px bajo la regla, interlineado 66,
  **4 o 5 líneas** (subido de 42px y de 3-4 líneas para mejor legibilidad y para
  llenar el espacio inferior; a 48px la 5ª línea termina en y≈1266, dentro de la
  safe zone). El interlineado y el límite de caracteres se ajustaron en proporción.

La **intro se renderiza según su fondo:**
- **Si la intro usa foto** (fallback o por defecto): PNG único con foto + overlay
  + texto fundidos, como hasta ahora. No se anima.
- **Si la intro usa video** (consiguió clip): se parte en **dos capas** como las
  noticias — `bg_intro.mp4` (el clip) y `over_intro.png` (overlay + texto con
  fondo transparente). El texto va encima del video en movimiento.

En ambos casos el diseño visual de la intro (badge, título DM Serif, bajada) es
idéntico; solo cambia si el overlay se funde con una foto o va transparente sobre
un video.

Las posiciones de la regla y del cuerpo **se calculan**, no se fijan: dependen
de cuántas líneas tenga el título. Con título de 2 líneas y cuerpo de 4, la
última línea base cae en y=1234. Con título de 3 caería en y=1354, dentro de la
safe zone inferior — **por eso el título está limitado a 2 líneas**.

Escapa `&`, `<` y `>` en todo texto antes de insertarlo.

Render de la capa overlay: `rsvg-convert -w 1080 -h 1920 over_N.svg -o over_N.png`
(el PNG conserva la transparencia donde no hay overlay ni texto).

### Estructura de la tarjeta de intro (distinta)

La intro **no usa la plantilla de noticia**: es la portada del video y debe
distinguirse. Mismo fondo + overlay embebido, con el título y la bajada
**traducidos por idioma** (equivalente de marca, no traducción literal).

**El tamaño del título se calcula, no se fija.** Las longitudes varían mucho por
idioma —"L'IA aujourd'hui, sans détour" es mucho más largo que "AI today, no
fluff"— y a tamaño fijo el francés se desborda. Procedimiento:

1. Parte el título en líneas (2 por defecto). Textos de marca por idioma:
   - es: `IA hoy,` / `sin rodeos`
   - en: `AI today,` / `no fluff`
   - pt: `IA hoje,` / `sem rodeios`
   - fr: `L'IA` / `aujourd'hui,` / `sans détour` *(3 líneas, ver abajo)*
2. Mide el ancho real de cada línea en DM Serif Display (renderiza a 100px y
   mide en píxeles, como con los títulos de noticia). Toma la línea más ancha.
3. `tamaño = min(200, floor(900 / ancho_max_a_100px * 100))`. El techo es 200px,
   el ancho útil 900px desde x=90.
4. **Si el tamaño resultante baja de 150px, parte el título en 3 líneas** y
   recalcula. Es lo que hace el francés: a 2 líneas daría 126px (muy pequeño),
   a 3 líneas sube a 170px.

**El badge de fecha va a una altura FIJA**, no relativo al título. Antes se
posicionaba "90px sobre la primera línea del título", pero cuando el título es
alto (2 líneas a 200px) su primera línea sube y el badge se montaba detrás del
texto. Ahora el badge se ancla arriba y el título se centra debajo:

- **Badge** (rect ACCENT + texto Barlow Bold 36px color BASE, `letter-spacing="5"`,
  campo `badge` del idioma): **a 200px del borde superior** (el rect en y=200).
  Fijo en altura, independiente del título.
  - **El ancho del rect NO es fijo — se calcula midiendo el texto.** El texto de
    la fecha varía de largo según el mes ("24 JULIO" es corto, "28 SEPTIEMBRE" o
    "SEPTEMBER 28" son mucho más largos), así que un ancho fijo (como el viejo
    392px) se queda corto y el texto se sale por la derecha. Mide el ancho real
    del texto `badge` en Barlow Bold 36px con `letter-spacing="5"` (mismo método
    que el título: renderiza y mide), y haz:
    `ancho_rect = ancho_texto_medido + 44` (22px de padding a cada lado).
  - El texto va en `x = rect_x + 22`, con `rect_x = 90` (margen izquierdo). Alto
    del rect 62px, texto centrado vertical dentro.
- **Centra el bloque título + regla + bajada** debajo del badge. El centro
  depende de cuántas líneas tenga el título, para que el francés (3 líneas) no
  empuje la bajada fuera de la safe zone:
  - Título de 2 líneas (es/en/pt) → centra en **y≈980**.
  - Título de 3 líneas (fr) → centra en **y≈910** (más arriba, cabe la 3ª línea).
  - Altura del bloque de título = `interlineado × (num_líneas − 1) + tamaño`,
    con interlineado ≈ `tamaño × 1.14`.
  - Primera línea base = `centro − altura_bloque/2 + tamaño × 0.75`.
- **Regla** (rect ACCENT 180×8): 66px bajo la última línea del título.
- **Bajada** (Barlow Bold 44px color ACCENT, `letter-spacing="2"`): 86px bajo la
  regla. Textos:
  - es: `Las 5 noticias del día, en un minuto`
  - en: `The day's 5 AI stories, in one minute`
  - pt: `As 5 notícias do dia, em um minuto`
  - fr: `Les 5 infos du jour, en une minute`

Verifica al final que todo el texto quede entre y=250 y y=1320. Con el centrado
y el cálculo de tamaño, los cuatro idiomas caben: es/en a 200px en 2 líneas, pt
a 170px en 2, fr a 170px en 3.

El reparto es deliberado: el título es eslogan (engagement), la bajada es
información (qué recibe y en cuánto). Única tarjeta con tono de gancho — las
noticias siguen la regla sobria de la sección 2. `DM Serif Display` contrasta a
propósito con la Anton de las noticias; ya está en `setup.sh`.

Después de renderizar, **comprueba que la foto está ahí**.

- **Intro (PNG único):** mide un píxel de la zona alta del PNG final. La zona
  lleva el overlay a solo 0.50 de opacidad, así que si la foto se pintó el píxel
  es distinto del color BASE puro.
- **Noticias (dos capas):** verifica sobre la **capa foto** `bg_N.jpg`, no sobre
  el overlay. La capa foto no lleva overlay, así que basta con que no sea un
  color plano; si `bg_N.jpg` existe y tiene variación de píxeles, la foto está.

```python
from PIL import Image
# intro:
px = Image.open("card_intro.png").convert("RGB").getpixel((540, 250))
base_rgb = tuple(int(BASE[i:i+2], 16) for i in (1, 3, 5))
if max(abs(a - b) for a, b in zip(px, base_rgb)) < 12:
    raise SystemExit("intro: el fondo no se renderizó")
# noticia N: confirmar que la capa foto no está vacía/plana
im = Image.open(f"bg_{n}.jpg").convert("RGB")
if im.getextrema() == ((0,0),(0,0),(0,0)):
    raise SystemExit(f"bg_{n}: la foto no se descargó")
```

Si falla, no sigas: es exactamente el error que produjo un video entero de
tarjetas sin foto. **No des por buena una tarjeta solo porque el archivo existe.**

**Safe zones:** nada de texto arriba de y=250 ni debajo de y=1320.

---

## 6. Ensamblaje

Cada escena es un clip con la duración exacta de su audio. **La intro es
estática; las noticias llevan fondo animado.**

### Intro (estática)

PNG único + audio, sin movimiento:

```bash
ffmpeg -y -loop 1 -framerate 30 -i card_intro.png -i aud_intro.mp3 \
  -c:v libx264 -profile:v high -preset medium -crf 20 -pix_fmt yuv420p \
  -r 30 -g 60 -c:a aac -b:a 192k -ar 48000 \
  -t "$DUR_INTRO" -movflags +faststart clip_intro.mp4
```

### Noticias (fondo animado, overlay quieto)

La foto se anima con un efecto Ken Burns; el overlay+texto va encima, inmóvil.
**El efecto se elige de forma determinista por número de noticia** (no aleatorio,
para poder reproducir y depurar):

| Noticia | Efecto |
|---|---|
| 1 | zoom-in |
| 2 | zoom-out |
| 3 | pan diagonal |
| 4 | zoom-in |
| 5 | zoom-out |

Método validado: `zoompan` con `d=1` y `-frames:v` limitado (el `d=$FRAMES`
clásico es lentísimo). Se escala a 2× antes de animar para evitar el jitter de
zoompan, y se baja a 1080×1920 en el mismo filtro. `FRAMES = round(DUR_N × 30)`.

```bash
FPS=30
FRAMES=$(python3 -c "print(round($DUR_N * $FPS))")

# expresión de zoom según el efecto:
#   zoom-in  : z='min(1+0.18*on/FRAMES,1.18)'  x,y centrados
#   zoom-out : z='max(1.18-0.18*on/FRAMES,1.0)' x,y centrados
#   pan      : z='1.18'  x='(iw-iw/zoom)*(on/FRAMES)'  y='(ih-ih/zoom)*(on/FRAMES)'

# ejemplo zoom-in:
ffmpeg -y -loop 1 -framerate $FPS -t "$DUR_N" -i bg_N.jpg \
       -loop 1 -framerate $FPS -t "$DUR_N" -i over_N.png \
       -i aud_N.mp3 \
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

El movimiento es de 18% a lo largo de toda la escena: perceptible sin marear,
igual que el deep dive. Si en la revisión se ve demasiado, baja el `0.18` a `0.15`.

### Escenas con fondo en video (intro, n2, n4 si hubo clip)

Cuando la escena tiene fondo en **video** (`bg_N.mp4` en vez de `bg_N.jpg`), **no
se aplica Ken Burns** — el clip ya se mueve solo. El clip ya viene recortado a
1080×1920 y con la duración del audio (paso 3). Solo se superpone el overlay y
se añade el audio:

```bash
ffmpeg -y -i bg_N.mp4 -loop 1 -i over_N.png -i aud_N.mp3 \
  -filter_complex "[0:v][1:v]overlay=0:0:format=auto[v]" \
  -map "[v]" -map 2:a -t "$DUR_N" \
  -c:v libx264 -profile:v high -preset medium -crf 20 -pix_fmt yuv420p \
  -r 30 -g 60 -c:a aac -b:a 192k -ar 48000 -movflags +faststart clip_N.mp4
```

Para la **intro con video**: mismo patrón, usando `card_intro` como capa overlay
transparente. Ojo: la intro con video **sí necesita** que su overlay+texto sea
un PNG con transparencia (como las noticias), no el PNG opaco de la intro
estática. Si la intro cayó a foto (sin clip), usa el bloque de intro estática de
arriba con su PNG único.

**Regla de decisión por escena:**
- ¿Existe `bg_N.mp4`? → usa este bloque (video, sin Ken Burns).
- ¿Solo `bg_N.jpg`? → usa el bloque de Ken Burns (foto animada).

Así intro/n2/n4 usan video si lo consiguieron, y foto si cayeron al fallback,
sin ramas especiales: la sola presencia del `.mp4` decide.

### Concatenar

Concatena los clips en orden (intro primero) con `-f concat -c copy` en un
`video_mudo.mp4` (mudo de música, ya trae la voz de cada escena).

### Cama musical continua bajo las noticias

La cama del día se tiende en **una sola pasada** sobre el video concatenado,
para que fluya continua bajo las 5 noticias sin cortes entre escenas. La intro
**no** lleva cama (ya tiene su cabezote), así que la música entra cuando termina
la intro y corre hasta el final.

```bash
# cama del día, misma rotación que la paleta: bed_(día % 5)_norm.mp3
IDX=$(( 10#$(date +%j) % 5 ))
BED="audio/bed_${IDX}_norm.mp3"

# RECALCULAR DUR_INTRO aquí mismo, midiendo el audio de la intro. NO confiar en la
# variable de un bloque anterior: si la rutina corre la producción en fragmentos,
# las variables de shell no persisten entre bloques y DUR_INTRO llegaría vacía,
# lo que rompería el cálculo y dejaría el video SIN música (bug observado).
DUR_INTRO=$(ffprobe -v error -show_entries format=duration -of csv=p=0 aud_intro.mp3)

# salvaguardas: si algo falla, abortar con mensaje claro en vez de producir video mudo
if [ -z "$DUR_INTRO" ]; then echo "ERROR: no pude medir aud_intro.mp3 para la cama musical"; exit 1; fi
if [ ! -f "$BED" ]; then echo "ERROR: no existe la cama $BED"; exit 1; fi

DUR_TOTAL=$(ffprobe -v error -show_entries format=duration -of csv=p=0 video_mudo.mp4)
# la música empieza al terminar la intro y termina con el video
MUSIC_START="$DUR_INTRO"
MUSIC_DUR=$(python3 -c "print(round($DUR_TOTAL-$DUR_INTRO,3))")
FADE_OUT=$(python3 -c "print(round($MUSIC_DUR-2,3))")

ffmpeg -y -i video_mudo.mp4 \
  -stream_loop -1 -i "$BED" \
  -filter_complex "\
    [1:a]atrim=0:$MUSIC_DUR,volume=-14dB,\
afade=t=in:st=0:d=1.5,afade=t=out:st=$FADE_OUT:d=2,\
adelay=$(python3 -c "print(int($MUSIC_START*1000))")|$(python3 -c "print(int($MUSIC_START*1000))")[m];\
    [0:a][m]amix=inputs=2:duration=first:normalize=0[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -movflags +faststart "$NOMBRE"

# verificación post-mezcla: confirmar que el audio final tiene música bajo las noticias
NIVEL=$(ffmpeg -hide_banner -ss $(python3 -c "print(round($DUR_INTRO+3,1))") -t 3 -i "$NOMBRE" -af volumedetect -f null /dev/null 2>&1 | grep -o 'mean_volume: [-0-9.]*' | head -1)
echo "nivel de audio en zona de noticias: $NIVEL (debe haber señal, no silencio)"
```

- Cama a **−14 dB**. Los beds ya vienen normalizados a −20 LUFS, así que este
  valor los deja audibles bajo la voz sin taparla. **Ojo: no bajes de −18 dB o
  la música desaparece** — con −22 dB la voz real de edge-tts la enmascara por
  completo (bug corregido: antes estaba en −22 y no se oía).
- `-stream_loop -1`: el bed de 30s se repite hasta cubrir las noticias.
- `adelay` = duración de la intro, para que la música no pise el cabezote.
- Fade in de 1.5s al entrar, fade out de 2s al terminar el video.
- `-c:v copy`: el video no se recodifica, solo se reemplaza la pista de audio.
- Verifica de oído en la primera corrida: la música debe sentirse presente pero
  la voz siempre por encima. Si tapa la voz, sube a −16; si no se oye, baja a −12.

**Nombre del archivo final: `YYYYMMDD.mp4`** — año (4 dígitos) + mes (2 dígitos)
+ día (2 dígitos), sin separadores. Para hoy: `20260728.mp4`. Este formato
ordena los archivos cronológicamente solos en la carpeta.

```bash
NOMBRE=$(date +%Y%m%d).mp4    # 20260728.mp4
```

Cuando se reactive el multi-idioma, el nombre lleva el código de idioma como
sufijo: `20260728_en.mp4`, `20260728_es.mp4`, etc. Por ahora, con solo inglés,
basta `YYYYMMDD.mp4`.

---

## 7. Subida a YouTube

Sube el `YYYYMMDD.mp4` al canal con la YouTube Data API v3, usando las
credenciales del `.env` (`YT_CLIENT_ID`, `YT_CLIENT_SECRET`, `YT_REFRESH_TOKEN`,
`YT_PRIVACY_STATUS`). El título, la descripción y los tags son los del
manifiesto (los de metadatos de YouTube, con gancho — no los de la tarjeta).

```python
import os, json
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload

creds = Credentials(
    None,
    refresh_token=os.environ["YT_REFRESH_TOKEN"],
    client_id=os.environ["YT_CLIENT_ID"],
    client_secret=os.environ["YT_CLIENT_SECRET"],
    token_uri="https://oauth2.googleapis.com/token",
)
yt = build("youtube", "v3", credentials=creds)

# --- Hora de publicación: 4:00am hora de Nueva York, fija ---
# zoneinfo maneja EDT/EST solo según la fecha (no hay que tocar nada por horario de verano).
NY = ZoneInfo("America/New_York")
HORA_NY = 4
ahora_ny = datetime.now(NY)
objetivo = ahora_ny.replace(hour=HORA_NY, minute=0, second=0, microsecond=0)
if objetivo <= ahora_ny:          # si ya pasaron las 4am hoy, programa para mañana
    objetivo += timedelta(days=1)
publish_at = objetivo.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

meta = json.load(open("meta_en.json"))   # título, descripción, tags del manifiesto
body = {
    "snippet": {
        "title": meta["titulo_video"],
        "description": meta["descripcion"],
        "tags": meta["tags"],
        "categoryId": "28",               # Science & Technology
    },
    "status": {
        # Para programar, el video DEBE subirse como 'private' + publishAt.
        # YouTube lo hace público solo a esa hora. (.env debe estar en private.)
        "privacyStatus": "private",
        "publishAt": publish_at,
        "selfDeclaredMadeForKids": False,
        # "No" a la divulgación de contenido alterado/sintético (la pregunta "AI use").
        # Correcto para este contenido: voz sintética + stock real de Pexels NO cuenta
        # como A/S — esa divulgación es solo para contenido realista que engañe sobre
        # algo real (deepfakes, eventos falsos). Narrar noticias reales con voz IA no
        # lo requiere. Ver política de A/S content de YouTube.
        "containsSyntheticMedia": False,
    },
}
req = yt.videos().insert(
    part="snippet,status",
    body=body,
    media_body=MediaFileUpload(os.environ["NOMBRE"], resumable=True),
)
resp = req.execute()
vid = resp["id"]
print(f"VIDEO_URL=https://www.youtube.com/watch?v={vid}")
print(f"STUDIO_URL=https://studio.youtube.com/video/{vid}/edit")
print(f"PUBLISH_AT={publish_at}  (4:00am America/New_York)")
```

- `categoryId: 28` es Ciencia y Tecnología, la correcta para el nicho.
- La subida es `resumable`, así que un corte de red no la deja a medias.
- **Programación:** el video sube ahora (2am) pero se publica solo a las **4:00am
  hora de Nueva York**. Como la rutina corre de madrugada, ese mismo día a las 4am
  NY se hace público. `zoneinfo` ajusta EDT/EST automáticamente según la fecha.
  **Nota:** el margen entre la corrida (2am) y la publicación (4am) es de 2 horas. Si la rutina tardara más de dos horas y pasara de las 4am, la lógica
  programa para el día siguiente (no intenta una hora ya pasada). En una corrida
  normal esto no ocurre, pero si ves la publicación caer al día siguiente, esa es
  la causa: la corrida se pasó de las 4am.
- **`publishAt` exige `privacyStatus: private`.** Si el `.env` estuviera en
  `public`, el video se publicaría de inmediato y `publishAt` se ignora. Por eso
  el código fija `private` explícito, sin importar el `.env`.
- **Captura el `VIDEO_URL` que imprime**: es lo que me tienes que reportar, junto
  con la hora programada.
- Si la subida falla (cuota, auth, red), no te quedes callado: dímelo con el
  error exacto y deja el `YYYYMMDD.mp4` disponible para que yo lo suba a mano.

**Sobre el token:** si la API responde con error de credenciales
(`invalid_grant`), el refresh token expiró — pasa si la app OAuth quedó en modo
"prueba". Avísame para regenerarlo; no es un fallo del video.

---

## 8. Entrega

Cuando el video esté subido, preséntame:

1. **La URL del video** (`https://www.youtube.com/watch?v=...`) bien visible al
   principio, para revisarlo. Y la de Studio por si quiero editarlo.
2. **La hora a la que se publicará** (4:00am NY de ese día).
3. El `YYYYMMDD.mp4` (por si quiero el archivo)
4. El `manifiesto.json`
5. Un resumen corto: título y descripción que quedaron publicados, duración
   total del video, y qué conceptos de fondo se descartaron si hubo. No listes
   la foto de cada escena.

El video queda **programado como privado** y YouTube lo hace público solo a las
4:00am NY. Al revisarlo por la URL antes de esa hora, lo verás como privado —es
lo esperado. Si algo está mal, tienes hasta las 4am para cambiarlo en Studio.

---

## Reglas de fallo

- Si una noticia no se puede verificar en fuente primaria, descártala y busca otra.
- Si el audio de una escena pasa de 13s, acorta el guion y regenera esa escena.
- Si un título o cuerpo excede los límites de caracteres, reescríbelo. No lo
  renderices confiando en que "quizá quepa".
- Si algo falla de forma irrecuperable, entrégame lo que alcanzaste y dime
  exactamente en qué paso se rompió. No inventes contenido para completar.

---

## Al terminar

Dame tu **top 3 de sugerencias** para mejorar, priorizadas — solo las tres más
importantes, no una lista larga. El pipeline ya está bastante pulido, así que
enfócate en lo que de verdad movería la aguja.

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
