# Gramática visual · Canal senior formato largo (1920x1080)

15 tratamientos. La rutina elige 3-4 por acto y los rota. Ninguno se
repite más de dos veces seguidas.

Convenciones usadas abajo:
- `DUR` = duración de la escena en segundos · `FPS=30` · `FRAMES=$((DUR*FPS))`
- Método Ken Burns validado: `zoompan ... d=1` + `-frames:v $FRAMES`
  (NO usar `d=$FRAMES`, se cuelga).
- Pre-escalado 1.5x antes de zoompan para evitar jitter de borde de píxel.
- El texto SIEMPRE va en capa aparte (PNG con alfa) compuesta con
  `overlay=0:0` sobre la imagen ya animada. Nunca se anima junto.

Dificultad: ● fácil · ●● media · ●●● alta

---

## FAMILIA 1 · Movimiento sobre imagen fija

### 1. Ken Burns direccional ●
**Qué es.** Zoom o desplazamiento lento y continuo sobre una foto.
Cuatro variantes: zoom-in, zoom-out, pan lateral, pan diagonal.

**Cuándo funciona.** Es el caballo de batalla: sirve para casi cualquier
plano narrativo. Zoom-in cuando la voz *entra* en un tema o intensifica;
zoom-out cuando *concluye* o abre contexto; pan lateral para recorrer una
escena amplia (paisaje, grupo); pan diagonal para dar inquietud suave.
**Cuándo NO.** Si la foto ya tiene movimiento implícito fuerte, o si es
un primerísimo plano donde el zoom deforma.

**Implementación.**
```bash
# zoom-in (intensidad 0.12 = sutil, adecuada para público senior)
ffmpeg -y -loop 1 -framerate $FPS -t $DUR -i img.jpg \
 -filter_complex "scale=2880:1620,zoompan=z='min(1+0.12*on/$FRAMES\,1.12)':d=1:x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1920x1080,setsar=1" \
 -frames:v $FRAMES -c:v libx264 -preset veryfast -crf 20 -pix_fmt yuv420p out.mp4

# zoom-out : z='max(1.12-0.12*on/$FRAMES\,1.0)'
# pan lateral : z='1.12' x='(iw-iw/zoom)*(on/$FRAMES)' y='ih/2-(ih/zoom/2)'
# pan diagonal: z='1.12' x='(iw-iw/zoom)*(on/$FRAMES)' y='(ih-ih/zoom)*(on/$FRAMES)'
```

---

### 2. Parallax de dos capas ●●●
**Qué es.** Sujeto y fondo se separan y se mueven a distinta velocidad,
simulando profundidad 3D a partir de una sola foto.

**Cuándo funciona.** Momentos de alto impacto: apertura del video, cierre
de un acto, la frase-tesis. Es el tratamiento que más "producción" aparenta.
Usar 1-2 veces por video como máximo — su fuerza está en la escasez.
**Cuándo NO.** Fotos sin un sujeto claramente recortable (multitudes,
paisajes puros, texturas). El recorte fallido se nota muchísimo.

**Implementación.** Requiere máscara alfa, que Pexels no provee.
```bash
# 1) Segmentar sujeto (rembg, una vez por asset)
rembg i img.jpg sujeto.png          # sujeto con alfa
# 2) Fondo: rellenar el hueco (inpaint) o usar la original como fondo
# 3) Animar cada capa a distinta velocidad y componer
ffmpeg -y -loop 1 -t $DUR -i img.jpg -loop 1 -t $DUR -i sujeto.png \
 -filter_complex "\
  [0:v]scale=2400:1350,zoompan=z='1.05+0.03*on/$FRAMES':d=1:s=1920x1080,setsar=1[bg];\
  [1:v]scale=2400:1350,zoompan=z='1.05+0.09*on/$FRAMES':d=1:s=1920x1080,setsar=1[fg];\
  [bg][fg]overlay=0:0" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```
**Nota.** `rembg` se instala vía pip y corre en CPU. Añade ~5-10 s por
imagen. Recomendado dejarlo para fase dos.

---

### 3. Push-in con desenfoque de entrada ●●
**Qué es.** La imagen arranca ligeramente desenfocada y se enfoca mientras
la cámara se acerca. Simula el ojo ajustando.

**Cuándo funciona.** Al **entrar a un acto nuevo** o al revelar un
concepto. Da sensación de "aterrizar" en una idea. Excelente después de
una disolvencia entre capítulos.
**Cuándo NO.** En planos informativos rápidos: el desenfoque inicial
roba legibilidad si hay texto encima desde el primer frame.

**Implementación.**
```bash
# El blur decae en el primer ~25% de la escena
ffmpeg -y -loop 1 -framerate $FPS -t $DUR -i img.jpg \
 -filter_complex "scale=2880:1620,\
  zoompan=z='min(1.02+0.10*on/$FRAMES\,1.12)':d=1:x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1920x1080,\
  gblur=sigma='max(0\,8-32*t/$DUR)':eval=frame,setsar=1" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```

---

### 4. Deriva con viñeta respirante ●
**Qué es.** Plano casi quieto (movimiento apenas perceptible) con una
viñeta oscura que se abre y cierra muy lentamente.

**Cuándo funciona.** Momentos de **reflexión**: cuando la voz dice la
frase más importante del acto, o en pasajes emocionales. El movimiento
mínimo obliga al espectador a escuchar en vez de mirar.
**Cuándo NO.** En secuencias de ritmo alto o listas — mata la energía.

**Implementación.**
```bash
ffmpeg -y -loop 1 -framerate $FPS -t $DUR -i img.jpg \
 -filter_complex "scale=2400:1350,\
  zoompan=z='1.04+0.02*on/$FRAMES':d=1:x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':s=1920x1080,\
  vignette=angle='PI/5+PI/40*sin(2*PI*t/8)':eval=frame,setsar=1" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```
Ciclo de respiración de 8 s: lo bastante lento para no notarse como efecto.

---

## FAMILIA 2 · Composición

### 5. Split-screen comparativo ●●
**Qué es.** Dos imágenes lado a lado, con una línea divisoria fina.

**Cuándo funciona.** Nativo para las arquitecturas de contraste:
*Two People Same Age*, *Then vs Now*, *Myth vs Reality*. Úsalo justo
cuando la voz nombra los dos términos de la comparación.
**Cuándo NO.** Si los dos assets tienen encuadres muy distintos (uno
plano cerrado, otro general) — el ojo no compara, se confunde.

**Implementación.**
```bash
# Dos mitades de 960x1080 + línea divisoria de 4 px
ffmpeg -y -loop 1 -t $DUR -i izq.jpg -loop 1 -t $DUR -i der.jpg \
 -filter_complex "\
  [0:v]scale=960:1080:force_original_aspect_ratio=increase,crop=960:1080[l];\
  [1:v]scale=960:1080:force_original_aspect_ratio=increase,crop=960:1080[r];\
  [l][r]hstack=inputs=2,drawbox=x=958:y=0:w=4:h=1080:color=white@0.9:t=fill,setsar=1" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```
**Variante "wipe":** la línea divisoria se desplaza revelando un lado.
Cambia `x=958` por `x='960*t/$DUR'` con `eval=frame`.

---

### 6. Tarjeta con margen ●
**Qué es.** La imagen NO llena la pantalla: flota sobre fondo de color
sólido, con margen generoso y sombra suave.

**Cuándo funciona.** Como **respiro de ritmo**. Si llevas 4-5 planos
full-bleed seguidos, una tarjeta cambia por completo la respiración del
video. También para citas, datos de estudios y momentos "de apunte".
**Cuándo NO.** Dos tarjetas seguidas — pierde el efecto de contraste.

**Implementación.**
```bash
# Imagen a 1440x810 centrada sobre fondo crema, con sombra
ffmpeg -y -loop 1 -t $DUR -i img.jpg \
 -filter_complex "\
  color=c=0x1C1A17:s=1920x1080:d=$DUR[bgc];\
  [0:v]scale=1440:810:force_original_aspect_ratio=increase,crop=1440:810,\
      zoompan=z='1.03+0.02*on/$FRAMES':d=1:s=1440x810,setsar=1[card];\
  color=c=black@0.5:s=1460x830:d=$DUR,boxblur=20[sh];\
  [bgc][sh]overlay=(W-w)/2:(H-h)/2+8[b1];\
  [b1][card]overlay=(W-w)/2:(H-h)/2" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```

---

### 7. Texto-izquierda / imagen-derecha ●●
**Qué es.** Pantalla partida verticalmente: a un lado la afirmación clave
en tipografía grande, al otro la imagen.

**Cuándo funciona.** Es el tratamiento **más legible en televisión**, que
es donde te ve el público senior. Ideal para la frase-tesis de cada acto y
para datos numéricos. También cuando el asset disponible es mediocre: al
ocupar media pantalla, se nota menos.
**Cuándo NO.** Si el texto es largo — este tratamiento pide 8-12 palabras
máximo.

**Implementación.**
```bash
# Panel de texto (SVG 960x1080) → PNG, y la imagen a la derecha
rsvg-convert panel.svg -o panel.png     # fondo sólido + texto Lora/Nunito
ffmpeg -y -loop 1 -t $DUR -i img.jpg -loop 1 -t $DUR -i panel.png \
 -filter_complex "\
  color=c=0x1C1A17:s=1920x1080:d=$DUR[bg];\
  [0:v]scale=960:1080:force_original_aspect_ratio=increase,crop=960:1080,\
      zoompan=z='1.03+0.02*on/$FRAMES':d=1:s=960x1080,setsar=1[im];\
  [bg][im]overlay=960:0[b1];\
  [b1][1:v]overlay=0:0" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```
Alternar el lado (texto a la derecha) entre usos para no fijar patrón.

---

### 8. Mosaico progresivo ●●
**Qué es.** Tres o cuatro imágenes aparecen en cuadrícula, una por una,
sincronizadas con la enumeración de la voz.

**Cuándo funciona.** Arquitecturas de cuenta regresiva (*5 Signs*,
*4 Habits*): el mosaico se va llenando mientras la narradora enumera, y
al final quedan todas visibles como resumen. Muy satisfactorio de ver.
**Cuándo NO.** Más de 4 celdas: cada imagen queda demasiado pequeña para
TV.

**Implementación.**
```bash
# 4 celdas de 960x540, apareciendo cada 2.5 s
ffmpeg -y -loop 1 -t $DUR -i a.jpg -loop 1 -t $DUR -i b.jpg \
      -loop 1 -t $DUR -i c.jpg -loop 1 -t $DUR -i d.jpg \
 -filter_complex "\
  color=c=0x1C1A17:s=1920x1080:d=$DUR[bg];\
  [0:v]scale=960:540:force_original_aspect_ratio=increase,crop=960:540[a];\
  [1:v]scale=960:540:force_original_aspect_ratio=increase,crop=960:540[b];\
  [2:v]scale=960:540:force_original_aspect_ratio=increase,crop=960:540[c];\
  [3:v]scale=960:540:force_original_aspect_ratio=increase,crop=960:540[d];\
  [bg][a]overlay=0:0:enable='gte(t,0)'[o1];\
  [o1][b]overlay=960:0:enable='gte(t,2.5)'[o2];\
  [o2][c]overlay=0:540:enable='gte(t,5)'[o3];\
  [o3][d]overlay=960:540:enable='gte(t,7.5)',setsar=1" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```

---

## FAMILIA 3 · Transiciones

> Regla general: **el corte seco debe ser ~60% de las transiciones.**
> El error clásico de los canales de IA es transicionar todo con efectos.

### 9. Corte seco ●
**Qué es.** Sin transición. Un plano termina, el siguiente empieza.

**Cuándo funciona.** Por defecto, dentro de un mismo acto. Es lo que usa
el cine y lo que hace que un video se sienta editado por una persona.
**Cuándo NO.** Entre actos — ahí conviene marcar el cambio.

**Implementación.** Concat demuxer, sin filtro:
```bash
printf "file '%s'\n" plano*.mp4 > lista.txt
ffmpeg -y -f concat -safe 0 -i lista.txt -c copy secuencia.mp4
```

---

### 10. Disolvencia lenta ●
**Qué es.** Fundido cruzado de ~0.8-1.2 s entre dos planos.

**Cuándo funciona.** **Solo entre actos**, nunca dentro de uno. Marca
"cambio de capítulo" y le da al espectador un microdescanso.
**Cuándo NO.** Como transición general: convierte el video en presentación.

**Implementación.**
```bash
OFF=$(python3 -c "print($DUR_A - 1.0)")
ffmpeg -y -i actoA.mp4 -i actoB.mp4 \
 -filter_complex "[0:v][1:v]xfade=transition=fade:duration=1.0:offset=$OFF" \
 -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```

---

### 11. Barrido de luz ●●
**Qué es.** Un destello suave cruza la pantalla y revela el plano
siguiente.

**Cuándo funciona.** Transición **ocasional** (1-2 por video) para marcar
un giro dentro de un acto: el "sí, pero", la revelación. Sutil funciona;
brillante se ve barato.
**Cuándo NO. ** Repetido — pierde todo su efecto y grita "plantilla".

**Implementación.** xfade nativo, o gradiente móvil para más control:
```bash
# Opción simple con xfade
ffmpeg -y -i a.mp4 -i b.mp4 \
 -filter_complex "[0:v][1:v]xfade=transition=wipeleft:duration=0.7:offset=$OFF" \
 -c:v libx264 -crf 20 out.mp4

# Opción con destello: overlay de un gradiente blanco que cruza
# (sweep.png = franja blanca difuminada de 400px, con alfa)
ffmpeg -y -i a.mp4 -loop 1 -i sweep.png \
 -filter_complex "[0:v][1:v]overlay=x='-400+2320*t/0.7':y=0:enable='between(t,$OFF,$OFF+0.7)'" \
 -c:v libx264 -crf 20 out.mp4
```

---

### 12. Match cut por color ●●
**Qué es.** Cortar de una imagen a otra que comparte paleta dominante.
No es un efecto: es una **decisión de montaje**.

**Cuándo funciona.** Siempre que se pueda. Es el tratamiento con mejor
relación costo/beneficio de toda la lista: cuesta casi nada y es lo que
más "mano humana" aparenta, porque implica que alguien ordenó los planos.
**Cuándo NO.** Cuando el guion exige contraste visual fuerte (antes/después).

**Implementación.** Analizar color dominante y ordenar los assets:
```python
from PIL import Image
import colorsys, json

def dominante(path):
    im = Image.open(path).convert('RGB').resize((60, 60))
    r, g, b = map(lambda c: sum(c)/len(c), zip(*im.getdata()))
    h, s, v = colorsys.rgb_to_hsv(r/255, g/255, b/255)
    return h, s, v

# Dentro de cada acto, ordenar los planos por matiz (h) para que los
# cortes encadenen colores cercanos.
planos.sort(key=lambda p: dominante(p)[0])
```

---

## FAMILIA 4 · Énfasis

### 13. Congelado con texto ●●
**Qué es.** El movimiento se detiene en seco y aparece la frase clave.

**Cuándo funciona.** **Máximo 2-3 veces por video**, en la tesis del video
y en el punto de giro. El contraste con el movimiento continuo lo vuelve
imposible de ignorar.
**Cuándo NO.** Más de 3 veces: deja de ser énfasis y se vuelve tic.

**Implementación.**
```bash
# Congela en el segundo T y sostiene 1.5 s con overlay de texto
ffmpeg -y -i plano.mp4 -loop 1 -i frase.png \
 -filter_complex "\
  [0:v]trim=0:$T,setpts=PTS-STARTPTS[pre];\
  [0:v]trim=$T:$((T+1)),setpts=PTS-STARTPTS,\
      select='eq(n,0)',loop=45:1:0,setpts=N/$FPS/TB[frz];\
  [frz][1:v]overlay=0:0[frzt];\
  [pre][frzt]concat=n=2:v=1:a=0" \
 -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```

---

### 14. Desaturación selectiva ●●●
**Qué es.** El fondo pierde color y un elemento queda en color.

**Cuándo funciona.** Para **dirigir la mirada** a un detalle concreto
mientras la voz lo nombra. Muy efectivo en público senior, que agradece
que le señalen dónde mirar.
**Cuándo NO.** Sin máscara precisa se ve sucio. Y es el efecto que más
rápido envejece si se abusa.

**Implementación.** Requiere máscara, igual que el parallax.
```bash
# Con máscara previa (rembg → mask.png, blanco = zona a conservar en color)
ffmpeg -y -loop 1 -t $DUR -i img.jpg -loop 1 -t $DUR -i mask.png \
 -filter_complex "\
  [0:v]scale=1920:1080,split[c][g];\
  [g]hue=s=0[bw];\
  [c][1:v]alphamerge[col];\
  [bw][col]overlay=0:0,setsar=1" \
 -frames:v $FRAMES -c:v libx264 -crf 20 -pix_fmt yuv420p out.mp4
```
**Alternativa barata sin máscara:** desaturar todo el plano salvo un
degradado radial centrado en el punto de interés. Menos preciso pero
cero costo de segmentación.

---

### 15. Zoom brusco (punch-in) ●
**Qué es.** Salto instantáneo de escala sobre la misma imagen. No hay
movimiento: hay un corte a un plano más cerrado.

**Cuándo funciona.** Da **energía** y rompe monotonía. Es el recurso
clásico del video-ensayo en YouTube. Úsalo cuando la voz enfatiza una
palabra, o al iniciar una enumeración.
**Cuándo NO.** En pasajes calmados o emocionales — se siente agresivo.

**Implementación.** Dos crops de la misma foto, unidos con corte seco:
```bash
# Plano abierto
ffmpeg -y -loop 1 -t $T1 -i img.jpg \
 -vf "scale=1920:1080:force_original_aspect_ratio=increase,crop=1920:1080,setsar=1" \
 -frames:v $((T1*FPS)) -c:v libx264 -crf 20 a.mp4
# Plano cerrado (misma foto, 130%)
ffmpeg -y -loop 1 -t $T2 -i img.jpg \
 -vf "scale=2496:1404:force_original_aspect_ratio=increase,crop=1920:1080,setsar=1" \
 -frames:v $((T2*FPS)) -c:v libx264 -crf 20 b.mp4
# Unir con corte seco
printf "file 'a.mp4'\nfile 'b.mp4'\n" > l.txt
ffmpeg -y -f concat -safe 0 -i l.txt -c copy out.mp4
```
**Bonus:** el punch-in **multiplica tu biblioteca** — un mismo asset da
dos planos visualmente distintos.

---

## Reglas de combinación (para el pipeline)

1. Cada acto elige **3-4 tratamientos** y los rota.
2. **Ningún tratamiento se repite más de dos veces seguidas.**
3. El **corte seco domina** (~60%); disolvencia solo entre actos.
4. Al menos **un cambio de composición** por acto (tarjeta, split o
   texto-imagen) para que el ojo no se acostumbre.
5. **Ningún plano dura más de 7 segundos.** En 10 min = ~90 cortes.
6. Énfasis (13, 14, 15) con moderación: máximo 2-3 usos combinados
   por video.
7. Fase uno: implementar 1, 3, 4, 5, 6, 7, 8, 9, 10, 12, 15.
   Fase dos (requieren segmentación): 2 y 14. Opcional: 11.
