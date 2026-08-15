# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es esto

PWA de una sola pantalla para controlar el gasto de un viaje a Asia (ago–sep 2026). Interfaz en español, moneda base MXN. Todo el estado vive en `localStorage` del dispositivo: no hay backend, no hay cuentas, no hay sincronización.

El repo son **5 archivos**: `index.html` (todo el CSS y el JS inline), `sw.js`, `manifest.json`, `icon-192.png`, `icon-512.png`.

Producción: https://mbasilio-code.github.io/asia2026/ (GitHub Pages sobre `main`, raíz del repo).

## Restricciones del proyecto (no negociables)

- **Sin frameworks, sin bundler, sin dependencias npm.** La simplicidad es un requisito explícito, no un accidente. No introducir React, Tailwind, build steps ni `package.json`.
- **No separar el CSS ni el JS de `index.html`.** El `<style>` y el `<script>` se quedan inline.
- **Al tocar `index.html`, subir `CACHE` en [sw.js:3](sw.js#L3)** (`asia2026-v3` → `v4` → …). Sin eso los dispositivos ya instalados siguen sirviendo la versión anterior indefinidamente.
- No afirmar que algo funciona sin haberlo abierto en un navegador sobre HTTP.

## Comandos

No hay build, ni linter, ni suite de pruebas versionada. El único flujo es servir la carpeta por HTTP y probar en el navegador.

En esta máquina **no hay Python ni Node instalados** (`python`/`python3` son los stubs de Microsoft Store, no intérpretes reales). El servidor de desarrollo es un script PowerShell con `System.Net.HttpListener`, fuera del repo:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "c:\Users\Mike Bas\Projects\asia2026-serve.ps1"
```

Sirve `http://localhost:8080/` con los MIME correctos (`sw.js` como `application/javascript`, obligatorio o el registro del service worker falla) y expone una ruta `/__probe` que sirve una copia instrumentada de `index.html` — captura `window.onerror`, `unhandledrejection`, `console.error/warn`, el estado del SW y las cachés, y lo hace POST a `/__report`, que el servidor escribe a disco. `index.html` en disco nunca se modifica.

**Nunca abrir con `file://`**: sin origen HTTP el service worker no se registra y la app no se comporta como en producción.

Mientras se desarrolla: DevTools → **Application** → **Service Workers** → marcar **Update on reload**. Es imprescindible porque el `fetch` handler es cache-first ([sw.js:28](sw.js#L28), `return hit || red`); sin esa casilla el navegador sirve siempre la copia cacheada. La opción se pierde al cerrar DevTools.

### Verificar sin navegador a mano

No hay motor JS local, pero **Edge headless sí sirve** y es como se verificaron v1.2 y v1.3:

```powershell
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" `
  --headless=old --disable-gpu --no-sandbox --user-data-dir="<perfil temporal>" `
  --virtual-time-budget=6000 --dump-dom "http://localhost:8080/"
```

Detalles que cuestan tiempo si se redescubren:

- **`--headless=old` funciona; `--headless=new` no** — con el modo nuevo `--dump-dom` y `--screenshot` devuelven vacío.
- **Usar un `--user-data-dir` limpio por corrida.** Si no, el service worker cacheado de la corrida anterior sirve HTML viejo y las pruebas mienten.
- **El viewport ignora `--window-size`** (queda en ~756px de ancho) aunque la captura sí lo respeta: una captura estrecha recorta y parece desbordamiento. Para medir desbordamiento hay que leer `scrollWidth` vs `clientWidth` desde JS, no mirar la imagen.
- **Para capturar una hoja inferior hace falta `--virtual-time-budget`**, o sale a medio deslizarse (`transform` con transición de .22s en [index.html](index.html)).
- El patrón de arnés que funciona: inyectar un `<script>` **antes de `</body>`**, es decir después del script de la app. Al ser scripts clásicos del mismo documento, el arnés ve todos los `const`/`let`/`function` de nivel superior (`S`, `calc`, `render`, `hojaAbono`, …) y puede hacer aserciones y `.click()` reales. Los resultados se escriben en un `<pre id="__out">` y se leen con `--dump-dom`.
- **Al inyectar con PowerShell, usar `.Replace()` literal, no el operador `-replace`.** El regex de .NET expande `$0` dentro del texto insertado y corrompe el arnés en silencio.

## Los tres números de versión (no confundirlos)

| Dónde | Valor actual | Qué es | Cuándo se sube |
|---|---|---|---|
| [sw.js:3](sw.js#L3) `CACHE` | `asia2026-v4` | Nombre de la Cache Storage | **En cada cambio a `index.html`** |
| [index.html:219](index.html#L219) `KEY` | `asia2026-v1` | Clave de `localStorage` | **Casi nunca** — ver abajo |
| [index.html:730](index.html#L730) | `v1.3` | Cadena visible en Ajustes | Manual, cosmético |

⚠️ **`KEY` no es una versión de caché.** Es dónde viven los datos del usuario. Cambiarla equivale a borrarle todos sus movimientos: la app no encuentra estado previo y cae en `nuevoEstado()`. Los cambios de esquema se resuelven con `migrar()`, no tocando `KEY`. La similitud de nombres entre `asia2026-v1` y `asia2026-v4` es una trampa fácil.

## Modelo de datos

Un único objeto `S` serializado a JSON en `localStorage[KEY]`:

```js
S = {
  ajustes: { saldoInicial: 40000,
             tasas: {MXN:1, JPY:.125, KRW:.0135, CNY:2.60, USD:18.50},
             monedaUltima: "MXN" },
  movs: [ /* movimientos */ ]
}
```

Cada movimiento:

```js
{ id, fecha:"YYYY-MM-DD", concepto, categoria, tipo:"gasto"|"ingreso",
  montoPlan,            // siempre MXN, lo presupuestado
  montoReal,            // siempre MXN, lo que realmente pasó (null si pendiente)
  estado:"pendiente"|"confirmado",
  moneda, montoLocal,   // moneda original y monto en esa moneda (null si fue MXN)
  nota,
  abonos: [],           // pagos parciales — ver abajo
  bolsa: false,         // ¿es una bolsa? sólo cambia UX, no aritmética
  fechaInicio, fechaFin, pais   // sólo en bolsas; null en todo lo demás
}
```

Cada abono:

```js
{ id, fecha, monto,        // monto SIEMPRE en MXN, congelado al capturarlo
  moneda, montoLocal,      // lo que se tecleó (montoLocal null si fue MXN)
  concepto }               // opcional; vacío se muestra como "Gasto suelto"
```

`nuevoEstado()` ([index.html:283](index.html#L283)) construye el estado desde `SEED` ([index.html:237](index.html#L237)), tuplas `[fecha, concepto, categoria, tipo, montoPlan, bolsaMeta?]` donde el sexto elemento opcional `{ini, fin, pais?}` marca la bolsa.

### Pagos por partes: el concepto central

**Cualquier movimiento se puede pagar de golpe o por partes.** Un movimiento nace `pendiente` con sólo `montoPlan`. A partir de ahí hay dos caminos:

- **Pagar todo** → `hojaConfirmar` ([index.html:734](index.html#L734)) captura `montoReal` y lo marca `confirmado`. Es el flujo original.
- **Abonar una parte** → `hojaAbono` ([index.html:791](index.html#L791)) empuja un abono a `m.abonos`.

En cuanto un movimiento tiene abonos, cambia de aritmética. `porPartes()` ([index.html:365](index.html#L365)) es el interruptor: `m.bolsa || m.abonos.length > 0`.

```js
real(m)  // dinero que YA se movió: la suma de abonos
proy(m)  // lo que sigue comprometido: max(0, montoPlan − ejercido)
efectivo(m) = real(m) + proy(m)
```

Un movimiento a medio pagar aporta **a los dos lados**: lo abonado al saldo real, lo que falta al proyectado. Renta de $26,850 con un abono de $13,425 → `real −13,425`, `proy −13,425`. Por eso **abonar no mueve el saldo proyectado**: cada peso que sale de caja sale también de lo comprometido. Es la propiedad que hace correcto todo el modelo.

`efectivo()` sigue valiendo `−max(montoPlan, ejercido)` mientras algo esté abierto, que es exactamente la semántica anterior generalizada — por eso el saldo corrido de `listaHTML` no necesitó cambios.

De ahí salen las tres cifras de la portada, en `calc()` ([index.html:434](index.html#L434)): **saldo real** (suma de `real`), **saldo proyectado** (real + suma de `proy`) y **desviación** (`desvDe`, [index.html:387](index.html#L387)).

**Regla A para el sobregiro:** pagar de más se reconoce en la desviación de inmediato; el ahorro sólo al cerrar. Si no, el proyectado ya habría absorbido el golpe mientras la desviación seguiría diciendo "vas en plan".

`gastadoViaje` excluye la categoría `Hogar`.

### Cierre automático

`trasAbonar(m)` ([index.html:395](index.html#L395)) corre después de tocar los abonos de un movimiento:

- **Movimiento normal**: al quedar cubierto (`ejercido >= montoPlan`) se cierra solo, con `montoReal = ejercido` y aviso "«concepto» cubierta". Si un abono se borra o se reduce por debajo del plan, se vuelve a abrir.
- **Bolsa**: nunca. Ahí el sobrante importa y la liquidación es manual, desde `hojaBolsa`.

Esa es la única diferencia de comportamiento entre ambos.

### Qué es una bolsa (y qué ya no)

Tras v1.3, `bolsa:true` significa **sólo dos cosas**:

1. Aparece en **Gasto rápido** en Inicio, resuelta por categoría y fecha.
2. Si tiene `pais`, **preselecciona la moneda** del tramo y muestra los chips visibles.

Además no se cierra sola y usa vocabulario propio ("Reservado/Ejercido/Disponible" en vez de "Planeado/Pagado/Falta"). Todo lo demás es común con cualquier movimiento. **No inventar mecanismos nuevos de parcialidades: la aritmética de abonos ya cubre todos los casos.**

### Dos periodizaciones conviven

`Comida` y `Traslados` van por quincena; `Souvenirs` va por **tramo de país**. Por eso cada bolsa lleva su propio `fechaInicio`/`fechaFin` y **`quincena()` no sirve como resolutor**.

`bolsaVigente(cat, hoy)` ([index.html:417](index.html#L417)) es de dos niveles:

```
vigente  → fechaInicio ≤ hoy ≤ fechaFin, gana la de fechaInicio MÁS RECIENTE
rezagada → ninguna cubre hoy: la abierta más reciente ya vencida
futura   → sólo quedan bolsas que aún no arrancan
```

El desempate por `fechaInicio` más reciente es lo que hace funcionar el **tramo envolvente de Souvenirs EUA (29 ago – 26 sep)**: cede ante Japón/Corea/China mientras estás en cada país, y reaparece solo el 25 de septiembre para el duty free del regreso. Sin ese desempate, una bolsa liquidada haría que el resolutor cayera hacia atrás a cualquier tramo olvidado — bug silencioso.

`rezagadas()` ([index.html:430](index.html#L430)) alimenta el empujón de liquidación en Inicio. El cierre nunca es automático: se liquida a mano, en el aeropuerto.

### Divisas: la tasa se congela al capturar

`monto` de un abono **siempre queda en MXN**, convertido con la tasa vigente en ese momento; `moneda` y `montoLocal` conservan lo que se tecleó. Cambiar las tasas en Ajustes **no reconvierte** nada ya capturado — es deliberado. Cada abono congela la tasa de su propio día, que es más fiel que congelar toda una bolsa a una sola tasa.

Moneda por defecto al abonar:

| Caso | Arranca en |
|---|---|
| Bolsa con `pais` (Souvenirs) | `MON_PAIS[pais]`, chips visibles |
| Bolsa sin país (Comida, Traslados) | `S.ajustes.monedaUltima` |
| Movimiento normal | **MXN siempre** |

Un movimiento normal no hereda `monedaUltima` a propósito: la renta se paga en pesos aunque el último abono haya sido en yenes, y equivocarse ahí multiplica el monto en silencio. `monedaUltima` sólo se actualiza si el abono fue en moneda extranjera o si el destino era una bolsa.

## Arquitectura del render

Sin framework y sin diffing: **`render()` reemplaza `view.innerHTML` completo** y después `enlazar()` vuelve a colgar todos los manejadores.

```
render()  ([index.html:475](index.html#L475))
  ├── marca el tab activo en <nav>, muestra/oculta el FAB
  ├── view.innerHTML = { inicio | movs | resumen | ajustes }[tab]()
  └── enlazar()   ← re-vincula TODOS los listeners del contenido
```

⚠️ **Gotcha principal:** cualquier elemento interactivo nuevo dentro de una vista queda muerto tras el siguiente `render()` a menos que se enganche dentro de `enlazar()` ([index.html:1034](index.html#L1034)). Los listeners inline en el HTML generado no sobreviven. Excepción: las hojas inferiores (`abrir()`) enganchan sus botones justo después de renderizarse, porque el sheet vive fuera de `#view` y no lo toca `render()`.

⚠️ **XSS:** todo se arma con template literals. Cualquier cadena del usuario (`concepto`, `nota`, categoría, concepto de abono) debe pasar por `esc()` ([index.html:354](index.html#L354)). Ojo: `aviso()` usa `textContent`, así que ahí **no** hay que escapar (sería doble escape).

### Vistas

- `vInicio` ([index.html:591](index.html#L591)) — hero, **Gasto rápido** (`tarjetaRapido`, [index.html:562](index.html#L562)), empujón de bolsas por liquidar, y lista de hoy + 3 días.
- `vMovs` ([index.html:621](index.html#L621)) — misma lista con chips de filtro.
- `vResumen` ([index.html:630](index.html#L630)) — **tarjeta de Bolsas** (reservado/ejercido/disponible), gasto por categoría, corte por quincena, totales.
- `vAjustes` ([index.html:699](index.html#L699)) — saldo inicial, tasas, botón de bolsas de Souvenirs, respaldo/restauración/reset.

`listaHTML` ([index.html:545](index.html#L545)) agrupa por día con encabezado sticky. **El saldo corrido se calcula sobre `orden()` completo, no sobre el subconjunto filtrado.**

`orden()` ([index.html:355](index.html#L355)) ordena por fecha y, a igualdad, pone **ingresos antes que gastos**.

⚠️ **Tres agregados necesitan conciencia de abonos**, o un movimiento a medio pagar reporta cero ejercido: el `real` por categoría, el "Sale" por quincena (usa `Math.abs(efectivo(m))`) y `gastadoViaje`. Todo cálculo nuevo que sume dinero debe usar `real()`/`proy()`/`gastoReal()`, nunca `montoReal ?? montoPlan` a pelo.

### Interacción de cada fila

Un overlay `.hit` cubre la fila **excepto** los 48px derechos.

| Toque | Movimiento normal sin abonos | Con abonos | Bolsa |
|---|---|---|---|
| ✓ (derecha) | `hojaConfirmar` — dos caminos | `hojaBolsa` | `hojaBolsa` |
| Cuerpo | `hojaEditar` | `hojaEditar` | `hojaBolsa` |

`hojaBolsa` ([index.html:891](index.html#L891)) es la hoja de pagos y sirve para ambos: cambia etiquetas y botones según `m.bolsa`.

Las filas con abonos muestran el avance como texto (`Hogar · $13,425 de $26,850`) y un botón `.expand` que despliega las sub-filas. Las bolsas además llevan barra `.mini`; los movimientos normales **no** (deliberado: un renglón basta).

### La hoja de abono

`hojaAbono(mov, abono?, volver?)` ([index.html:791](index.html#L791)). En su forma mínima pide **una sola cosa: el monto**. Moneda, fecha y nota son enlaces discretos `.opts` **debajo del botón**, no campos; cada uno despliega su control al tocarlo. Las bolsas conservan la forma rica: atajos de monto (`ATAJOS`) y, si hay país, chips de moneda visibles.

El orden importa: el botón **Abonar va antes** de nota y fecha para que el camino corto sea teclear → Abonar sin bajar la vista.

## Trampas conocidas

- **Fechas:** `aDate()` ([index.html:351](index.html#L351)) parsea el ISO a mano en hora local. Nunca `new Date("2026-08-29")` — se interpreta como UTC y desplaza un día en México.
- **Migración:** `migrar()` ([index.html:315](index.html#L315)) normaliza en memoria y corre en **dos puntos**: al cargar y al importar un respaldo. Es aritméticamente neutral: un respaldo viejo produce exactamente los mismos tres números que antes. Detecta bolsas legadas por el sufijo `(bolsa del bloque)` y convierte una bolsa ya confirmada en un abono sintético equivalente. Es idempotente.
- **Bolsas de Souvenirs:** están en `SEED` pero la migración **no las inventa** en datos existentes. Se crean desde Ajustes (`hojaSouvenirs`, [index.html:991](index.html#L991)), con montos y fechas editables, omitiendo las que ya existan.
- **El editor no crea bolsas.** `hojaEditar` ([index.html:939](index.html#L939)) muestra tramo y país sólo si `m.bolsa` ya es true. Crear bolsas es cosa de Ajustes.
- **Liquidar una bolsa en cero pide segundo toque.** Devolvería la reserva completa como "ahorro".
- **Reset conserva ajustes:** rehace `movs` desde `SEED` pero preserva `S.ajustes`.
- **Validación de importación mínima:** `ajustes` presente, `movs` array, y por movimiento `id`, `fecha`, `montoPlan` numérico y `abonos` array si viene. Un respaldo anterior a v1.2 (sin abonos) sigue funcionando.
- **Exportar tiene tres niveles de fallback:** Web Share con archivo → descarga por `<a download>` → textarea. Pensado para iOS Safari.
- **Cambio de día en caliente:** un listener de `visibilitychange` re-renderiza si la app estuvo en segundo plano y cambió la fecha.
- **`VIAJE_INI` / `VIAJE_FIN`** sólo alimentan el chip de cuenta regresiva; no filtran ni afectan cálculos.
- **`abiertos`** ([index.html:482](index.html#L482)) es un `Set` de ids desplegados; vive en memoria, no se persiste, y se limpia al importar o resetear.

## Publicar

`main` se despliega directo a GitHub Pages. Tras hacer push, por el patrón stale-while-revalidate del SW el dispositivo abre **una vez más** con la versión anterior y muestra la nueva en el siguiente arranque. Eso es el comportamiento diseñado, no un fallo de despliegue — pero sólo funciona si se subió `CACHE`.

**Antes de cada despliegue: Ajustes → Descargar respaldo.** Los datos viven sólo en el dispositivo.
