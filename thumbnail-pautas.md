# Pautas de thumbnail · Canal senior formato largo
# Bloque para el pipeline: el agente ESCRIBE el prompt de GPT Image 2
# para cada video. Esto le da reglas duras + plantilla por arquitectura
# + libertad en el detalle.

## Cómo usar este bloque (instrucción al agente)

Vas a redactar el prompt en INGLÉS para GPT Image 2 que genera el
thumbnail del video de hoy. Sigue las REGLAS DURAS siempre. Toma la
PLANTILLA DE COMPOSICIÓN de la arquitectura elegida. Llena tú los
detalles (descripción física, escena, texto-gancho) según el tema.
Genera máximo 2 imágenes (ver VERIFICACIÓN).

---

## REGLAS DURAS (aplican a todo thumbnail, sin excepción)

1. **Una sola idea.** Un foco. Nada de tres elementos compitiendo.
2. **Texto de 1 a 5 palabras**, en INGLÉS, MAYÚSCULAS, condensed sans-serif
   pesada, blanco con contorno oscuro grueso y sombra. Perfectamente
   escrito. Es un GANCHO DE CURIOSIDAD (una pregunta o una promesa), no
   una etiqueta descriptiva.
3. **El texto va en una franja (arriba o abajo) que NO tapa ninguna cara.**
   La cara es el imán de clics; el texto no compite con ella.
4. **Contraste de dos zonas.** Cálido vs frío, luz vs sombra, o sujeto
   iluminado sobre fondo oscuro. Debe leerse la tensión en <1 segundo.
5. **Si hay una cara, el sujeto MIRA hacia el elemento de interés**
   (el otro sujeto, el objeto, el texto), no a la cámara. Crea un vector
   que dirige el ojo del espectador.
6. **Emoción FÍSICA y concreta**, no vaga. "Dolor de espalda con mueca",
   "energía al caminar" — no "triste" ni "feliz" a secas.
7. **Si hay dos personas comparadas, declara explícitamente** si son la
   MISMA persona en dos estados o dos personas distintas. Nunca ambiguo.
8. **Descripción física completa de cada persona**: edad, pelo, barba,
   ropa, postura, expresión, dirección de mirada. Sin huecos: donde no
   describes, el modelo improvisa y ahí salen las caras deformes.
9. **Equilibrio de peso visual** entre las dos zonas (tamaño, cercanía).
10. **Emoción cálida/digna, NUNCA shock ni gritos.** El público senior
    lee la cara sobreexcitada como estafa. Vitalidad, descubrimiento,
    malestar creíble — sí. Ojos desorbitados y bocas abiertas — no.
11. **Acabado premium, colores saturados pero realistas.** "Photorealistic,
    natural skin texture, anatomically correct symmetrical faces, five
    natural fingers per hand, no distortion." Llamativo, no caricaturesco.
12. Cerrar SIEMPRE con: "Only the text \"[EL TEXTO]\" appears. No other
    text, no watermark, no logos."
13. Tamaño de generación 1536x1024; luego se recorta a 1280x720 (YouTube).
14. Margen de seguridad: ni texto ni caras tocando los bordes.

---

## PLANTILLAS DE COMPOSICIÓN POR ARQUITECTURA

El agente toma el esqueleto y llena {lo que va entre llaves}.

### Two People Same Age → split de dos zonas, misma persona
"Split composition. Both figures are clearly the SAME person: identical
face, {pelo/barba}, so the viewer reads one person living two outcomes.
LEFT: {estado positivo, luz cálida, mirada hacia la derecha}. RIGHT: the
exact same person, {estado negativo con emoción física concreta, luz
fría}, framed slightly larger for balance. Hard warm-vs-cold vertical
divide. {Flecha amarilla del lado bueno al malo, opcional}."

### Then vs Now → split temporal
"Split composition contrasting {joven/antes} on the left versus
{mayor/ahora} on the right, {mismo sujeto o mismo objeto}. {Vector de
mirada o flecha cruzando}. Warm-vs-cool divide."

### Myth vs Reality → símbolo desmentido
"A single older adult with an expression of {realización/'not what you
think'}, looking toward a {símbolo/objeto} that is {tachado con una X
roja / partido}. Two-zone contrast."

### Study That Surprised → cara + hueco para dato
"Close-up of {persona mayor} with an expression of quiet surprise or
discovery, looking toward {espacio donde iría un número grande}. Warm
subject over darker background."

### First Hour / Decade Timeline → sujeto + marca temporal
"{Persona mayor} in {momento del día / acción}, with a large {reloj /
número de edad} as the contrasting element, {sujeto mira hacia él}.
Warm light on the subject."

### 5 Signs / 4 Habits → cara atenta + espacio para número
"{Persona mayor} with an attentive, slightly concerned but dignified
expression, looking toward {espacio superior donde iría '5 SIGNS'}.
Object of daily life ({taza/zapatos/cama}) as secondary element."

### What Doctors Are Rethinking → cara pensativa + contraste
"Thoughtful {persona mayor}, hand near chin, two-zone lighting split
suggesting 'old idea vs new understanding'. Subject looks toward the
brighter side."

---

## LO QUE EL AGENTE LLENA LIBREMENTE (por tema)

- La descripción física concreta de la(s) persona(s).
- La escena/entorno de cada zona.
- El TEXTO-GANCHO (1-5 palabras) — debe crear curiosidad y alinear con
  el título del video sin prometer de más. Ej.: "WHY IS HE TIRED?",
  "AFTER 60", "MOST PEOPLE MISS THIS", "THE REAL REASON", "AT 3 AM".
- Si usa flecha, X roja u otro señalador.

---

## VERIFICACIÓN (regla dura de costo)

Tras generar, revisa: (a) el texto está bien escrito y sin deformación;
(b) las caras no tienen deformidades evidentes (ojos, manos). Si algo
falla, regenera UNA vez. **Máximo 2 generaciones.** Si la segunda también
falla, usa la mejor de las dos y continúa. NUNCA bloquear la publicación
por un thumbnail imperfecto.

---

## Aprendizajes de calibración (referencia de por qué existen las reglas)

- Dos personas "parecidas pero no iguales" es el peor resultado: o son
  claramente la misma o claramente distintas. (→ regla 7)
- El texto sobre la cara mata el CTR. Franja limpia. (→ regla 3)
- Mirada a cámara desperdicia el vector; mirar al elemento arrastra el
  ojo por todo el thumbnail. (→ regla 5)
- "Triste" se lee débil; "dolor físico con mueca" se lee fuerte y
  específico. (→ regla 6)
