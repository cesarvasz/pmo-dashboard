# PMO Dashboard — Referencia Técnica

## Estructura general

El proyecto es un **único archivo HTML** (`pmo-dashboard.html`) que contiene todo:
- CSS con variables de tema (dark/light)
- HTML del layout (sidebar + content)
- JavaScript con toda la lógica

No hay framework, no hay build step. La única dependencia externa es **Chart.js** (cargado desde CDN).

```
pmo-dashboard.html
├── <style>          → Estilos + variables de tema CSS
├── <body>           → Layout HTML (sidebar, header, main)
└── <script>         → Toda la lógica JS (tema, navegación, páginas)
```

---

## Arquitectura del código JS

El script está dividido en secciones claramente marcadas con comentarios `════`:

```
TEMA
CONFIGURACIÓN GLOBAL
MULTI-SELECT UTILITIES
REGISTRO DE PÁGINAS (PAGES)
NAVEGACIÓN
COUNTDOWN
UTILIDADES COMPARTIDAS
PÁGINA: INICIATIVAS
PÁGINA: REQ
PÁGINA: CONTROL TOWER (OVERVIEW)
PLACEHOLDER
INIT
```

### Patrón de páginas

Cada página sigue el mismo patrón:

```
load(container)          ← punto de entrada, llama a la API
  └─ process(items)      ← transforma datos crudos en objetos limpios
       └─ render(...)    ← escribe el HTML en el DOM
            ├─ renderCards()
            ├─ renderFilters()
            ├─ renderPMHealth()
            └─ renderTable() / renderSections()
```

Para agregar una nueva página basta con:
1. Agregar un objeto al array `PAGES` con `{ id, label, icon, subtitle, load }`
2. Implementar la función `load(container)`

---

## Configuración global

| Constante | Valor | Descripción |
|---|---|---|
| `API_KEY` | JWT token | Token de autenticación Monday.com |
| `REFRESH_SECS` | `300` | Segundos entre auto-recargas (5 min) |

---

## Tema (dark / light)

| Función | Descripción |
|---|---|
| `applyTheme(theme)` | Aplica `data-theme` al `<html>`, actualiza ícono, guarda en `localStorage` |
| `toggleTheme()` | Alterna entre `"dark"` y `"light"` |
| `initTheme()` | Al cargar: lee `localStorage` o detecta preferencia del sistema |

Las variables CSS (`--bg-base`, `--accent`, `--pill-atrasado-bg`, etc.) están definidas en `:root` para dark y en `[data-theme="light"]` para light. Cambiar el tema solo requiere cambiar el atributo en `<html>`.

---

## Navegación

### Array `PAGES`
Registro central de todas las páginas. Cada objeto tiene:
```js
{ id: "overview", label: "Control Tower", icon: "◎", subtitle: "...", load: loadOverview }
```
Si `load` es `null`, la página muestra un placeholder y aparece deshabilitada en el sidebar.

### Variables de filtro inicial (navegación desde Control Tower)

| Variable | Descripción |
|---|---|
| `iniInitialPM` | Si tiene valor, `iniRender` lo aplica como filtro de PM al cargar y lo resetea a `null` |
| `reqInitialPM` | Ídem para REQ |

Permiten navegar desde Control Tower a una página ya filtrada por un PM específico sin necesidad de que la página esté cargada de antemano.

### Funciones de navegación

| Función | Descripción |
|---|---|
| `buildNav()` | Genera el HTML del sidebar a partir de `PAGES` |
| `navigate(id)` | Activa una página: actualiza título, marca nav item, llama a `page.load()` |
| `refreshPage()` | Re-llama a `activePage.load()` |
| `toggleSidebar()` | Colapsa/expande el sidebar (clase CSS `collapsed`) |

---

## Countdown de auto-recarga

| Función | Descripción |
|---|---|
| `startCountdown(onExpire)` | Inicia contador regresivo de `REFRESH_SECS`. Llama `onExpire()` al llegar a 0 |
| `stopCountdown()` | Detiene el intervalo y limpia el texto del contador |

Cada página arranca el countdown con `startCountdown(() => loadXxx(container))` al terminar de cargar datos.

---

## Utilidades compartidas

### Fechas

| Función | Firma | Descripción |
|---|---|---|
| `businessDays` | `(start, end) → number` | Cuenta días hábiles entre dos fechas (excluye sáb/dom). El día `start` no se cuenta, el día `end` sí |
| `addBusinessDays` | `(date, n) → Date` | Suma `n` días hábiles a una fecha |
| `fmtDate` | `(date) → string` | Formatea fecha como `"29 may. 2026"` (locale `es-GT`) |
| `isToday` | `(date) → boolean` | Retorna `true` si la fecha es hoy (ignora hora) |

### API y DOM

| Función | Descripción |
|---|---|
| `mondayFetch(query)` | Envía una query GraphQL a `api.monday.com/v2`. Lanza error si HTTP falla o si la respuesta trae `errors` |
| `getCSSVar(name)` | Lee el valor de una variable CSS del documento |
| `showLoader(container, msg)` | Reemplaza el contenido del container con un spinner |
| `showError(container, msg)` | Reemplaza el contenido con un cuadro de error rojo |

### Multi-select dropdown

| Función | Descripción |
|---|---|
| `msToggle(id)` | Abre/cierra el panel del dropdown con `id`. Cierra cualquier otro que esté abierto |
| `renderMS(id, labelText, options, selected, toggleFn, allFn)` | Genera el HTML completo de un dropdown multi-select. `options` es `[{value, label, count}]`. `toggleFn` y `allFn` son strings con el nombre de la función global a llamar |

---

## Página: Iniciativas

Fuente de datos: **Monday.com board** `18412996100` + **Google Apps Script** (calendario de meetings).

### Constantes de configuración

| Constante | Valor | Descripción |
|---|---|---|
| `INI_BOARD_ID` | `"18412996100"` | ID del board en Monday.com |
| `CALENDAR_WEBAPP_URL` | URL de GAS | Endpoint del calendario de meetings |
| `INI_LIMITS` | `{ "New": 5, "Meeting 1": 10 }` | Días hábiles límite por status |
| `INI_APPROVED` | `Set(["PM Aprobado", "REQ Aprobado"])` | Statuses que se muestran como Aprobadas |
| `INI_SKIP` | `Set(["Meeting 2"])` | Statuses que se excluyen de la vista |
| `INI_SEC_ORDER` | Array de statuses | Orden en que aparecen las secciones |
| `INI_SEC_LABEL` | Objeto | Etiqueta visible de cada sección |
| `INI_ACTIVE_STS` | `Set(["New", "Meeting 1"])` | Statuses considerados "en proceso" |

### Estado de filtros (variables globales)

| Variable | Descripción |
|---|---|
| `iniData` | Array de todos los items procesados |
| `iniPMs` | Array de PMs seleccionados (vacío = todos) |
| `iniStatuses` | Array de statuses seleccionados |
| `iniEstado` | Filtro de estado activo (`"ATRASADO"`, `"PARA HOY"`, etc.) |
| `iniSinAgendar` | `boolean` — mostrar solo items sin meeting agendada |
| `calMeetings` | `Map<codigo, {M1:[], M2:[]}>` — meetings del calendario |

### Lógica de estado por item

```
status ∈ INI_APPROVED    → estado = "APROBADA"
status = "Sin Valor Def" → estado = "EN_ESPERA"
status ∈ INI_SKIP        → estado = "SKIP" (oculto)
status ∈ INI_LIMITS:
  creacion existe:
    días > límite        → "ATRASADO"
    días = límite        → "PARA HOY"
    días < límite        → "EN TIEMPO"
  creacion no existe     → "Sin fecha"
```

**"Para Hoy"** tiene una condición especial: aplica si el deadline calculado es hoy **O** si hay una meeting agendada en el calendario para hoy.

### Funciones de datos

| Función | Descripción |
|---|---|
| `fetchCalendar()` | GET al Google Apps Script. Retorna array de `{codigo, meeting, inicio, fin}` |
| `buildCalMap(meetings)` | Convierte el array en `Map<codigo, {M1:[...], M2:[...]}>` ordenado por fecha |
| `nextOrLatest(arr)` | De un array de meetings, retorna la próxima futura o la más reciente pasada |
| `calCell(arr)` | Genera el HTML de una celda de meeting (ícono + fecha + hora, con color por estado) |
| `loadIniciativas(container)` | Llama en paralelo a Monday y al calendario; al terminar llama a `iniRender` |
| `iniProcess(items)` | Transforma items raw de Monday en objetos `{id, name, grupo, pm, status, estado, dias, limite, deadline, creacion, ...}` |
| `iniGetFiltered(pms, sts, estado, sinAgendar)` | Filtra `iniData` por los parámetros dados |
| `iniFiltered()` | Shorthand: llama `iniGetFiltered` con los filtros activos actuales |
| `iniIsParaHoy(r)` | Retorna `true` si el item aplica como "Para Hoy" |
| `iniRenderCards(filtered)` | Dibuja las tarjetas de resumen (En Proceso, Atrasadas, Para Hoy, En Tiempo, Aprobadas, Sin Agendar) |
| `iniRenderFilters()` | Dibuja los dropdowns de filtro (PM, Status) |
| `calcPMHealth(pmName)` | Calcula `{score, status, active, total, atrasados}` para un PM. `score` = promedio de días de exceso en items atrasados |
| `iniRenderPMHealth()` | Dibuja las tarjetas de salud por PM (On Track / At Risk / Off Track) |
| `iniSelectPM(pm)` | Toggle: si ese PM ya está solo seleccionado, lo deselecciona; si no, lo selecciona exclusivamente |
| `iniRenderSections(filtered)` | Dibuja cada sección (tabla por status) con sus headers, filas y badges |
| `iniRefresh()` | Re-renderiza filtros, cards, PM health y secciones con los filtros actuales |
| `iniRender(container, items)` | Renderizado inicial: procesa datos, aplica `iniInitialPM` si está definido, crea estructura DOM y llama a `iniRefresh` |

### Funciones de toggle de filtros

| Función | Descripción |
|---|---|
| `iniTogglePM(v, checked)` | Agrega/quita un PM del filtro |
| `iniToggleAllPM(checked)` | Limpia selección de PMs |
| `iniToggleSt(v, checked)` | Agrega/quita un status del filtro |
| `iniToggleAllSt(checked)` | Limpia selección de statuses |
| `iniToggleEstado(estado)` | Toggle del filtro de estado (mutuamente exclusivo con sinAgendar) |
| `iniToggleSinAgendar()` | Toggle del filtro "Sin Agendar" (mutuamente exclusivo con estado) |
| `iniReset()` | Limpia todos los filtros |

---

## Página: REQ (Requerimientos VALOR Lite)

Fuente de datos: **Monday.com board** `18413204916`.

### Mapeo de columnas (`REQ_COLS`)

| Clave | ID de columna | Descripción |
|---|---|---|
| `id` | `pulse_id_mm3fs4r8` | ID del REQ |
| `pm` | `multiple_person_mm3gq5vr` | Project Manager |
| `resp` | `multiple_person_mkvdkw7j` | Responsable |
| `status` | `status` | Status del item |
| `cpmStart` | `timeline9` | Fecha inicio CPM (formato `"YYYY-MM-DD - YYYY-MM-DD"`) |
| `estDev` | `date_mm3gqxw0` | Fecha estimada de desarrollo |
| `costRH` | `labor_budget_spent` | Costo de Recursos Humanos |
| `costSoft` | `numeric_mm3gbavc` | Costo de Software |
| `benefit` | `numeric_mkvcd6nf` | Beneficio esperado |
| `creation` | `pulse_log_mkvyjb6s` | Fecha de creación |
| `tld` | `dropdown_mm3gpacy` | TLD asignado (`JA`, `LM`, `S/dev`) |
| `vDone` | `date_mm3ggd8v` | Fecha fin Valuación |
| `aDone` | `date_mm3gfn1r` | Fecha fin Aprobación |
| `lDone` | `date_mm3g8mqz` | Fecha fin Launch/Desarrollo |
| `oDone` | `date_mm3g5j38` | Fecha fin Operación |

### Pipeline de fases

```
Valuación → Aprobación → Desarrollo → Operación → Cierre ROI → Cerrados
                                                              → En Espera
```

Colores por fase definidos en `REQ_GROUP_COLOR`.

### Lógica de deadline por fase

| Fase | Fecha inicio | Cálculo |
|---|---|---|
| Valuación | `cpmStart` | `+1` día hábil |
| Aprobación | `vDone` | `+2` días hábiles |
| Desarrollo | `aDone` | Si hay `estDev` → usa `estDev` directo. Si TLD=`JA` → `+7`. Si TLD=`LM` → `+32`. Si TLD=`S/dev` → mismo día |
| Operación | `lDone` | `+3` días hábiles |
| Cierre ROI | `oDone` | `+20` días hábiles |

### Estado calculado

```
grupo = "Cerrados"   → "CERRADO"
grupo = "En Espera"  → "EN_ESPERA"
sin deadline         → "EN PROCESO"
deadline < hoy       → "ATRASADO"
deadline = hoy       → "PARA HOY"
deadline > hoy       → "EN TIEMPO"
```

### Estado de filtros (variables globales)

| Variable | Descripción |
|---|---|
| `reqData` | Array de todos los items procesados |
| `reqPMs` | PMs seleccionados |
| `reqStatuses` | Statuses seleccionados |
| `reqGroups` | Fases seleccionadas |
| `reqFase` | Filtro de fase única (usado internamente) |
| `reqEstado` | Filtro de estado activo |

### Funciones de datos

| Función | Descripción |
|---|---|
| `loadReq(container)` | Fetch a Monday.com; llama a `reqRender` al terminar |
| `reqProcess(items)` | Transforma items raw en objetos con `{id, name, grupo, pm, resp, estado, deadline, inicio, dias, limite, costRH, costSft, benefit, valueNet, tld}` |
| `reqGetFiltered(pms, sts, grps, fase, estado)` | Filtra `reqData`. Excluye siempre los `"CERRADO"` |
| `reqFiltered()` | Shorthand con filtros actuales |
| `fmtMoney(n)` | Formatea número como `"$1,234"` |
| `reqRenderCards(filtered)` | Dibuja tarjetas: En Proceso, Atrasados, Para Hoy, En Tiempo, En Espera, Costo Total, Benefit Total, Beneficio Neto |
| `reqRenderFilters()` | Dibuja dropdowns PM, Status, Fase |
| `reqRenderPMHealth()` | Dibuja tarjetas de salud por PM |
| `reqSelectPM(pm)` | Toggle de selección exclusiva de PM |
| `reqRenderTable(filtered)` | Genera el HTML completo de la tabla de REQs |
| `reqRefresh()` | Re-renderiza todos los componentes REQ |
| `reqRender(container, items)` | Renderizado inicial: procesa datos, aplica `reqInitialPM` si está definido, crea estructura DOM y llama a `reqRefresh` |

### Helpers de celda en la tabla REQ

| Función | Descripción |
|---|---|
| `diffCell(r)` | Días hábiles de diferencia vs hoy. Verde `+N días`, amarillo `Hoy`, rojo `-N días` |
| `deadlineCell(r)` | Fecha del deadline coloreada (rojo=atrasado, amarillo=hoy, verde=en tiempo) |
| `estadoPill(r)` | Badge de estado con clase CSS y texto correspondiente |

---

## Página: Control Tower

Primera página del dashboard. Agrega ambos boards (Iniciativas + REQ) en una sola vista y agrupa los datos por PM en tarjetas de portafolio. No tiene estado de filtros propio — solo consume los datos procesados al vuelo.

### Fuentes de datos

Hace tres fetches en paralelo al cargar:
1. Board de Iniciativas (`INI_BOARD_ID`)
2. Board de REQ (`REQ_BOARD_ID`)
3. Calendario de meetings (Google Apps Script)

Reutiliza `iniProcess`, `reqProcess` y `buildCalMap` de las otras páginas para no duplicar lógica.

### Lógica de salud por PM

Combina atrasados de Iniciativas + REQ para calcular el estado del portafolio:

| Condición | Estado |
|---|---|
| 0 atrasados en total | On Track (verde) |
| 1–2 atrasados en total | At Risk (amarillo) |
| 3+ atrasados en total | Off Track (rojo) |

### Funciones

| Función | Descripción |
|---|---|
| `overviewGoToIni(pm)` | Establece `iniInitialPM = pm` y llama `navigate("iniciativas")`. Si `pm` es `null`, navega sin filtro |
| `overviewGoToReq(pm)` | Establece `reqInitialPM = pm` y llama `navigate("req")`. Si `pm` es `null`, navega sin filtro |
| `loadOverview(container)` | Fetches en paralelo a ambos boards y al calendario; llama a `overviewRender` |
| `overviewRender(container, iniItems, reqItems)` | Construye el HTML completo: barra de resumen global + grid de tarjetas por PM |

### Estructura visual

```
┌─ Barra de resumen global ──────────────────────────────────────────┐
│  [Iniciativas → ver dashboard]   │   [REQ → ver dashboard]        │
│  N en proceso · N atrasadas · …  │   N en proceso · N atrasados … │
└────────────────────────────────────────────────────────────────────┘

┌─ PM Card ──────────────────────────────────────────────────────────┐
│  Nombre PM                              [● On Track / ⚠ At Risk]  │
├───────────────────────────┬────────────────────────────────────────┤
│  INICIATIVAS  Ver detalle→│  REQ               Ver detalle →      │
│  · N en proceso           │  · N en proceso                       │
│  · N atrasadas            │  · N atrasados                        │
│  · N para hoy             │  · N para hoy                         │
└───────────────────────────┴────────────────────────────────────────┘
```

Clic en la mitad de Iniciativas → `overviewGoToIni(pm)` → navega a Iniciativas filtrado por ese PM.
Clic en la mitad de REQ → `overviewGoToReq(pm)` → navega a REQ filtrado por ese PM.
Clic en la barra global → navega al dashboard correspondiente sin filtro de PM.

### Clases CSS específicas

| Clase | Descripción |
|---|---|
| `.pf-global` | Contenedor de la barra de resumen superior |
| `.pf-global-block` | Cada mitad de la barra (Iniciativas / REQ), clickeable |
| `.pf-global-sep` | Separador vertical entre bloques |
| `.pf-gstat` | Stat individual en la barra (número + label) |
| `.pf-grid` | Grid auto-fill de tarjetas de PM |
| `.pf-card` | Tarjeta de un PM, borde coloreado según salud |
| `.pf-header` | Fila superior de la card (nombre + badge de salud) |
| `.pf-boards` | Contenedor flex de las dos mitades |
| `.pf-section` | Mitad clickeable (Iniciativas o REQ) |
| `.pf-section.pf-empty` | Mitad sin items activos — no interactiva |
| `.pf-go` | Texto "Ver detalle →", visible solo en hover |
| `.pf-stats` | Lista de stats dentro de una sección |
| `.pf-stat` / `.pf-dot` | Fila de stat individual con punto de color |

---

## Cómo agregar una nueva página

1. Agrega una entrada al array `PAGES`:
```js
{ id: "mi-pagina", label: "Mi Página", icon: "◉", subtitle: "Descripción", load: loadMiPagina }
```

2. Implementa la función:
```js
async function loadMiPagina(container) {
  showLoader(container);
  try {
    const json = await mondayFetch(`{ boards(ids:[ID]) { ... } }`);
    // procesar y renderizar
    startCountdown(() => loadMiPagina(container));
  } catch(err) {
    showError(container, err.message);
  }
}
```

3. No es necesario ningún cambio más: el sidebar y la navegación se generan automáticamente.

---

## Esquema de colores de estado

| Estado | Color | Clase CSS |
|---|---|---|
| Atrasado | Rojo `#ef4444` | `pill-atrasado` |
| Para Hoy | Amarillo `#f59e0b` | `pill-parahoy` |
| En Tiempo | Verde `#10b981` | `pill-entiempo` |
| Aprobado | Morado `#6c63ff` | `pill-aprobada` |
| N/A / Skip | Gris | `pill-skip` |
