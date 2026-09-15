# MATRANSA — App de tareos de producción

> **Este repositorio es PÚBLICO.** No debe contener nombres de trabajadores, backups,
> itemizados, cotizaciones ni datos de clientes. Ver `.gitignore`.

Archivo único `index.html` — HTML + JS vanilla como módulo ES, **sin build**. Se publica en
GitHub Pages: `https://rvillar-prog.github.io/Matransa`.

Backend: Firestore `matransa-c562c`, auth anónima, `initializeFirestore` con
`persistentLocalCache` + `persistentMultipleTabManager`.
Colecciones: `historial`, `trabajadores`, `proyectosTC`, `proyectos`, `ofs`, `planM3`,
`ausencias`, `notas`.

La versión viva se declara en `VERSION_APP`, en las primeras líneas del archivo a propósito
(para poder leerla sin recorrerlo entero), y se muestra en la pestaña Configuración.

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
`MATRANSA_BACKUPS`.

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
una tarea que cruza el mediodía se dibuja en dos tramos.

### Quién es capacidad — `plantillaDelDia(fecha)`

| | regla | por qué |
|---|---|---|
| Operario productivo | capacidad fija, salvo ausencia registrada | tiene horario |
| Practicante | cuenta **solo el día que tiene tareo**, y ese día entero | no tiene horario fijo; si le dejaron trabajo, vino |
| Coordinador | **nunca** es capacidad | supervisa; contarlo taparía la falta de personal |

Es la única fuente. Kanban (global y por área) y la página Personal beben de ahí.

---

## Trampas conocidas

- **`fechaCreacion` tiene formato mixto** (ISO en los nuevos, dd/mm/aaaa en los viejos).
  Usar siempre `parseFechaCreacion()`.
- **Firestore sin conexión falla en silencio**: la promesa nunca resuelve. Por eso existe el
  helper `escribir()`. No llamar a `updateDoc`/`addDoc` directo sin envolverlos.
- **`renderTabs()` emite `onclick="showPage('...')"` como texto**, a propósito.
- **No escribir `\uXXXX` dentro de HTML crudo.** Es válido en un string de JS pero sale como
  texto literal en el HTML. Ya pasó una vez y se vio en pantalla.
- **"Lijado" existe en dos áreas** (ACABADO y PINTURA DE FIERRO), así que `areaDeOperacion()`
  devuelve `null` para ese nombre a propósito: adivinar sería peor que no derivar.
- En los backups, las OF traen `id`, no `fbId` (la app lo mapea al cargar).
- Los redibujados llegan con cada snapshot de Firestore. Cualquier efecto visual que se dispare
  al dibujar (scroll, foco) necesita una marca para no repetirse — ver `_foco.pintado`.

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

## Estado vivo

Este archivo describe lo que no cambia. Lo que sí cambia — qué se hizo la última semana, qué
decisiones están abiertas, cómo va la adopción — vive en `..\Docs\ESTADO.md`, fuera del repo
porque contiene nombres de trabajadores.
