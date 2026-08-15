# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es esto

PWA de una sola pantalla para controlar el gasto de un viaje a Asia (ago–sep 2026). Interfaz en español, moneda base MXN. Todo el estado vive en `localStorage` del dispositivo: no hay backend, no hay cuentas, no hay sincronización.

El repo son **5 archivos**: `index.html` (todo el CSS y el JS inline), `sw.js`, `manifest.json`, `icon-192.png`, `icon-512.png`.

Producción: https://mbasilio-code.github.io/asia2026/ (GitHub Pages sobre `main`, raíz del repo).

## Restricciones del proyecto (no negociables)

- **Sin frameworks, sin bundler, sin dependencias npm.** La simplicidad es un requisito explícito, no un accidente. No introducir React, Tailwind, build steps ni `package.json`.
- **No separar el CSS ni el JS de `index.html`.** El `<style>` y el `<script>` se quedan inline.
- **Al tocar `index.html`, subir `CACHE` en [sw.js:3](sw.js#L3)** (`asia2026-v4` → `v5` → …). Sin eso los dispositivos ya instalados siguen sirviendo la versión anterior indefinidamente.
- No afirmar que algo funciona sin haberlo abierto en un navegador sobre HTTP.

## Comandos

No hay build, ni linter, ni suite de pruebas versionada. El único flujo es servir la carpeta por HTTP y probar en el navegador.

En esta máquina **no hay Python ni Node instalados** (`python`/`python3` son los stubs de Microsoft Store, no intérpretes reales). El servidor de desarrollo es un script PowerShell con `System.Net.HttpListener`, fuera del repo:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "c:\Users\Mike Bas\Projects\asia2026-serve.ps1"
```

Acepta `-Root` y `-Port`, así que sirve igual una copia instrumentada desde otra carpeta. Usa los MIME correctos (`sw.js` como `application/javascript`, obligatorio o el registro del service worker falla) y expone `/__probe`, que sirve una copia instrumentada de `index.html` — captura `window.onerror`, `unhandledrejection`, `console.error/warn`, el estado del SW y las cachés, y lo manda por POST a la ruta de reporte, que el servidor escribe a disco. `index.html` en disco nunca se modifica.

**Nunca abrir con `file://`**: sin origen HTTP el service worker no se registra y la app no se comporta como en producción.

Mientras se desarrolla: DevTools → **Application** → **Service Workers** → marcar **Update on reload**. Es imprescindible porque el `fetch` handler es cache-first ([sw.js:28](sw.js#L28), `return hit || red`); sin esa casilla el navegador sirve siempre la copia cacheada. La opción se pierde al cerrar DevTools.

### Verificar sin navegador a mano

No hay motor JS local, pero **Edge headless sí sirve**. Así se verificaron v1.2, v1.3 y v2.0.

⚠️ **Lo que cambió con Edge 151** (lo documentado antes vale para versiones viejas):

- **`--headless=old` ya no existe.** Chromium lo quitó en la 132; Edge 151 lo acepta como bandera pero se comporta como el modo nuevo. Usar **`--headless=new`**.
- **`--dump-dom` devuelve vacío en ambos modos.** Ya no sirve para leer resultados. La salida del arnés se manda por **POST a la ruta de reporte del servidor** (XHR **síncrono**, para que termine antes de que se agote el `--virtual-time-budget`) y se lee del archivo en disco.
- **`--screenshot` sí funciona** con `--headless=new`.
- **El viewport queda ~62px más ancho que `--window-size`** (con `--window-size=560` el `innerWidth` real es 536). Como el `.wrap` tiene `max-width:540px`, capturar a 430 **recorta y parece desbordamiento sin serlo**. Para medir desbordamiento de verdad hay que comparar `scrollWidth` vs `clientWidth` desde JS, nunca mirar la imagen.

```powershell
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" `
  --headless=new --disable-gpu --no-sandbox --user-data-dir="<perfil limpio>" `
  --log-level=3 --virtual-time-budget=8000 --window-size=560,1700 `
  --hide-scrollbars --screenshot="<salida>.png" "http://localhost:8099/"
```

Otros detalles que cuestan tiempo si se redescubren:

- **Usar un `--user-data-dir` limpio por corrida.** Si no, el service worker cacheado de la corrida anterior sirve HTML viejo y las pruebas mienten. En el arnés conviene **no copiar `sw.js`** a la carpeta servida: sin SW no hay caché que mienta.
- **Para capturar una hoja inferior hace falta `--virtual-time-budget`**, o sale a medio deslizarse (`transform` con transición de .22s).
- El patrón de arnés que funciona: inyectar un `<script>` **antes de `</body>`**, es decir después del script de la app. Al ser scripts clásicos del mismo documento, el arnés ve todos los `const`/`let`/`function` de nivel superior (`S`, `calc`, `render`, `sync`, `hojaMov`, …) y puede hacer aserciones y `.click()` reales.
- **Al inyectar con PowerShell, usar `.Replace()` literal, no el operador `-replace`.** El regex de .NET expande `$0` dentro del texto insertado y corrompe el arnés en silencio.
- **`Start-Process -ArgumentList` no entrecomilla.** Cualquier ruta con espacios (todas las de este proyecto) hay que entrecomillarla a mano dentro de la cadena, o el servidor arranca con argumentos partidos y nunca escucha.
- **No redirigir `2>$null` en un `.exe`.** En PowerShell 5.1 eso envuelve cada línea de stderr en un `ErrorRecord` y con `$ErrorActionPreference="Stop"` aborta aunque el proceso haya salido con código 0.
- **Los `.ps1` se leen como ANSI si no tienen BOM.** Un guion largo o un acento en el script rompe el parseo con errores incomprensibles ("string is missing the terminator"). Mantener los `.ps1` en ASCII puro.

## Los tres números de versión (no confundirlos)

| Dónde | Valor actual | Qué es | Cuándo se sube |
|---|---|---|---|
| [sw.js:3](sw.js#L3) `CACHE` | `asia2026-v6` | Nombre de la Cache Storage | **En cada cambio a `index.html`** |
| [index.html:231](index.html#L231) `KEY` | `asia2026-v1` | Clave de `localStorage` | **Casi nunca** — ver abajo |
| [index.html:757](index.html#L757) | `v2.1` | Cadena visible en Ajustes | Manual, cosmético |

⚠️ **`KEY` no es una versión de caché.** Es dónde viven los datos del usuario. Cambiarla equivale a borrarle todos sus movimientos: la app no encuentra estado previo y cae en `nuevoEstado()`. Los cambios de esquema se resuelven con `migrar()`, no tocando `KEY`. La similitud de nombres entre `asia2026-v1` y `asia2026-v6` es una trampa fácil.

## Modelo de datos

Un único objeto `S` serializado a JSON en `localStorage[KEY]`:

```js
S = {
  ajustes: { saldoInicial: 40000,
             tasas: {MXN:1, JPY:.125, KRW:.0135, CNY:2.60, USD:18.50},
             monedaUltima: "MXN",
             paisActual: "México" },   // ← manual, decide accesos rápidos y moneda
  movs: [ /* movimientos */ ]
}
```

Cada movimiento:

```js
{ id, fecha:"YYYY-MM-DD", concepto, categoria, tipo:"gasto"|"ingreso",
  montoPlan,                                  // siempre MXN, lo presupuestado
  estado:"por_pagar"|"a_medias"|"pagado",     // SIEMPRE derivado por sync()
  cerradoAMano,        // true = se cerró sin cubrir el plan (liberó el remanente)
  abonos: [],          // el ÚNICO registro de dinero movido
  nota,
  bolsa: false,        // etiqueta: sale en accesos rápidos. Nada más.
  pais,                // de qué país es la bolsa (null = no sale en accesos rápidos)
  fechaInicio,         // FECHA DE RESERVA de una bolsa (ver abajo); null si no es bolsa
  fechaFin }           // informativo
```

`fecha` es cuándo se gasta; `fechaInicio` es cuándo el dinero **deja de estar
disponible**. `fechaRes(m) = m.fechaInicio || m.fecha` ([index.html:382](index.html#L382))
es la clave de orden del libro. No hay campo `fechaReserva` a propósito: duplicar
un dato que `fechaInicio` ya guarda es la clase de error que causó el bug de v2.0
(`montoReal` contra `abonos`). Tampoco hay regla de quincena en el código — sería
falsa para las bolsas de Souvenirs, que van por tramo de país.

Cada abono:

```js
{ id, fecha, monto,        // monto SIEMPRE en MXN, congelado al capturarlo
  moneda, montoLocal,      // lo que se tecleó (montoLocal null si fue MXN)
  concepto }               // opcional; vacío se muestra como "Gasto suelto"
```

**No existe `montoReal`.** Se retiró en v2.0: si algo está pagado, el monto es `ejercido(m)`. Tener dos registros del mismo hecho (un total y una lista de abonos) es lo que permitía que un movimiento apareciera saldado con abonos incompletos.

### Los tres estados (el corazón de v2.0)

```
por_pagar → sin abonos
a_medias  → tiene abonos pero no cubren el plan
pagado    → los abonos cubren el plan, o se cerró a mano
```

**`estado` se persiste pero NUNCA se asigna a mano.** Lo recalcula `sync(m)` ([index.html:393](index.html#L393)) después de cualquier cambio a los abonos, al plan o a `cerradoAMano`:

```js
m.estado = (m.cerradoAMano || (ej>0 && ej>=m.montoPlan)) ? "pagado"
         : (ej>0 ? "a_medias" : "por_pagar");
```

Se persiste para que el respaldo sea legible; se deriva para que sea **imposible** que un movimiento a medias termine marcado como pagado. Toda mutación de abonos debe terminar en `sync()`.

⚠️ **Regla dura:** un `a_medias` jamás desaparece de la vista por defecto y jamás se muestra palomeado. Hay una prueba de regresión de este caso exacto (ver abajo).

### El bug que originó v2.0 (para no repetirlo)

v1.3 tenía dos estados y una hoja `hojaConfirmar` cuyo campo venía **precargado con el plan** y cuyo botón principal se re-etiquetaba a `"Pagar " + <lo que teclearas>`. Teclear la mitad y tocarlo marcaba el movimiento como `confirmado` con `montoReal` a la mitad: desaparecía del filtro Pendientes, reaparecía palomeado en Todos, y como no tenía abonos el ✓ volvía a abrir la misma hoja, sin camino a un segundo abono.

De ahí las tres decisiones que **no hay que deshacer**:

1. El campo de abono **nace vacío**. Ningún control acepta un monto arbitrario y lo llama saldado.
2. "Pagar el resto" es un botón aparte que **muestra su monto** y precarga un abono; no es un camino de pago distinto.
3. No hay ✓ separado del cuerpo de la fila: **toda la fila abre la misma hoja**.

### Aritmética: una sola, sin ramas

Bolsas y gastos son lo mismo. Seis funciones de doble camino de v1.3 (`real`, `proy`, `desvDe`, `porPartes`, `trasAbonar`, `filaHTML`) quedaron en **cero ramas**:

```js
const ejercido = m => suma de m.abonos.monto        // siempre MXN
const falta    = m => max(0, m.montoPlan - ejercido(m))
const real     = m => signo(m) * ejercido(m)                        // ya se movió
const proy     = m => esPagado(m) ? 0 : signo(m) * falta(m)         // sigue comprometido
const efectivo = m => real(m) + proy(m)                             // −max(plan, ejercido)
const desvDe   = m => esPagado(m) ? signo(m)*(ejercido(m)-m.montoPlan)
                                  : -max(0, ejercido(m)-m.montoPlan)
```

Un movimiento a medias aporta **a los dos lados**: lo abonado a real, lo que falta a proyectado. Por eso **abonar no mueve el saldo proyectado**: cada peso que sale de caja sale también de lo comprometido. Es la propiedad que hace correcto todo el modelo.

`efectivo()` vale `−max(plan, ejercido)` mientras algo siga abierto: la salida
total del movimiento. Lo consume el corte por quincena de `vResumen`, que sí
quiere la salida total del periodo. **Ya no alimenta ningún saldo por fila.**

### Las dos cifras verificables (y por qué no hay saldo corrido)

```js
saldoReal()  = saldoInicial + Σ real(m)          // lo que hay en la bolsa ahora
disponible() = saldoReal() + Σ proy(m)           // menos todo lo comprometido y sin pagar
```

`disponible()` ([index.html:459](index.html#L459)) es **la única definición**: la
usan el hero del Dashboard y la tira de Movimientos, así que no pueden discrepar
(hay prueba, C1).

⚠️ **El disponible es invariante ante cualquier abono.** Al pagar `f`, el saldo
real baja `f` **y** el compromiso baja `f`: `Δ = −f + f = 0`. Sólo se mueve con
sorpresas — baja al sobregirarse, sube al cerrar algo liberando su remanente.

Ésa es la propiedad que hace que **el dinero de una bolsa nunca se vea
disponible**: se descuenta desde que la bolsa existe, sin importar su fecha. Es
más fuerte que "reservado desde el inicio de la quincena".

**Por eso no hay saldo corrido bajo cada fila, y no hay que reponerlo.** Hasta
v2.0 `listaHTML` acumulaba `efectivo(m)` en orden de fecha, lo que producía un
número que: contaba el plan completo de lo ya abonado a medias, daba por
recibidos ingresos pendientes, **no restaba nada posterior a esa fecha** (de ahí
que el dinero de las bolsas de septiembre se viera disponible en agosto), y era
una proyección a una fecha mientras el hero es de hoy. Un "disponible" por fila
sería la misma constante repetida; cualquier variante no constante reintroduce
la trampa. Se sustituyó por una sola cifra etiquetada arriba de la lista
(`tiraDisponible`, [index.html:593](index.html#L593)).

**Regla A para el sobregiro:** pagar de más se reconoce en la desviación de inmediato; el ahorro sólo al cerrar. Si no, el proyectado ya habría absorbido el golpe mientras la desviación seguiría diciendo "vas en plan".

`gastadoViaje` excluye la categoría `Hogar`.

⚠️ **Todo cálculo nuevo que sume dinero debe usar `real()`/`proy()`/`gastoReal()`/`falta()`**, nunca leer `montoPlan` a pelo ni inventar un total.

### Qué es una bolsa (y qué ya no)

Desde v2.0, `bolsa:true` significa **una sola cosa**: el movimiento sale en los **accesos rápidos** de Inicio, para el país que tenga en `pais`. Nada más. Misma aritmética, misma fila, misma hoja, mismos filtros que cualquier otro movimiento.

Una bolsa **sin `pais` no sale en accesos rápidos**, pero sigue siendo un movimiento normal en "Me falta pagar" y en Movimientos. Se le asigna país desde Ajustes → *Administrar bolsas* ([`hojaBolsas`](index.html#L980)) o desde el editor.

**No inventar mecanismos nuevos de parcialidades: la aritmética de abonos ya cubre todos los casos.**

### El país actual manda

`ajustes.paisActual` se elige a mano en Ajustes (México / EUA / Japón / Corea / China) y decide **dos cosas**:

1. Qué bolsas salen en accesos rápidos — `bolsasDelPais()` ([index.html:425](index.html#L425)), que es un `filter` por `pais` y `abierto`, ordenado por fecha.
2. Qué moneda se preselecciona en `hojaAbono` y `hojaRapido` — `monedaPais()`.

⚠️ **Las fechas no resuelven nada.** v1.3 tenía un resolutor de dos niveles (vigente / rezagada / futura) con desempate por `fechaInicio` más reciente; se eliminó entero junto con `rezagadas()` y el empujón de liquidación. `fechaInicio`/`fechaFin` sobreviven como dato informativo. **No reintroducir un resolutor por fechas.**

Si dos bolsas abiertas comparten país salen **las dos**, ordenadas por fecha. Es deliberado: sin desempates invisibles.

### Divisas: la tasa se congela al capturar

`monto` de un abono **siempre queda en MXN**, convertido con la tasa vigente en ese momento; `moneda` y `montoLocal` conservan lo que se tecleó. Cambiar las tasas en Ajustes **no reconvierte** nada ya capturado — es deliberado. Cada abono congela la tasa de su propio día.

La moneda por defecto sale de `paisActual`, **salvo** al editar un abono ya capturado (conserva la suya) y al usar "Pagar el resto" (va en MXN, porque el faltante es un monto en MXN).

## Migración

`migrar()` ([index.html:327](index.html#L327)) corre en **dos puntos**: al cargar y al importar un respaldo. Es idempotente y no borra campos.

Convierte los dos estados de v1.x a los tres nuevos. La idea central: **todo `montoReal` de v1 se vuelve un abono sintético**, porque en v2 el único registro de dinero movido son los abonos.

```
confirmado, ejercido>0 y montoReal===ejercido → pagado + cerradoAMano
      (así se veía una bolsa liquidada a propósito: NO se debe reabrir)
confirmado en cualquier otro caso            → abono sintético por la diferencia,
      luego la regla normal: cubre el plan → pagado; no lo cubre → a_medias
pendiente con abonos                         → a_medias
pendiente sin abonos                         → por_pagar
```

⚠️ **El discriminador `montoReal === ejercido` es indispensable.** Sin él, el mapeo "confirmado + abonos que no cubren → a_medias" reabre las bolsas que el usuario liquidó deliberadamente y mueve sus números.

**Neutralidad aritmética:** cinco de las seis clases de registro dan los tres números idénticos antes y después (hay prueba automatizada que compara contra las fórmulas v1.3 congeladas). La sexta cambia **a propósito**: `confirmado + montoReal < montoPlan + sin abonos` es el registro que el botón buggy dejaba mal cerrado; se reabre a `a_medias` y su remanente vuelve al proyectado. Eso es el objetivo de la versión, no un efecto secundario. `REABIERTOS` cuenta cuántos fueron y la app lo avisa al arrancar.

También detecta bolsas legadas por el sufijo `(bolsa del bloque)`. **No inventa movimientos**: las bolsas de Souvenirs están en `SEED` pero se crean a mano desde Ajustes.

## Arquitectura del render

Sin framework y sin diffing: **`render()` reemplaza `view.innerHTML` completo** y después `enlazar()` vuelve a colgar todos los manejadores.

```
render()  ([index.html:481](index.html#L481))
  ├── marca el tab activo en <nav>, muestra el FAB sólo en Movimientos
  ├── view.innerHTML = { inicio | movs | resumen | ajustes }[tab]()
  └── enlazar()   ← re-vincula TODOS los listeners del contenido
```

⚠️ **Gotcha principal:** cualquier elemento interactivo nuevo dentro de una vista queda muerto tras el siguiente `render()` a menos que se enganche dentro de `enlazar()` ([index.html:1029](index.html#L1029)). Los listeners inline en el HTML generado no sobreviven. Excepción: las hojas inferiores (`abrir()`) enganchan sus botones justo después de renderizarse, porque el sheet vive fuera de `#view` y no lo toca `render()`.

⚠️ **XSS:** todo se arma con template literals. Cualquier cadena del usuario (`concepto`, `nota`, categoría, país, concepto de abono) debe pasar por `esc()` ([index.html:368](index.html#L368)). Ojo: `aviso()` usa `textContent`, así que ahí **no** hay que escapar (sería doble escape).

### Vistas

- `vInicio` ([index.html:638](index.html#L638)) — **tres bloques, en este orden**: hero (saldo real / disponible tras compromisos / desviación), `tarjetaRapido` (accesos rápidos del país), botón grande **Registrar gasto**, y `tarjetaFalta` (**Me falta pagar**, `por_pagar` y `a_medias` juntos, tope de 12). Su titular es `falta(m)`.
- `vMovs` ([index.html:658](index.html#L658)) — `tiraDisponible()`, luego filtros `[Me falta pagar] [Pagado] [Todo]`, el primero por defecto (`let filtro="falta"`).
- `vResumen` ([index.html:667](index.html#L667)) — **una sola tabla** por categoría con planeado / pagado / falta, corte por quincena y totales. Ya no hay tarjeta de Bolsas aparte. El corte por quincena sigue usando `m.fecha` (es un reporte de gasto, no de reserva).
- `vAjustes` ([index.html:719](index.html#L719)) — país actual, saldo inicial, tasas, administrar bolsas, respaldo/restauración/reset.

`listaHTML` ([index.html:571](index.html#L571)) agrupa por **fecha de reserva**
(`fechaRes`) con encabezado sticky; cada encabezado muestra lo que falta pagar
ese día, etiquetado.

`orden()` ([index.html:383](index.html#L383)) ordena por `fechaRes(m)` y, a
igualdad, pone **ingresos antes que gastos**.

⚠️ **Todo número visible lleva etiqueta.** La columna derecha de cada fila dice
`FALTA` / `PAGADO` / `POR COBRAR` bajo el monto. Un número sin encabezado en esa
posición se lee como "saldo", que es exactamente el malentendido que costó v2.1.

### Interacción: un toque, una hoja

**No hay botón ✓.** Un overlay `.hit` cubre la fila completa (`inset:0`) y abre `hojaMov` ([index.html:804](index.html#L804)), que es LA hoja para todo: ver planeado/pagado/falta, lista de abonos, **Abonar**, **Pagar el resto**, **Cerrar ya** (libera el remanente), **Editar**, y **Reabrir** si ya está pagado.

El titular de cada fila es **siempre el remanente** (`falta(m)`) mientras siga abierta, y lo pagado cuando cierra. **Nunca el plan**: mostrar el plan de un movimiento a medias fue el bug de v2.1 (la Renta decía 25,662 cuando faltaban 8,500). El plan sigue visible como contexto en el renglón de avance (`Hogar · pagado $17,162 de $25,662`), y la fila lleva la guarda punteada en ámbar (`.row.medias`). Las filas con abonos tienen un botón `.expand` que despliega las sub-filas; cada sub-fila abre `hojaAbono` para editar o borrar ese abono.

Cerrar un movimiento **sin haber pagado nada** pide segundo toque: devolvería el plan completo como "ahorro".

### Las hojas

- `hojaMov` — la única hoja de un movimiento (arriba).
- `hojaAbono(mov, abono?, volver?, pre?)` ([index.html:708](index.html#L708)) — pide **una sola cosa: el monto**. Moneda, fecha y nota son enlaces discretos `.opts` **debajo del botón**, no campos. El botón **Abonar va antes** de nota y fecha para que el camino corto sea teclear → Abonar sin bajar la vista. `pre` precarga el faltante (lo usa "Pagar el resto").
- `hojaRapido` ([index.html:857](index.html#L857)) — **Registrar gasto**: monto → categoría → Guardar. Nace **pagado**, con fecha de hoy y moneda del país actual. El concepto es opcional (por defecto, la categoría).
- `hojaEditar` ([index.html:929](index.html#L929)) — concepto, fecha, plan, nota, categoría, tipo y el selector **Acceso rápido (bolsa)** que fija `bolsa` y `pais` de una vez.
- `hojaBolsas` ([index.html:980](index.html#L980)) — asignar país a las bolsas existentes y crear las de Souvenirs que falten.

## Pruebas

No hay suite versionada, pero **sí hay un arnés reproducible** (fuera del repo, en el scratchpad de la sesión). Lo que debe seguir pasando:

- **Regresión de la renta (obligatoria).** Abonar 13,425 a un plan de 26,850 **por la UI** debe dejar el movimiento en `a_medias`, visible en el filtro por defecto, **sin** clase `.ok`, mostrando "13,425 de 26,850", y con camino a un **segundo** abono desde la fila. Contra v1.3 esta prueba falla en 7 de 8 aserciones; ese es el punto.
- **Neutralidad**: los tres números calculados con las fórmulas v1.3 congeladas contra las de v2, sobre un estado que cubre las cinco clases neutras.
- **Idempotencia** de `migrar()` y **importación de respaldos v1.1 y v1.3**.
- **País**: cambiarlo cambia los chips y la moneda por defecto; sin bolsas del país el bloque se oculta.
- **Registrar gasto**: nace pagado, con la tasa del día y la moneda tecleada conservada en el abono.
- **Filtros**: "Pagado" nunca contiene un `a_medias`.
- **XSS**: concepto y nota maliciosos se ven como texto.
- **Desbordamiento**: `scrollWidth === clientWidth` (medido en JS, no en la captura).

Consistencia del disponible (v2.1), todas sobre el DOM renderizado:

- **C1**: la tira de Movimientos **es idéntica** al "Disponible tras compromisos" del Dashboard, y ambas a `disponible()`.
- **C2**: `saldoReal() − Σ falta(gastos abiertos) + Σ falta(ingresos abiertos) === disponible()`.
- **C3**: caso reportado — plan 25,662 con 17,162 abonados ⇒ el titular de la fila dice **8,500**, jamás 25,662, y la columna está etiquetada.
- **C4**: abonar 1, 500, 4,250 u 8,500 **no mueve** el disponible.
- **C5**: sobregirarse **sí** lo baja por el exceso; "Cerrar ya" **sí** lo sube por el remanente liberado.
- **C6**: barrido del DOM en Inicio y Movimientos — ningún titular muestra el plan de un `a_medias`.
- **C7**: `fechaRes()` ordena por reserva y es **no-op** mientras `fechaInicio === fecha`.

## Trampas conocidas

- **Fechas:** `aDate()` ([index.html:365](index.html#L365)) parsea el ISO a mano en hora local. Nunca `new Date("2026-08-29")` — se interpreta como UTC y desplaza un día en México.
- **Orden de definición:** `migrar()` usa `ejercido()` y `sync()`, así que la aritmética se define **antes** del bloque que carga `S` y llama a `migrar(S)`. Mover ese bloque hacia arriba rompe con un error de TDZ (`const` en zona muerta).
- **Reset conserva ajustes:** rehace `movs` desde `SEED` pero preserva `S.ajustes` (incluido `paisActual`).
- **Validación de importación mínima:** `ajustes` presente, `movs` array, y por movimiento `id`, `fecha`, `montoPlan` numérico y `abonos` array si viene. Un respaldo v1.x (sin `abonos`, sin `estado` nuevo) sigue funcionando.
- **Exportar tiene tres niveles de fallback:** Web Share con archivo → descarga por `<a download>` → textarea. Pensado para iOS Safari.
- **Cambio de día en caliente:** un listener de `visibilitychange` re-renderiza si la app estuvo en segundo plano y cambió la fecha.
- **`VIAJE_INI` / `VIAJE_FIN`** sólo alimentan el chip de cuenta regresiva; no filtran ni afectan cálculos.
- **`abiertos`** es un `Set` de ids desplegados; vive en memoria, no se persiste, y se limpia al importar o resetear.
- **No reintroducir un saldo corrido por fila** sin leer antes la sección "Las dos cifras verificables". Parece una mejora obvia y es justo el número que confundió al usuario en v2.0.

## Publicar

`main` se despliega directo a GitHub Pages. Tras hacer push, por el patrón stale-while-revalidate del SW el dispositivo abre **una vez más** con la versión anterior y muestra la nueva en el siguiente arranque. Eso es el comportamiento diseñado, no un fallo de despliegue — pero sólo funciona si se subió `CACHE`.

**Antes de cada despliegue: Ajustes → Descargar respaldo.** Los datos viven sólo en el dispositivo.
