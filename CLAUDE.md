# OVERLOAD — Gym Tracker: Contexto completo de desarrollo

## Descripción del proyecto
App web de seguimiento de entrenamiento personal. **Un único archivo HTML** (`index.html`) con CSS y JS embebidos. Sin frameworks, sin build system, vanilla JS. UI en español.

- **Producción:** `https://martins7777.github.io/gym-tracker/`
- **Repo GitHub:** `https://github.com/Martins7777/gym-tracker`
- **Directorio de trabajo local:** `c:\Users\PC\OneDrive\Documentos\gym-tracker-github\`

## Flujo de trabajo con Git
1. Editar `index.html` directamente en el directorio local
2. Ejecutar en terminal para publicar:
   ```bash
   cd "C:\Users\PC\OneDrive\Documentos\gym-tracker-github"
   git add index.html
   git commit -m "descripción del cambio"
   git push origin main
   ```
3. GitHub Pages despliega automáticamente — el móvil ve los cambios al instante

---

## Arquitectura

### Archivo único
Todo el código vive en `index.html`:
- Estilos en `<style>`
- Lógica en `<script>`
- HTML estructural en `<body>`

No hay módulos, imports ni bundler.

### Estado global (usuario activo)
```js
var workouts = [];  // sesiones del usuario activo
var medidas  = [];  // medidas corporales del usuario activo
```
Estos arrays **siempre pertenecen al usuario activo**. Son la única fuente de verdad en memoria.

### Autenticación y sesión
- Login con selector de usuario + contraseña
- Sesión guardada en `sessionStorage` con clave `SESSION_KEY`
- Variables globales: `SESSION_ROLE`, `SESSION_UID`, `SESSION_NAME`
- Rol `'admin'` → ve todos los usuarios y puede cambiar entre ellos
- Rol `'user'` → solo ve sus propios datos
- `getContextUid()` → devuelve el uid del usuario cuyo contexto está activo (admin puede ver otro usuario)
- La selección de usuario en el selector del login se preserva entre el paso "elegir usuario" y el paso "introducir contraseña" (via `prevSel`)

### Multi-usuario (localStorage cache)
- `localStorage['gym_users']` → `[{id, name}]`
- `localStorage['gym_active_uid']`
- `localStorage['gym_w_{uid}']` → workouts en JSON
- `localStorage['gym_m_{uid}']` → medidas en JSON
- **`switchUser(id)` es la ÚNICA función que cambia el usuario activo**

### Google Sheets (backend cloud)
- URL del Apps Script guardada en `localStorage['gym_gs_url']`
- `gsRefresh()` → GET: carga datos del usuario activo desde GS
- `gsSaveAll()` → POST: sube datos de TODOS los usuarios a GS
- `gsLoadAllWorkouts()` → carga workouts de todos los usuarios (para autocomplete)
- `_syncToken` previene race conditions (se incrementa al cambiar de usuario)
- Sync **siempre manual** — no hay auto-save ni auto-refresh
- Endpoints:
  - GET: `?action=getWorkouts&user={uid}` / `?action=getMedidas&user={uid}` / `?action=getAllWorkouts`
  - POST body: `{ action: 'saveWorkouts'|'saveMedidas', user: uid, data: [] }`

### Chunking en Google Sheets (IMPORTANTE)
Google Sheets limita las celdas a ~50.000 caracteres. JSON grandes (como importaciones masivas de datos) se fragmentan en filas adicionales.

El script de Apps Script (`google-appscript.txt`) implementa chunking con:
- Fila principal: `[uid, fragment_0]`
- Filas adicionales: `[uid:chunk:1, fragment_1]`, `[uid:chunk:2, fragment_2]`, ...
- `CHUNK_SIZE = 45000` caracteres por celda
- `setData()`, `getData()`, `getAllWorkouts()` manejan el chunking transparentemente

**Si el usuario tiene muchos datos y falla al guardar en GS → verificar que el script desplegado tiene la versión con chunking.**

---

## Secciones de la UI

| ID | Función |
|---|---|
| `sec-log` | Registro de ejercicio (pantalla por defecto) |
| `sec-history` | Historial de sesiones |
| `sec-progress` | Progreso con gráfico Chart.js |
| `sec-medidas` | Medidas corporales con gráfico |
| `sec-stats` | Estadísticas globales + Resumen personal (1RM) |
| `sec-io` | Config: URL GS, gestión usuarios, backup JSON |

Navegación: `showSection(id)` → oculta todas, muestra la seleccionada, llama a `triggerSectionRender(id)`.

---

## Estructura de datos

### Workout / Sesión
```js
{
  id: Date.now(),       // timestamp como ID
  date: "YYYY-MM-DD",  // siempre formato ISO local
  exercises: [
    { id: Date.now(), name: String, weight: String, sets: String, reps: String }
  ]
}
```

### Medida
```js
{ id: Date.now(), date: "YYYY-MM-DD", hombros?, pecho?, biceps?, cintura?, muslo?, gemelo? }
```
Campos opcionales — solo se guardan los que tienen valor. Todas en cm.

### Backup JSON (v2)
```js
{ version: 2, users: [{id, name}], data: { [uid]: { name, workouts, medidas } } }
```

---

## Convenciones de código

- **ES5 compatible**: `var`, `function`, sin arrow functions, sin template literals, sin `const`/`let`
- Excepción: `async/await` en funciones GS (`gsRefresh`, `gsSaveAll`, import)
- IDs de elementos HTML: camelCase (`exName`, `workoutDate`, `syncBanner`)
- IDs de filas editables: `row-{section}-{eid}` (`row-hist-123`, `row-prog-123`)
- Tras cualquier mutación de datos: `saveActiveUserToCache()` + `markUnsaved()` + `showBanner(msg, 'ok')`
- Gráficos Chart.js: siempre destruir antes de recrear (`chartInstance.destroy()`)

---

## CSS

### Variables
```css
--primary: #2563eb   /* azul principal */
--dark:    #1e293b   /* fondo nav, headers */
--light:   #f8fafc   /* fondo body */
--danger:  #ef4444   /* rojo */
--accent:  #f59e0b   /* amarillo/naranja */
```

### Breakpoint responsive
`@media (min-width: 600px)` — mobile-first. En mobile: nav en columna, grid de inputs en 1 columna.

### Clases clave
- `.card` — contenedor blanco con sombra
- `.hidden` — `display: none`
- `.btn`, `.btn-primary`, `.btn-danger` — botones
- `.session-card`, `.ex-row` — filas del historial
- `.ex-pill` — chip de datos (peso, series×reps)
- `.ex-name` — nombre de ejercicio azul+subrayado, clickable → navega a Progreso
- `.sync-dot` con `.ok`/`.err`/`.spin` — indicador de estado GS
- `#syncBanner` con `.show .ok`/`.err`/`.busy` — banner flotante
- `.muscle-group-header` — fila separadora por grupo muscular en tabla de stats
- `.ex-sug-item`, `.ex-sug-match` — autocomplete de ejercicios

---

## Funciones clave

| Función | Propósito |
|---|---|
| `addExercise()` | Añade ejercicio al workout del día, guarda y actualiza UI |
| `deleteSession(id)` | Elimina sesión completa por ID |
| `deleteExercise(sid, eid)` | Elimina ejercicio de una sesión |
| `enableEdit(sid, eid, section)` | Convierte fila en modo edición inline |
| `saveEdit(sid, eid, section)` | Persiste edición inline |
| `renderHistory()` | Re-renderiza sección Historial |
| `renderChart()` | Re-renderiza gráfico de progreso (solo peso kg, con reps encima de cada punto) |
| `renderStats()` | Re-renderiza estadísticas (KPIs + Resumen personal agrupado por músculo) |
| `renderMedidasChart(key)` | Re-renderiza gráfico de medida específica |
| `renderExHistoryPanel(name)` | Panel "últimas sesiones" al escribir en el input de ejercicio |
| `buildAllExerciseNames()` | Construye lista de todos los ejercicios (usuario activo + todos los usuarios de GS) |
| `showExSuggestions(query)` | Muestra dropdown de autocomplete con nombres de ejercicios |
| `hideExSuggestions()` | Oculta el dropdown de autocomplete |
| `updateExerciseSelectors()` | Llama a `buildAllExerciseNames()` |
| `getMuscleGroup(name)` | Clasifica un ejercicio en grupo muscular |
| `fixEncoding(s)` | Corrige Mojibake UTF-8/Latin-1 (`decodeURIComponent(escape(s))`) |
| `refreshAllUI()` | Re-renderiza todo con los datos del usuario activo |
| `showBanner(msg, type)` | Notificación flotante. type: `'ok'`/`'err'`/`'busy'` |
| `todayLocal()` | Fecha local como `YYYY-MM-DD` (sin desfase UTC) |
| `normDate(raw)` | Normaliza fechas de diversas fuentes a `YYYY-MM-DD` |
| `fmtDate(iso)` | `YYYY-MM-DD` → `DD/MM/YYYY` para mostrar |
| `exportData()` | Descarga backup JSON v2 del usuario activo |
| `completeLogin(role, uid, name)` | Finaliza login con animación de transición |
| `doLogout()` | Cierra sesión con animación de vuelta al login |
| `goToProgress(name)` | Navega a sección Progreso y selecciona el ejercicio por nombre |
| `gsRefresh()` | Carga datos del usuario activo desde Google Sheets |
| `gsSaveAll()` | Sube datos de todos los usuarios a Google Sheets |
| `gsLoadAllWorkouts()` | Carga workouts de todos los usuarios (para autocomplete global) |

---

## Gráfico de Progreso (sec-progress)

- **Solo muestra peso (kg)** — no hay comparativa de repeticiones como dataset separado
- Las **repeticiones se muestran encima de cada punto** del gráfico como texto (`Xr`)
- Implementado via plugin Chart.js custom: `_repsLabelPlugin` (id: `'repsLabels'`)
- Fuente del label: `'600 10px sans-serif'`, color igual al de la línea del usuario
- Posición: `textBaseline = 'bottom'`, 5px encima del punto (`point.y - 5`)
- Multi-usuario: selector de botones coloreados por usuario, comparativa simultánea

---

## Sección Estadísticas (sec-stats)

### KPIs
- **Sesiones**: número total de sesiones del usuario activo
- **Ejercicios**: número de ejercicios **distintos** (únicos, no acumulado)

### Resumen personal
- Título: "Resumen personal" (subtítulo: fórmula Epley — w·(1+r/30))
- Columnas: Ejercicio | Última serie | 1RM estimado
- Ordenado por 1RM descendente dentro de cada grupo muscular
- Agrupado por músculo con separadores `.muscle-group-header`

### Grupos musculares
```js
var MUSCLE_ORDER = ['Pecho', 'Espalda', 'Hombro', 'Pierna', 'Brazo', 'Otros'];
```
Clasificación via `getMuscleGroup(name)` — normaliza acentos antes de comparar con regex.

### Nombres de ejercicio clickables
- Clase `.ex-name` → azul + subrayado
- `onclick="goToProgress('nombre')"` → navega a Progreso con ese ejercicio seleccionado

---

## Autocomplete de ejercicios

- Reemplaza el `<datalist>` nativo con un dropdown custom `#exSuggestionsBox`
- `allExerciseNames` → array global con nombres de todos los usuarios (activo + GS)
- Se construye en `buildAllExerciseNames()`, llamado en `postLoginInit()` y `updateExerciseSelectors()`
- `gsLoadAllWorkouts()` se ejecuta en background en `postLoginInit()` para cargar ejercicios de todos los usuarios
- Muestra hasta 25 resultados, máximo 30 si query vacía
- Resalta el texto coincidente con `<span class="ex-sug-match">`
- Cierra con Escape, Enter, o click fuera

---

## Animaciones de login

### Login → App
```
loginScreen.classList.add('exiting')  // opacidad 0, scale 1.04, 280ms
→ setTimeout(280ms)
→ loginScreen.hidden, nav visible, appMain visible
→ appMain.classList.add('app-enter')  // slideIn desde abajo, 280ms
→ animationend → remove 'app-enter'
```

### App → Login (logout)
```
loginScreen.classList.remove('hidden')
loginScreen.classList.add('login-enter')  // fadeIn desde scale 0.97, 240ms
→ animationend → remove 'login-enter'
```

---

## Encoding / Mojibake

- Algunos datos importados desde Excel tienen caracteres Mojibake (ej. `"JalÃ³n"` → `"Jalón"`)
- `fixEncoding(s)` los corrige: `decodeURIComponent(escape(s))`
- Se aplica en: carga desde GS (`gsRefreshUser`), importación de JSON, y render en `renderStats()`

---

## Google Apps Script (backend)

Archivo de referencia: `c:\Users\PC\OneDrive\Documentos\gym-tracker\google-appscript.txt`

Hojas que gestiona:
- `users` → col A=uid, col B=nombre, col C=contraseña
- `workouts` → col A=uid/uid:chunk:N, col B=JSON fragmentado
- `medidas` → col A=uid/uid:chunk:N, col B=JSON fragmentado

Acciones GET: `getUsers`, `getUsersAdmin`, `getWorkouts`, `getMedidas`, `getAllWorkouts`
Acciones POST: `login`, `saveWorkouts`, `saveMedidas`, `saveUsers`

Contraseña admin: `7777`

**Despliegue:** script.google.com → Implementar → Aplicación web → Ejecutar como: Yo → Acceso: Cualquier usuario

---

## Dependencias externas
- `chart.js` vía CDN: `https://cdn.jsdelivr.net/npm/chart.js`
- Sin otras dependencias

---

## Notas importantes

- No hay auto-save ni auto-refresh a GS — siempre manual
- `Date.now()` como ID — suficiente para uso personal
- Al añadir nueva sección: registrarla en `showSection()` y `triggerSectionRender()`
- El archivo `gym-tracker-v14.html` (en la carpeta antigua) es versión anterior de referencia, no activa
- La carpeta `c:\Users\PC\OneDrive\Documentos\gym-tracker\` es la versión local antigua — ya no se usa
- **El directorio de trabajo activo es `gym-tracker-github\`** — todos los cambios van aquí
