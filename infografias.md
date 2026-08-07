# Infografías animadas · Canal senior formato largo (1920x1080)

Son la señal más fuerte de "trabajo transformativo" ante el YPP: nadie
llama presentación de diapositivas a un video con visualizaciones propias.

**Meta: 2-4 infografías por video de 10 minutos** (~60-120 s de metraje).

---

## REGLA CERO · Los datos

1. **Todo número en pantalla necesita fuente citable**, y la fuente va en
   la descripción del video. Sin excepción.
2. Si no hay dato sólido, el gráfico es **cualitativo**: diagramas,
   flujos, secuencias temporales. Sin cifras inventadas para "rellenar".
3. Nunca visualizar un dato como si fuera universal cuando es de un
   estudio concreto. En pantalla: "In one 10-year study of 5,000 adults".
4. Prohibido graficar pronósticos individuales ("your risk drops 40%").
   Eso es afirmación clínica, no explicación.

---

## DOS TÉCNICAS DE IMPLEMENTACIÓN

### Técnica A · Revelado (barata)
Un solo SVG con el estado final → PNG → ffmpeg lo revela progresivamente
con `crop`/`overlay` y expresiones de tiempo.
Sirve para: barras, tablas, arrays de iconos, diagramas de flujo.
Costo: ~1 render de SVG por infografía.

### Técnica B · Secuencia de frames (flexible)
Python genera un SVG por frame con valores interpolados → rsvg-convert →
ffmpeg encodea la secuencia.
Sirve para: contadores, líneas que se dibujan, manecillas de reloj,
arcos, cualquier valor que cambie de forma continua.
Costo: ~300 renders para 10 s a 30 fps (~15 s de proceso). Aceptable.

```python
# Esqueleto de la Técnica B
import subprocess, os
FPS, DUR = 30, 10
N = FPS * DUR

def ease_out_cubic(t):          # 0..1 → 0..1
    return 1 - (1 - t) ** 3     # NUNCA usar interpolación lineal:
                                # se ve robótica. El easing es lo que
                                # hace que parezca diseñado.

os.makedirs("frames", exist_ok=True)
for i in range(N):
    p = ease_out_cubic(min(i / (N * 0.6), 1))   # anima en el 60% inicial
    svg = plantilla_svg(progreso=p)             # tu función
    open(f"frames/f{i:04d}.svg", "w").write(svg)
    subprocess.run(["rsvg-convert", "-w", "1920", "-h", "1080",
                    f"frames/f{i:04d}.svg", "-o", f"frames/f{i:04d}.png"],
                   check=True)

subprocess.run(["ffmpeg", "-y", "-framerate", str(FPS),
                "-i", "frames/f%04d.png",
                "-c:v", "libx264", "-crf", "20", "-pix_fmt", "yuv420p",
                "info.mp4"], check=True)
```

---

## LOS 12 TIPOS

### 1. Barras que crecen · Técnica A o B ●
**Cuándo.** Comparar magnitudes entre grupos o edades. El más versátil.
Ej.: masa muscular a los 40 / 60 / 80.
**Cuándo NO.** Más de 5 barras — ilegible en TV.
**Implementación (A).** SVG con las barras completas; ffmpeg revela de
abajo hacia arriba con un `crop` animado, o superpone un rectángulo del
color de fondo que sube.
```bash
ffmpeg -y -loop 1 -t $DUR -i barras.png \
 -filter_complex "color=c=0x1C1A17:s=1920x1080:d=$DUR[m];\
   [0:v][m]overlay=0:'0-1080+1080*min(t/2.5,1)'" \
 -frames:v $FRAMES -c:v libx264 -crf 20 out.mp4
```
Escalonar las barras (cada una arranca 0.2 s después) se ve mucho mejor;
eso pide Técnica B.

---

### 2. Línea de tiempo horizontal · Técnica A ●
**Cuándo.** Arquitectura *First Hour* y *Decade Timeline*. Una línea con
hitos que aparecen uno a uno mientras la voz los nombra.
**Cuándo NO.** Si los hitos no tienen orden real; forzar cronología
donde no la hay confunde.
**Implementación.** SVG con la línea y todos los hitos; overlay de cada
hito con `enable='gte(t,N)'` sincronizado con la narración.

---

### 3. Diagrama del cuerpo con zonas · Técnica A ●●
**Cuándo.** El tipo estrella del canal: explicar mecanismo. Silueta
neutra, y las zonas se iluminan mientras la voz explica qué pasa en cada
una.
**Cuándo NO.** Nunca marcar zonas como "dañadas" o "enfermas" — eso es
diagnóstico visual. Solo "aquí ocurre X".
**Implementación.** SVG de silueta + capas por zona con `opacity`
animada. Con Técnica A: una PNG por zona, superpuestas con `enable` y
un fundido corto (`fade=in:st=N:d=0.4`).
**Nota.** La silueta hay que dibujarla una vez (SVG propio, reutilizable
con distintas zonas). Es la inversión que más rinde: sirve para decenas
de videos y es 100% original.

---

### 4. Tabla comparativa que se llena · Técnica A ●
**Cuándo.** *Myth vs Reality*, *Then vs Now*. Dos columnas, filas que
aparecen una por una.
**Cuándo NO.** Más de 4 filas en pantalla a la vez.
**Implementación.** SVG con la tabla completa; overlay fila por fila con
`enable='gte(t,N)'`.

---

### 5. Contador numérico · Técnica B ●●
**Cuándo.** Una sola cifra con peso, subiendo hasta su valor.
Ej.: "6,000,000 adults in Colombia live with hypertension".
**Cuándo NO.** Más de un contador por video — pierde impacto.
**Implementación.** Técnica B obligatoria (el texto cambia por frame).
Formatear con separador de miles y easing out.

---

### 6. Arco o dona de proporción · Técnica B ●●
**Cuándo.** Proporciones simples: "8 de cada 10". El arco se dibuja
hasta su porcentaje.
**Cuándo NO.** Comparar más de 3 categorías — usa barras.
**Implementación.** SVG `<circle>` con `stroke-dasharray` y
`stroke-dashoffset` interpolado por frame.
```python
CIRC = 2 * 3.14159 * R
offset = CIRC * (1 - pct * p)   # p = progreso con easing
```

---

### 7. Línea que se dibuja · Técnica B ●●
**Cuándo.** Tendencias a lo largo de décadas. Ej.: cómo cambia la
calidad del sueño de los 40 a los 80.
**Cuándo NO.** Con menos de 4 puntos — no es una tendencia, es una
comparación (usa barras).
**Implementación.** `<path>` con `stroke-dasharray` = longitud total y
`stroke-dashoffset` decreciendo. Mismo truco que el arco.

---

### 8. Array de iconos (waffle) · Técnica A ●
**Cuándo.** Humanizar una proporción: 10 siluetas, 8 se colorean.
Comunica "8 de cada 10" mucho mejor que un porcentaje para este público.
**Cuándo NO.** Con proporciones que no sean sobre X de cada 10 o 100.
**Implementación.** SVG con las 10 siluetas en gris + capa de las
coloreadas; overlay una por una con `enable`.

---

### 9. Comparador antes/después · Técnica A ●●
**Cuándo.** Dos estados del mismo objeto o escena, con una línea
divisoria que se desplaza revelando.
**Cuándo NO.** Si las dos imágenes no están alineadas — el efecto exige
mismo encuadre.
**Implementación.** Igual que el split-screen wipe de la gramática
visual (`drawbox` con `x` animado), pero sobre dos gráficos SVG.

---

### 10. Cadena causa-efecto · Técnica A ●●
**Cuándo.** Explicar un mecanismo en pasos: "menos luz → menos
melatonina → sueño fragmentado". Es el formato más didáctico y el que
mejor encaja con la regla de explicar en vez de prescribir.
**Cuándo NO.** Cadenas de más de 4 eslabones.
**Implementación.** SVG con cajas y flechas; cada eslabón aparece con
`enable` + `fade`. Las flechas pueden dibujarse con la técnica del
`stroke-dashoffset` si quieres que "avancen".

---

### 11. Pila que se acumula · Técnica B ●●
**Cuándo.** Ideas de acumulación: efecto compuesto de un hábito a lo
largo de semanas o años.
**Cuándo NO.** Si la acumulación es una promesa de resultado
individual — eso es afirmación clínica.
**Implementación.** Rectángulos apilándose con easing escalonado.

---

### 12. Reloj de 24 horas · Técnica B ●●
**Cuándo.** Temas de sueño y ritmo circadiano — que son 20 de tus 80
temas. Un dial de 24 h con zonas coloreadas y una manecilla que gira.
**Cuándo NO.** Fuera de temas cronobiológicos; es muy específico.
**Implementación.** Rotación de la manecilla por frame:
`transform="rotate({angulo} {cx} {cy})"`, ángulo = 360 * hora/24.

---

## Mapeo arquitectura → infografía sugerida

| Arquitectura | Infografías naturales |
|---|---|
| Study That Surprised | 1 barras · 5 contador · 8 waffle |
| What Doctors Are Rethinking | 4 tabla · 10 cadena |
| Two People Same Age | 9 comparador · 4 tabla |
| Then vs Now | 9 comparador · 7 línea · 4 tabla |
| Myth vs Reality | 4 tabla · 8 waffle |
| First Hour | 2 timeline · 3 cuerpo · 12 reloj |
| Decade Timeline | 2 timeline · 7 línea · 1 barras |
| 5 Signs | 2 timeline · 3 cuerpo · 10 cadena |
| 4 Habits | 1 barras · 11 pila · 4 tabla |

---

## Reglas de estilo (coherencia de marca)

- Paleta: fondo `#1C1A17`, acento cálido `#C9764A`, acento frío
  `#5A6B8C`, texto `#FFFFFF`, secundario `#8A857D`.
- Tipografía: **Lora** para títulos de la infografía, **Nunito** para
  etiquetas y cifras.
- Tamaño mínimo de etiqueta: **32 px** a 1080p. El público ve en TV a
  distancia; por debajo de eso no se lee.
- Máximo **5 elementos** por infografía (barras, filas, hitos).
- Cada infografía dura **20-40 s** en pantalla, no menos: el senior
  necesita tiempo para leer.
- **Easing siempre**, nunca interpolación lineal.
- Variar el tipo entre videos consecutivos: dos videos seguidos con
  gráfico de barras es exactamente el patrón que se detecta.

---

## Fases

**Fase 1** (Técnica A, sin dependencias nuevas): 1, 2, 4, 8, 10.
**Fase 2** (Técnica B, requiere el loop de frames): 5, 6, 7, 12.
**Fase 3** (requieren activo SVG propio): 3 silueta del cuerpo, 9, 11.

La silueta del cuerpo (tipo 3) es la que más rinde a largo plazo: se
dibuja una vez y sirve para decenas de videos.
