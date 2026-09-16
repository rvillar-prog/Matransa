# MATRANSA — App de tareos de producción

> **Este repositorio es PÚBLICO.** No debe contener nombres de trabajadores, backups,
> itemizados, cotizaciones ni datos de clientes. Ver `.gitignore`.

Archivo único `index.html` — HTML + JS vanilla como módulo ES, **sin build**. Se publica en
GitHub Pages: `https://rvillar-prog.github.io/Matransa`.

Backend: Firestore `matransa-c562c`, **entrada con Google corporativo** (@matransaperu.com),
`initializeFirestore` con `persistentLocalCache` + `persistentMultipleTabManager`.
Colecciones: `historial`, `trabajadores`, `proyectosTC`, `proyectos`, `ofs`, `planM3`,
`ausencias`, `notas`, `usuarios`.

La versión viva se declara en `VERSION_APP`, en las primeras líneas del archivo a propósito
(para poder leerla sin recorrerlo entero), y se muestra en la pestaña Configuración.

**No hay librería de gráficos y no se va a agregar ninguna.** El único script externo es
`xlsx.full.min.js`. Todo lo visual —Gantt, Kanban, Hoy en planta, los gráficos del Dashboard—
está dibujado a mano con HTML/CSS y SVG en línea.

---

## Estructura de carpetas en la máquina de Ricardo

```
C:\Users\rjvil\MATRANSA\
├── Matransa\      ← este repo (público)
├── Backups\       ← backup*.json descargados de la app — NUNCA al repo
├── Itemizados\    ← PDF y xlsx de cotizaciones — NUNCA al repo
├── Docs\          ← ESTADO.md (el handoff vivo) y notas — NUNCA al repo
└── Tests\         ← arnes.mjs, el arnés de validación
```

El arnés vive fuera del repo porque necesita los backups para correr. Desde `Tests\`:

```
node arnes.mjs
```

Encuentra solo `../Matransa/index.html` y `../Backups/`. Si faltan los backups corre las
pruebas que solo miran el código y avisa. Se puede forzar con `MATRANSA_HTML` y
`MATRANSA_BACKUPS`. Hoy son **354 pruebas**; la sección 7 necesita `backup5.json`.

---

## Modelo del ticket

```
tu       minutos por unidad (plan)
n        cantidad
tt       horas totales = tu × n / 60
tu_real  minutos por unidad medidos
nombres  array de 1 o 2 personas
```

**Un ticket con dos personas dura `tt` para ambas.** No es trabajo que se reparte: es tiempo
que las dos pasan ahí. Si lo hiciera una sola, duraría el doble. Por eso el `tt` completo se
carga a cada persona en todas las cuentas de carga.

### Estados terminales de un ticket
Hay tres, y la distinción es el motivo de existir de buena parte del código:

| estado | campos | significa |
|---|---|---|
| Ejecutado y medido | `completado`, `tu_real` | alimenta el tiempo estándar |
| Ejecutado sin medir | `ejecutadoSinMedir`, `fechaCierreSinMedir` | se hizo, pero no aporta al estándar |
| No ejecutado | `noEjecutado`, `motivoNoEjecucion`, `notaNoEjecucion` | no se hizo, con motivo |

`MOTIVOS_EXIGEN_NOTA = ['prioridad','insumo']` — en esos dos la nota **es** el dato.

### `visibleTareo` es una bandera de VISTA, no de validez (F2-26)
`visibleTareo:false` significa "no se dibuja en el tareo de hoy". Hoy solo lo escribe el cierre
por motivo, pero **la versión vieja de la app lo escribía al completar un ticket**: en el backup
del 14/9 hay 114 tickets completados y medidos con la bandera en `false`, de los 247 medidos que
existen. Cualquier cuenta que hable de **medición** (eficiencia, estándares, desviación) tiene
que ignorar `visibleTareo`; las que hablan de **lo que está pendiente hoy** tienen que mirarla.
Confundirlas fue el bug que tuvo al Dashboard midiendo sobre la mitad de la muestra.

---

## Jornada

```
horaFinJornada:     L-J 17:30 · V 18:00 · S 13:00
jornadaProductiva:  L-J 8.5h  · V 9h    · S 5h      (descuenta 1h de almuerzo)
REFRIGERIO_INI = 12, REFRIGERIO_FIN = 13            (no los sábados)
capacidadSemana:    8.5×4 + 9 + 5 = 48h
```

Nunca escribir `8`, `17` ni `46.75` a mano. Todo sale de esas funciones — hubo tres parches
seguidos arreglando justamente eso.

`tramosTarea(cursor, dur, hayRef)` reparte la duración de una tarea saltando el refrigerio:
una tarea que cruza el mediodía se dibuja en dos tramos. **Devuelve `{tramos:[{inicio,fin}], cursor}`
— objetos, no pares.**

### Quién es capacidad — `plantillaDelDia(fecha)`

| | regla | por qué |
|---|---|---|
| Operario productivo | capacidad fija, salvo ausencia registrada | tiene horario |
| Practicante | cuenta **solo el día que tiene tareo**, y ese día entero | no tiene horario fijo; si le dejaron trabajo, vino |
| Coordinador | **nunca** es capacidad | supervisa; contarlo taparía la falta de personal |

`dashCapacidades(rango)` aplica esas mismas tres reglas por área y por rango. **Pendiente:**
la excepción de Patrick Salazar (Pintura de Fierro cuenta como un operario más, decisión de
Ricardo del 15/9) está documentada pero **no implementada en el código**.

Punto ciego conocido y buscado: la capacidad se cuenta por el **área base del trabajador** y la
carga por el **área de ejecución**. Un área por encima del 100% puede estarse sosteniendo con
gente prestada de otra. Eso es señal, no error de cuenta.

### Unidades y catálogo
Las 59 operaciones tienen unidad definida (34 pieza, 10 mueble × mano, 8 mueble, y 7 repartidas
entre metro lineal, punto, unión, perforación, lote y unidad). `ETIQUETAS_UNIDAD` traduce cada
una al rótulo del campo de cantidad, que cambia solo al elegir la operación. **Nunca escribir
"N° unidades" a mano**: mientras el campo decía eso para las 59, todos contaron piezas.

### Cierres
`duplicado` es el único motivo que no cuenta contra el plan, y sólo se ofrece si `ticketGemelo()`
encuentra el otro ticket (misma fecha, tarea, producto y una persona en común). Si no hay gemelo,
la salida honesta es `cancelado`, que sí cuenta y exige nota. El motivo `error` está retirado:
vive en `MOTIVOS_HISTORICOS` para que los 32 tickets viejos se sigan leyendo.

Es la única fuente. Kanban (global y por área) y la página Personal beben de ahí.

---

## Dashboard — los dos universos (F2-26)

El Dashboard es un centro de control con jerarquía de diagnóstico (¿estamos bien? → ¿dónde está
el problema? → ¿qué operación lo provoca? → ¿por qué? → ¿quién necesita atención?). Todo el
motor vive en un bloque contiguo, entre el comentario `// ===== DASHBOARD — CENTRO DE CONTROL`
y `window.exportarExcelDash`. El arnés lo carga entero; **no extraer funciones sueltas de ahí.**

```
dashPlan(r)      cumplimiento, carga, pendientes
                 tickets del rango que cuentan contra el plan; solo se cae lo que nunca
                 debió existir (motivo 'duplicado' y el 'error' retirado)

dashMedidos(r)   eficiencia, desviación, Pareto, estándares, scatter
                 tu_real != null, SIN mirar visibleTareo
```

**Los dos universos no se mezclan en una misma resta.** Cuando se comparan plan y real, el plan
es el de los tickets medidos, nunca el total planificado. De mezclarlos salía el caso que no
cuadraba: `plan 70.9 / real 57.3 / eficiencia 108%` — la columna mostrada no era la que usaba la
fórmula.

```
EFICIENCIA = horas plan DE LO MEDIDO / horas reales DE LO MEDIDO × 100
```

Es la misma fórmula que ya usaba la app (`planCompletados/real`). **No cambiarla.**

Umbrales y criterios en `DASH_UMBRALES`, en un solo sitio, y la pantalla dice cuál está usando.
No son verdades universales y así se presentan.

**Reglas anti-ruido** (si todo sale rojo, nada es importante):
- un área alerta con ≥ `minTicketsArea` tickets y capacidad registrada
- una operación alerta con ≥2 ejecuciones y ≥ `horasPerdidasMin` horas perdidas
- un estándar alerta con ≥ `minEjecuciones` mediciones **y** ≥ `consistencia` del mismo signo
- el estado por área se juzga **contra la media de la planta**, no contra un umbral fijo: con
  umbrales absolutos las ocho áreas salían amarillas y la pantalla no decía nada
- máximo 6 alertas, agrupadas por familia

`calcularAtencion()` es la única fuente de las alertas de OF/entregas. El Dashboard la llama y
la reutiliza; no la duplica.

**El estándar nunca se cambia solo.** Se detecta, se marca y decide una persona.

### Excel
`CABECERA_EXCEL` y `filaExcel(t)` son la única definición de las columnas. Las usan
`exportarExcel()` (Historial, filtros del Historial) y `exportarExcelDash()` (Dashboard, filtros
del Dashboard). Una columna nueva llega a los dos y no se pueden desincronizar. El arnés cubre
los índices de columna con 6 pruebas.

---

## Sesión y permisos (F2-28)

Se entra con la cuenta de Google del dominio. La identidad la pone Google; la **autorización**
la pone Firestore:

```
usuarios/{correo} = { nombre, rol, admin, pestanas[], activo }
```

**Autenticado no es autorizado.** Cualquiera con un Gmail puede autenticarse; sin ficha no ve
un dato. La ficha se edita desde Configuración → Usuarios y permisos, no desde la consola de
Firebase: los permisos cambian con la realidad y esa es una decisión de jefatura, no una tarea
de programador. Se aplican en vivo — la ficha propia se escucha con `onSnapshot`.

- Una ficha **sin** `pestanas` vale por todas. Es el arranque: la primera cuenta se crea a mano
  en la consola con tres campos.
- `PLANTILLAS_ACCESO` son puntos de partida, no jaulas. Lo que manda es el arreglo `pestanas`.
- **Al último administrador no se le puede quitar el permiso**, ni a uno mismo el acceso: si no,
  la única salida sería la consola de Firebase.
- Los nombres de las personas ya **no** están en el código — viven en la ficha. El repo es
  público; que estuvieran ahí era una fuga.
- `requiereAdmin()` ya no es decorativo. Antes leía `perfilActivo` desde `localStorage` y
  bastaba escribir una línea en la consola del navegador para ser administrador.

### Orden de encendido, si hay que repetirlo
1. Proveedor Google habilitado + `rvillar-prog.github.io` en dominios autorizados
2. Ficha propia a mano en Firestore
3. Código con login publicado
4. Reglas **puente** (incluyen `usuarios`, el resto sigue en `request.auth != null`)
5. Las cuatro cuentas entran
6. Reglas **estrictas** + apagar el proveedor Anónimo

Saltarse el 4 deja a todo el mundo fuera: las reglas anteriores negaban `usuarios` por el
`match /{document=**} { allow read, write: if false; }` final, y sin poder leer la ficha la app
cierra la sesión. Los dos archivos viven en `..\Docs\firestore_*.rules`.

---

## Trampas conocidas

- **`fechaCreacion` tiene formato mixto** (ISO en los nuevos, dd/mm/aaaa en los viejos).
  Usar siempre `parseFechaCreacion()`.
- **Firestore sin conexión falla en silencio**: la promesa nunca resuelve. Por eso existe el
  helper `escribir()`. No llamar a `updateDoc`/`addDoc` directo sin envolverlos.
- El menú son **grupos desplegables** (`.tab-menu`) y `showPage` encuentra la pestaña activa
  por **`data-page`**.
- **No escribir `\uXXXX` dentro de HTML crudo.** Es válido en un string de JS pero sale como
  texto literal en el HTML. Ya pasó una vez y se vio en pantalla.
- **"Lijado" existe en dos áreas** (ACABADO y PINTURA DE FIERRO), así que `areaDeOperacion()`
  devuelve `null` para ese nombre a propósito: adivinar sería peor que no derivar.
- En los backups, las OF traen `id`, no `fbId` (la app lo mapea al cargar).
- Los redibujados llegan con cada snapshot de Firestore. Cualquier efecto visual que se dispare
  al dibujar (scroll, foco) necesita una marca para no repetirse — ver `_foco.pintado`.
- **Los filtros del Dashboard viven FUERA de `#dash-content`**, porque ese div se reescribe
  entero con cada snapshot y borraría lo que el usuario acaba de elegir.
- **Plurales en español**: usar `plural(n, sing, plu)`. Concatenar `'operación'+'es'` da
  "operaciónes".
- **`new Date('AAAA-MM-DD')` es medianoche UTC, no local.** En Lima (UTC−5) eso cae el día
  anterior a las 19:00. Comparado contra una fecha hecha con `new Date(a,m,d)` —que sí es
  local— el día no cuadra. Fue el bug F2-27: las ausencias no descontaban capacidad, y sólo se
  veía fuera de UTC. Para cualquier fecha guardada como `AAAA-MM-DD` usar **`fechaISOLocal()`**;
  para las `d/m/aaaa`, `fechaStrToDate()`. Nunca `new Date(texto)` a secas.
- **Un error que se presenta como "no pasó nada" es peor que el error.** Pasó en el login: al
  fallar la lectura de la ficha se mostraba el aviso y acto seguido se cerraba la sesión, lo que
  redibujaba la pantalla de entrada y borraba el mensaje. El aviso vive ahora en `_avisoLogin`,
  fuera del DOM, y sobrevive al redibujado.
- **El arnés hay que correrlo en la zona horaria de la planta.** El contenedor donde se
  desarrolla está en UTC y ahí varios bugs de fecha son invisibles. Desde `Tests\`:
  `node arnes.mjs` en la máquina de Ricardo (Lima) es la prueba que vale. Una prueba de fecha
  bien escrita pasa en cualquier zona; si sólo pasa en una, la prueba está mal o el código lo
  está.

---

## Cómo se trabaja aquí

Por cada cambio: **contexto → riesgo → implementación → validación → regresión → reporte**.

- Protocolo completo solo para riesgo medio o alto. Los cambios chicos van directo.
- **"No programar" es una salida válida** y se usa.
- **Un cambio = un tema.** Nada de refactors amplios sin pedido explícito.
- Antes de dar nada por bueno: correr el arnés (`node arnes.mjs` desde `Tests\`),
  `node --check` sobre el módulo, y verificar que no quedaron `\uXXXX` fuera de `<script>`.
- Subir `VERSION_APP` en cada entrega.
- **El botón de publicar es de Ricardo.** Los cambios se dejan en el archivo; él los revisa en
  GitHub Desktop y hace commit y push.

### Estado del diseño
Etapa 1 hecha (F2-22): tipografía (Archivo / IBM Plex Sans / IBM Plex Mono), escala de texto y
espacio en `:root`, menú desplegable. Etapa 2 hecha (F2-23): tokens de color, paleta de áreas
unificada, familias semánticas. F2-26: la cabecera deja de ser un bloque negro y pasa a
superficie clara con borde; entra `--mt-verde` como acento de marca.

**`--mt-verde` es provisional** (`#1b7a3e`): la app no tenía ningún verde de marca y el hex real
del logo todavía no está. Cambiarlo es una línea, en `:root`. **No es `--ok`**: `--ok` significa
"esto está bien" y `--mt-verde` significa "MATRANSA".

Quedan ~179 hexes sueltos en el JS (eran 484) y los estilos en línea todavía no pasaron a clases.

### Disciplina de color (C2)
Lo normal no se pinta: **el color marca la excepción**. Tokens `--tk-*`, una sola función
`estadoTicket(t)`. Al resaltar algo, usar marco y no relleno: el fondo de cada tarjeta ya
significa algo (estado del ticket, color del proyecto) y pintarlo encima borra esa información
justo en el elemento que se quiere leer.

---

## Cómo hablarle a Ricardo

Jefe de Operaciones, no programador. Quiere evaluaciones **directas, críticas y honestas**, no
optimismo. Si algo está mal, se dice. Español. Y si un número no se puede sostener con datos,
no se da.

El objetivo de fondo: **que el tareo y el Gantt se generen solos al cargar el itemizado.**
Generarlo a mano cuesta hoy 2 personas × 2 horas diarias.

---

## Pendiente que no es de código

**Cerrar la migración de seguridad.** Hecho: reglas por colección, borrado de `historial`
prohibido, validación de forma, login con Google y permisos por ficha. **Falta:**

1. Publicar `Docs\firestore_ESTRICTAS_manana.rules` — exige dominio, correo verificado y ficha
   vigente. Hasta que eso pase, el servidor sigue aceptando cualquier sesión.
2. **Apagar el proveedor Anónimo** en Authentication. Mientras siga encendido, la llave vieja
   funciona aunque la app ya no la use.
3. Verificar después que los cuatro siguen entrando.

**App Check** queda para después: bloquea el uso de la config desde un script externo, pero no
impide que alguien con la URL abra la app. Eso lo cierra el login, que ya está.

**MFA obligatorio desde el 20/10/2026** en la cuenta de Google de Ricardo, o pierde el acceso a
su propia consola.

---

## Estado vivo

Este archivo describe lo que no cambia. Lo que sí cambia — qué se hizo la última semana, qué
decisiones están abiertas, cómo va la adopción — vive en `..\Docs\ESTADO.md`, fuera del repo
porque contiene nombres de trabajadores.
