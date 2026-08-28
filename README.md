# user-metrics

Instrumentación centralizada de eventos para la tribu **User & Growth** de vana.

## Objetivo

Tener un mapeo general de **todos** los eventos que existen en vana y sus productos (vana pay, vana presta, vana card), detectar eventos duplicados con nombres distintos entre productos, y converger hacia una taxonomía canónica única gobernada por la tribu User.

## Contenido

| Archivo | Descripción |
|---|---|
| [`docs/event-mapping.md`](docs/event-mapping.md) | Análisis completo: inventario, clusters de duplicados, problemas de naming y taxonomía canónica propuesta |
| [`docs/dashboard/index.html`](docs/dashboard/index.html) | **Dashboard visual** (autocontenido, listo para GitHub Pages): KPIs, desglose por producto, clusters de duplicados y explorador de los 534 eventos |
| [`data/mixpanel_event_inventory.csv`](data/mixpanel_event_inventory.csv) | Inventario crudo: 551 eventos con volumen de 30 días, metadata de Lexicon (verified/hidden/dropped, tags, descripción) |
| [`data/mixpanel_event_by_product.csv`](data/mixpanel_event_by_product.csv) | Matriz evento × producto (`product_context`): 640 filas con volumen de 30 días |

### Ver el dashboard en GitHub Pages

El dashboard es un HTML autocontenido (datos embebidos, sin backend). Para publicarlo: **Settings → Pages → Deploy from a branch**, elegir la rama y la carpeta `/docs`; quedará en `https://<org>.github.io/user-metrics/dashboard/`. También se puede abrir el archivo localmente en el navegador.

## Fuente de datos

- **Mixpanel** (extraído 2026-08-28, ventana de volumen: últimos 30 días):
  - Proyecto `Vana` (id 3370988) — proyecto principal, con workspaces *vana pay*, *vana presta* y *vana card*. 544 eventos, ~54M ocurrencias/30d.
  - Proyecto `Vana Pay` (id 3449788) — **legado, prácticamente muerto** (24 eventos en 30d, solo auto-tracking).
  - Proyectos `Vana QA` y `Vana Pay QA` — ambientes de prueba, fuera del alcance.
- **Segment**: pendiente. Cuando haya un token de la Public API de Segment se cruzará este inventario contra los *sources* y *tracking plans* para atribuir cada evento a su producto/equipo de origen.

## Cómo regenerar el inventario

El inventario se extrae vía el conector MCP de Mixpanel (Lexicon `Get-Events` + Insights `$all_events` con breakdown por `$event_name`). Ver detalles de la metodología en `docs/event-mapping.md`.
