# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Resumen del proyecto

Canopy v2.0 es un sitio estático de Business Intelligence para editoriales. El repositorio contiene únicamente dos archivos HTML autocontenidos (sin build system, sin `package.json`, sin tests):

- `index.html` — landing page de marketing (en español, ~888 líneas). Secciones: hero, ventajas, módulos, análisis, stats, quienes somos, CTA. Ancla de descarga que apunta a `canopy-demo-v2.html`.
- `canopy-demo-v2.html` — aplicación demo completa (~7288 líneas). Es una SPA monolítica con CSS y JS inline, datos de ejemplo embebidos, sidebar de navegación y 13 páginas.

## Cómo correr / desarrollar

No hay build ni dependencias locales. Todo se sirve directo desde el filesystem:

```bash
# Abrir la landing en el navegador
xdg-open index.html        # o: open index.html en macOS

# O servir la carpeta (recomendado para que fetch/CDN funcionen sin problemas)
python3 -m http.server 8000
# luego abrir http://localhost:8000/
```

El demo (`canopy-demo-v2.html`) carga dos librerías por CDN (verificar conectividad si algo no renderiza):

- `xlsx.full.min.js` (SheetJS 0.18.5) — exportación a Excel.
- `Chart.js 4.4.1` — gráficos (donut, barras, líneas, scatter).

No hay linter, formateador ni tests configurados. Cualquier cambio se valida abriendo los archivos en el navegador y revisando la consola.

## Arquitectura del demo (`canopy-demo-v2.html`)

Todo vive en un único archivo. Entender estas piezas antes de editar ahorra tiempo:

### Datos

- `const DATA = { … }` declarado alrededor de la línea **1644**. Contiene todos los datos de ejemplo embebidos como literales JSON: `stock`, `consig`, `estados`, `cli_summary`, `ventas`, `ventas_mensual`, `analisis`, `config`, etc.
- `DATA.config` puede incluir `editorial_nombre` y `vendedor`. Si `vendedor` está seteado se activa "modo vendedor": se oculta la pestaña de Liquidaciones, se agrega un badge "VISTA DE X" en el topbar y se ocultan gráficos de vendedores (ver `init()` ~línea 1715).

### Estado de UI

- `const S = { … }` alrededor de la línea **1658**. Guarda paginación, ordenamiento y filtros por página (`S.consig`, `S.stock`, `S.estados`, `S.clientes`, `S.liq`, `S.rot`, `S.analisis`, `S.radar`). `S.cur` es la página activa.
- Utilidades globales: `$ = id => document.getElementById(id)`, `fmt` (formato corto `$1.2M`/`$45K`), `fmtF` (formato es-AR completo), `fmtD` (fecha `DD/MM/YYYY`), `isVenc`/`isProx` (vencimientos).

### Navegación (SPA)

- Sidebar + mobile-nav disparan `goPage(name, el, isMob)` (~línea 3311). Alterna `.page.active` sobre `#page-<name>`, actualiza título y oculta el botón de export global en páginas donde no aplica (`dashboard`, `radar`, `novedades`).
- Páginas definidas como `<div class="page" id="page-<name>">`: `dashboard`, `radar`, `novedades`, `consignaciones`, `stock`, `rotacion`, `estados`, `cobranza`, `clientes`, `ventas_mensual`, `liquidaciones`, `analisis`, `indicadores`.
- Algunas vistas necesitan re-render al activarse (Chart.js requiere dimensiones visibles): `cobranza`, `clientes` (treemap), `analisis` (scatter), `indicadores` (scatter). `goPage` ya lo maneja — replicar ese patrón si agregas una vista con gráficos.

### Render por sección

Cada página tiene su función `renderXxx()` llamada desde `init()` dentro de un wrapper `_safe()` que loguea errores sin romper las demás:

- `renderDash`, `renderNovedades`, `renderConsig`, `renderStock`, `renderRotacion`, `renderEC` (estados), `renderCli`, `renderVentasMensual`, `renderLiquidaciones`, `renderCobranza`, `initAnalisis`.
- `init()` también enriquece `DATA.stock` con `consignados`, `ventas_acum`, `ratio_cv` y `ratio_vc` a partir de `DATA.consig` y `DATA.ventas.top_titulos`. Si agregas campos derivados, este es el lugar.

### Gráficos

- `_mkCanvas(containerId, heightPx)` (~línea 1682) destruye la instancia previa registrada en `_CI[containerId]`, limpia el contenedor y devuelve un `<canvas>` nuevo. **Siempre** usar esta helper al crear gráficos: evita fugas de canvas al re-renderizar.
- Paleta en `const _P` (~línea 1705) — tonos verdes editoriales (`#1B3A2D` gold, `#4CAF82` green-secundario, `#27AE60` green, `#3498DB` blue, `#F39C12` orange, `#E74C3C` red).

### Exportación

- Botón `.btn-export-global` llama a `exportCurrentPage()` (~línea 3249). Hay un `switch` implícito por `S.cur` que define headers y filas. **Si agregás una página exportable, extender ese bloque.**
- Helper `dlXlsx(rows, header, filename)` usa SheetJS; alias `dlCSV` por compatibilidad (a pesar del nombre, exporta `.xlsx`).

### Persistencia

- `localStorage` se usa solo para marcar novedades leídas (`_NOV_STORAGE_KEY`, funciones `_novGetLeidas`/`_novSaveLeidas`).

## Convenciones

- **Idioma**: todo el contenido de UI está en español rioplatense (Argentina). Mantener tildes y ortografía al agregar strings.
- **Moneda**: pesos argentinos, separador de miles es-AR (`fmtF`). Usar `fmt` para KPIs compactos.
- **Fechas**: formato ISO `YYYY-MM-DD` en datos, se muestra `DD/MM/YYYY` vía `fmtD`.
- **CSS**: variables en `:root` al inicio de `<style>`. Reutilizar tokens (`--gold`, `--green`, `--border`, `--r` para radius) en vez de hardcodear colores.
- **Responsive**: hay breakpoints `@media(max-width:768px)`; la navegación móvil vive en `.mobile-nav` al final del `<body>`.

## Reglas de edición

- Los dos HTML son autocontenidos por diseño (se distribuyen como descarga única desde la landing). No dividirlos en archivos separados salvo que el usuario lo pida explícitamente.
- Al tocar una función `renderXxx`, asegurate que quede tolerante a campos faltantes (patrón `?.` y `|| []` usado en todo el código) — `DATA` puede variar entre demos.
- Al crear/re-crear un gráfico Chart.js usar siempre `_mkCanvas` para limpiar la instancia anterior.
- Si agregás un nuevo `data-page` / `<div class="page">`, sumarlo al diccionario `titles` dentro de `goPage` y, si tiene gráficos, al bloque de re-render post-navegación.

## Git

- Branch de trabajo actual: `claude/create-claude-md-iZExF`. Desarrollar y pushear a la rama indicada; `git push -u origin <branch>`.
