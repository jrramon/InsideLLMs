# Propuesta · segunda visita guiada 3D: el flujo residual

Estado: propuesta, sin cambios en el código todavía.
Fecha: 2026-08-03

---

## 1. Qué tenemos hoy

`dentro-de-un-llm-3d.html` (691 líneas, autocontenido, three.js r128 desde cdnjs) es un
**pasillo horizontal**: siete regiones colocadas en el eje X (`x = -32, -12, 8, 30, 52, 74, 96`)
y una cámara que vuela de una a otra en 8 pasos.

```
RA          RB         RC          RD          RE         RF        RG
-32        -12          8          30          52         74        96
embeddings  torre    dentro de   atención    cabezas    caché KV  generación
                     una capa
```

El motor ya resuelto y reutilizable:

| Pieza | Qué hace |
|---|---|
| `box / edges / beam / lambert / basic` | primitivas de escena con el lenguaje visual ya fijado |
| `makeLabel(text, opts)` | texto como sprite de canvas — sirve también para fórmulas |
| `addPulse / addCPulse` | puntos que viajan por una recta o por una curva Bézier |
| `region(x)` | crea un grupo posicionado en el pasillo |
| `STEPS[]` + `goStep()` | narrativa: cada paso lleva `target` y `off` de cámara |
| rig de cámara | esférico, con yaw/pitch/zoom del usuario encima del encuadre |
| `RM` (reduced motion) | `placeStatic()` congela todas las animaciones |
| fallback sin WebGL | pantalla alternativa |

Paleta ya establecida: fondo `#0E0E16`, texto `#EDECE4`, ámbar `#EF9F27` (foco / lo elegido),
verde `#17a673`–`#0F7A55` (k y v, contexto), violeta `#6C63D4`–`#8F86FF` (parámetros aprendidos),
gris `#8A8A82` (h sin contextualizar, y lo que se descarta).

**El hueco importante:** en el archivo actual **el MLP no existe**. La capa es solo atención.

---

## 2. Qué añade la explicación nueva

La explicación que te gusta más aporta ocho ideas, y solo dos de ellas están hoy en la escena:

| Idea | ¿Está en v1? |
|---|---|
| La tubería / pizarra compartida: cada capa **lee y escribe** | insinuada (el cubo sube), no explícita |
| Las capas **no reemplazan, suman**: `h_{ℓ+1} = h_ℓ + Attn + MLP` | a medias (`h′ = h + Δ`, sin MLP) |
| `h` es un vector concreto de ~8192 floats | no |
| El subíndice `t`: la rejilla `h_{ℓ,t}`, información que fluye entre posiciones | no |
| Q/K/V como pregunta / etiqueta / contenido, con softmax como competencia | sí, es el punto fuerte de v1 |
| Multi-cabeza y `W_O` | sí (paso 4 · zoom) |
| Máscara causal | no |
| Superposición de features, y por qué falla el logit lens | no |
| J-lens y las cabezas de *broadcast* | no |

Es decir: hay material nuevo para una pieza entera, no para un parche.

---

## 3. La propuesta

### Decisión de fondo: archivo nuevo, no ampliar el actual

Recomiendo **`flujo-residual-3d.html`**, pieza hermana con el mismo estilo, y no seguir
alargando el pasillo existente. Tres razones:

1. **La metáfora pide otro eje.** El pasillo de v1 es horizontal porque cuenta un *proceso*
   («primero esto, luego lo otro»). El flujo residual es una **tubería vertical** que atraviesa
   el modelo, y el eje horizontal queda libre para la dimensión `t` (las posiciones). Eso da
   gratis la rejilla `h_{ℓ,t}` de la explicación, que en un pasillo horizontal no cabe.
2. **Longitud.** v1 ya son 8 pasos. Añadir 7 u 8 más da una visita de cuarto de hora y una
   cámara recorriendo 200 unidades de mundo.
3. **v1 funciona.** No hay que tocarlo salvo un retoque (ver §6).

Las dos piezas se enlazan entre sí con un enlace discreto en el panel.

### Escena: una sala, no un pasillo

Cuatro tuberías verticales (una por posición) atravesadas por anillos-capa:

```
                         W_U → distribución sobre el vocabulario
      ╔═══════╤═══════╤═══════╤═══════╗
      ║   ▲   │   ▲   │   ▲   │   ▲   ║   h_{L,t}   (verde: contextualizado)
      ║  ┌┴───┴───┴───┴───┴──┐        ║
      ║  │  capa L: Attn+MLP │        ║   ← anillo que envuelve las 4 tuberías
      ║  └┬───┬───┬───┬───┬──┘        ║
      ║   ...                          ║
      ║  ┌┴───┴───┴───┴───┴──┐        ║
      ║  │  capa 1: Attn+MLP │        ║
      ║  └┬───┬───┬───┬───┬──┘        ║
      ║   │   │   │   │   │           ║   h_{0,t}   (gris: solo embedding)
      ╚═══╧═══╧═══╧═══╧═══╧═══════════╝
         el  gato negro  se  sentó
```

Cada anillo-capa tiene, visible y separado:

- un **brazo de lectura** que toma una copia de la tubería (no la vacía: la tubería sigue),
- el **módulo** (atención a un lado, MLP al otro, rejillas violetas como en v1),
- un **brazo de escritura** que vuelve a la tubería y desemboca en un nodo `+` ámbar.

Ese `+` es el corazón de la pieza: es lo que hace que las capas **añadan** en vez de reemplazar.

### Pasos propuestos (9)

| # | Título | Qué se ve |
|---|---|---|
| 0 | **Vista general · la tubería** | La sala entera. Cuatro tuberías grises abajo, verdes arriba. Anillos-capa. Arriba, `W_U`. Frase clave: «las capas no reemplazan, añaden». |
| 1 | **`h_{ℓ,t}` es un vector de números** | Zoom a un tramo de tubería: se abre en una tira de celdas con floats reales (`0.234, -1.567, 0.089, …`), una celda encendida y el rótulo «por sí solo, este número no significa nada». Se ven `dim 0` y `dim 8191`. |
| 2 | **La pizarra compartida** | Un solo anillo, a cámara lenta: brazo de lectura → módulo → brazo de escritura → `+`. Los números de la tira cambian *antes* y *después*. Rótulo: `h_{ℓ+1} = h_ℓ + Attn(h_ℓ) + MLP(h_ℓ)`. **Aquí entra el MLP, que hoy no existe en v1.** |
| 3 | **La oficina** | La narrativa que te gusta, literal: post-its que se van pegando a la tubería según sube — «sujeto de la oración» (Attn capa 1), «animal felino, 4 patas» (MLP capa 1), «probablemente va seguido de un verbo» (capa 2). Al final la tubería lleva encima toda la pila de notas. |
| 4 | **Atención: pregunta, etiqueta, contenido** | La analogía de la reunión. La `q` de «sentó» es una pregunta que sale de su tubería; sobre cada tubería anterior flotan su `k` (etiqueta) y su `v` (contenido). Haces `q·k`, el softmax los hace **competir** (se reescalan en vivo hasta sumar 1) y la suma ponderada regresa al `+` de «sentó». Anotaciones: `/√d_head`, y que q/k/v son **más finos** que la tubería (`d_head = 8192/64 = 128`). |
| 5 | **La máscara causal** | La misma rejilla vista de frente: el triángulo superior derecho apagado y tachado. «sentó» puede mirar a «el», «gato», «negro», «se» y a sí mismo; nunca hacia delante. Plano muy limpio en 3D. |
| 6 | **32 cabezas** | La atención del anillo estalla en 32 discos paralelos; cuatro rotulados (sintáctica, semántica, posicional, patrones). Se concatenan, `W_O` los embudo­na en **un solo Δ**, y ese Δ entra en el `+`. Diferencia con v1: aquí el Δ *aterriza en la tubería*, que es el punto. |
| 7 | **Superposición** | Vuelve la tira de 8192 celdas y, encima, muchísimas más flechas-concepto que dimensiones, cada concepto una dirección, solapadas y densas, con solo unas pocas encendidas a la vez. Cierra con «por eso el logit lens falla en las capas tempranas: las features están revueltas». |
| 8 | **Unembedding** | Lo alto de las tuberías: `W_U` puntúa `h_{L,t}` contra las 200.000 direcciones del vocabulario, softmax, y sale la distribución. Enlace de vuelta a la pieza 1, que sigue desde aquí. |

### Opcional: una tercera pieza sobre el J-lens

El material sobre el **J-space como *broadcast hub*** (ganancia, preservación de etiqueta,
ablación con `recall@25` de 0.86 → 0.67 sin que cambie apenas el output) está un nivel por
encima del resto: ya no es «cómo funciona un transformer» sino «qué hemos descubierto mirando
dentro de uno». Mezclarlo en el paso 8 desequilibra la pieza.

Recomiendo dejarlo fuera de esta segunda visita y, si quieres, hacerlo como tercera pieza —
donde la escena natural es la propia rejilla `h_{ℓ,t}` con las cabezas de broadcast encendidas
y el experimento de ablación como un interruptor que el usuario pulsa.

---

## 4. Continuidad con la pieza 1

- **Misma frase.** La explicación nueva usa «El gato bebe leche»; propongo mantener
  **«el gato negro se sentó»** de v1. No se pierde nada: v1 ya cuenta que la `q` de «sentó»
  encaja con la `k` de «gato», que es exactamente el ejemplo sujeto→verbo.
- **Mismos colores, misma tipografía, mismo panel, mismos controles** (flechas, puntos,
  arrastrar, pinch, `prefers-reduced-motion`, fallback sin WebGL).
- **Helpers copiados tal cual.** Cada archivo sigue siendo autocontenido, como ahora.
- **Fórmulas como sprites**, no LaTeX: igual que v1 dibuja `h′ = h + Δ` con `makeLabel`.

---

## 5. Notas técnicas

- **three.js por CDN.** Hoy se carga desde cdnjs, así que el archivo no funciona sin conexión
  y **no se puede publicar como Artifact** (la CSP bloquea hosts externos). Si quieres poder
  compartirlo por esa vía, hay que inlinear la librería (~600 KB, el archivo pasaría a ~640 KB).
  Decisión independiente de esta propuesta, pero conviene tomarla antes de duplicar el patrón
  en un segundo archivo.
- **Coste estimado:** entre 700 y 900 líneas, comparable a v1.
- **Riesgo de rendimiento:** 4 tuberías × ~8 anillos × (atención + MLP) es bastante geometría.
  Se controla con capas *representativas* (4 anillos detallados + el resto insinuados como
  siluetas), igual que v1 dibuja 6 losas para representar «decenas de capas».

---

## 6. Un retoque pendiente en la pieza 1

Si se publican las dos, hay una incoherencia menor: el paso 3 de v1 dice
«lo único que sale de la capa es `h′`» y presenta la capa como *solo* atención. Con la pieza 2
introduciendo el MLP, conviene añadir en v1 media frase — «…y un MLP que también suma lo suyo,
lo verás en la segunda visita» — para que no se contradigan.

---

## 7. Alternativa más barata, por si el alcance asusta

Si prefieres no abrir un archivo nuevo: se pueden **insertar dos pasos** en v1, entre el actual
paso 3 y el 4, reutilizando la región `RC` (dentro de una capa):

- un paso «el MLP también escribe», que añade el segundo módulo a la escena existente,
- un paso «la pizarra compartida», que reencuadra la cámara sobre el `+` y la torre `RB` a la vez.

Cubre las dos ideas más importantes de la explicación nueva (sumar en vez de reemplazar, y el
MLP), a coste bajo. Deja fuera la rejilla `h_{ℓ,t}`, la máscara causal y la superposición.
