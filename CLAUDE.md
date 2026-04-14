# Calendario Formaciones Kite — CLAUDE.md

Resumen técnico del proyecto construido en esta conversación.

---

## Qué es

Repositorio de formaciones dadas a clientes, originalmente un Excel convertido a HTML estático. Se amplió con un formulario para añadir nuevas formaciones de forma persistente y compartida entre usuarios, sin servidor ni base de datos propia.

---

## Arquitectura

```
Usuario
  │
  ▼
GitHub Pages  ──────────────────────────────────────────────────
  index.html                                                    │
  ├── Datos históricos: hardcodeados en el HTML (<tr> estáticos)│
  └── Datos nuevos: cargados al abrir vía fetch GET             │
                                                                │
                          Google Apps Script (doGet / doPost)  │
                                    │                           │
                                    ▼                           │
                          Google Sheets                         │
                          Hoja: "Formaciones"                   │
                          Columnas: cliente, formador,          │
                                    fecha, caso, tipo           │
──────────────────────────────────────────────────────────────────
```

---

## URLs

| Recurso | URL |
|---|---|
| App pública | https://mikylace-cell.github.io/formaciones-kite/ |
| Repositorio GitHub | https://github.com/mikylace-cell/formaciones-kite |
| Fichero principal | `index.html` en rama `main` |

---

## Ficheros

| Fichero | Descripción |
|---|---|
| `index.html` | Aplicación completa (HTML + CSS + JS en un solo fichero) |

---

## Google Apps Script

El script actúa como API REST mínima sobre Google Sheets.

### doGet — leer formaciones
Devuelve todas las filas de la hoja como JSON. Las cabeceras se normalizan a minúsculas. Las fechas se formatean con `Utilities.formatDate` en zona `Europe/Madrid` → `yyyy-MM-dd`.

### doPost — guardar formación
Recibe un JSON con `{cliente, formador, fecha, caso, tipo}` y hace `appendRow`. La fecha se guarda con prefijo `'` para evitar que Sheets la convierta a tipo Date.

### Publicar cambios
Cada vez que se modifica el script hay que crear una nueva versión:
**Implementar → Administrar implementaciones → lápiz → Nueva versión → Implementar**

---

## JavaScript relevante en index.html

### Constante de configuración
```javascript
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbw_.../exec';
```

### Carga al abrir
```javascript
fetch(APPS_SCRIPT_URL)
  .then(r => r.json())
  .then(list => { list.forEach(addRowFromData); sortRows(); update(); update2(); })
  .catch(() => {});
```

### Envío del formulario
```javascript
fetch(APPS_SCRIPT_URL, {
  method: 'POST',
  mode: 'no-cors',          // necesario para evitar error CORS con Apps Script
  headers: { 'Content-Type': 'text/plain' },
  body: JSON.stringify(data)
})
```
> `mode: 'no-cors'` devuelve respuesta opaca (sin body), por eso el `.then` no intenta parsear JSON sino que asume éxito.

---

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `Failed to fetch` abriendo desde `file://` | El navegador bloquea fetch desde origen local | Alojar en GitHub Pages (`https://`) |
| Google Drive mostraba el HTML como texto | Drive hace preview, no ejecuta JS | Usar GitHub Pages en su lugar |
| Condición "no configurado" siempre activa | El placeholder se comparaba con la URL real (mismo valor) | Eliminar la bifurcación, URL hardcodeada directamente |
| Registros nuevos no aparecían al refrescar | Cabeceras en Sheets en mayúsculas, JS esperaba minúsculas | `.toLowerCase().trim()` en el `doGet` |
| Fechas con formato `2026-04-28T22:00:00.000Z` | Sheets convierte strings de fecha a tipo Date | Guardar con prefijo `'` en `doPost`; formatear con `Utilities.formatDate` en `doGet` |
| Cambios en Apps Script sin efecto | No se había creado nueva versión al publicar | Siempre crear "Nueva versión" al reimplementar |

---

## Flujo para añadir una formación

1. Usuario abre la URL de GitHub Pages
2. Al cargar, el JS hace GET al Apps Script y añade las filas nuevas a la tabla
3. Usuario rellena el formulario (tab "+ Nueva Formación") y envía
4. JS hace POST al Apps Script → Apps Script hace `appendRow` en Sheets
5. La fila se añade visualmente a la tabla sin recargar
6. Cualquier otro usuario que abra o refresque la página verá el nuevo registro

---

## Mantenimiento

- **Añadir formadores**: editar el `<select id="f-formador">` en `index.html` y hacer commit en GitHub
- **Datos históricos**: están hardcodeados en el HTML como `<tr>` — para añadir registros pasados, editar directamente el HTML o añadirlos a Sheets (aparecerán al cargar)
- **Backup**: la hoja de Google Sheets es el único almacén de datos nuevos — no borrarla
