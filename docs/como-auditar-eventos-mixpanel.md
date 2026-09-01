# Cómo auditar nombres de eventos en Mixpanel (guía práctica)

Guía para encontrar por tu cuenta eventos duplicados, con nombres malformados o instrumentados dos veces. Los ejemplos usan el proyecto **Vana** (`3370988`), workspace *All Project Data* (`3877396`).

---

## Las 3 propiedades que lo revelan todo

Antes de cualquier query, hay que saber que **Segment adjunta metadata técnica a cada evento** que llega a Mixpanel. Tres propiedades son las que resuelven casi cualquier investigación:

| Propiedad | Qué te dice | Valores reales en vana |
|---|---|---|
| **`mp_lib`** | Qué SDK envió el evento | `Segment Actions: analytics.js` = **web** · `Segment Actions: analytics-kotlin` = **Android** |
| **`event_original_name`** | El nombre que la app envió **antes** de que Segment lo transformara | ← *la clave de todo* |
| `segment_source_name` | La source de Segment de origen | (frecuentemente vacío) |

Ver también `source_app` (`vana_android_app`, `vana_web_app`, `lms`, `casas_de_cobranza`…), `path` y `url` (qué pantalla web lo emitió) y `product_context` (`presta`, `pay`, `card`).

> **Por qué importa `event_original_name`:** el destino *Mixpanel (Actions)* en Segment construye el nombre de los eventos de pantalla como `Viewed {{name}}`. Comparar el nombre final con el original te dice **exactamente en qué punto del pipeline se ensució el nombre**.

### Cómo descubrir que estas propiedades existen

**Lexicon → Events → clic en cualquier evento → lista de propiedades.**
Lexicon: `https://mixpanel.com/project/3370988/view/3877396/app/lexicon`

O en Insights, al agregar un breakdown, escribe "original" o "lib" en el buscador de propiedades.

---

## Método 1 — Lexicon: ver *qué* nombres existen

Lexicon es el catálogo de eventos. Sirve para reconocer el problema, no para medirlo.

1. Ir a **Data Management → Lexicon → pestaña Events**.
2. Buscar `Viewed Viewed` → salen los 23 eventos de doble prefijo de inmediato.
3. **Para sufijos redundantes el buscador no alcanza** (busca "contiene", no "termina en"). Dos salidas:
   - **Ordenar por nombre (A–Z)** y escanear: los duplicados quedan pegados, porque comparten prefijo (`Viewed Bank Account Details`, `Viewed Bank Account Info`, `Viewed Bank Account Information`, `Viewed Bank Account Viewed`…). Con 177 eventos `Viewed *` es tedioso pero funciona.
   - **Exportar Lexicon a CSV** y filtrar en Google Sheets con regex (ver abajo).

### Fórmulas de Google Sheets para cazar patrones

Con la columna `A` = nombre del evento:

```
# cualquier palabra repetida en el nombre (atrapa "Viewed Viewed", "Wallet ... Wallet", etc.)
=REGEXMATCH(A2, "\b(\w+)\b.*\b\1\b")

# termina en un verbo de visualización → sufijo redundante
=REGEXMATCH(A2, "(Viewed|Displayed|Shown)$")

# espacios dobles (atrapa "Viewed Bank Ticket  Details")
=REGEXMATCH(A2, "  ")

# mismo nombre con distinto casing → duplicado invisible
=COUNTIF(ARRAYFORMULA(LOWER($A$2:$A$600)), LOWER(A2)) > 1
```

---

## Método 2 — Insights: el detector definitivo

Este es el método que encuentra **todos** los eventos malformados de un golpe, incluidos los que aparezcan en el futuro. La lógica: *si el nombre que la app envió ya contiene "Viewed", y ese evento pasa por el mapping que antepone "Viewed ", el resultado está duplicado.*

**Insights → New report** y configurar:

| Campo | Valor |
|---|---|
| Metric | `All Events` · measurement **Total events** |
| Filter | `event_original_name` **contains** `Viewed` |
| Breakdown 1 | `event_original_name` |
| Breakdown 2 | `mp_lib` (aparece como *Mixpanel Library*) |
| Date range | Last 30 days |
| Chart | Table o Bar |

Reporte ya construido: `https://mixpanel.com/project/3370988/view/3877396/app/insights#62VPeCSj7QxB`

> **La forma más rápida de aprenderlo:** abre ese reporte y desármalo. Toda la configuración está en el panel izquierdo; cambiando un solo campo (el valor del filtro) obtienes una variante nueva.

### Click por click, desde cero

| # | Acción | Por qué |
|---|---|---|
| 00 | Verifica arriba a la izquierda que el proyecto sea **Vana** y el workspace **All Project Data** | En *Vana QA* el reporte sale vacío |
| 01 | Botón **+ New → Insights**, o la URL directa `/project/3370988/view/3877396/app/insights` | La URL es a prueba de cambios de UI |
| 02 | En la sección de métricas (fila **A**), clic en el selector de evento → escribe **All Events** | Va `All Events` a propósito: todavía no sabemos qué eventos están mal, y eso es lo que queremos descubrir. Filtrar por propiedad sobre «todos los eventos» es el modo exploración |
| 03 | En la misma fila, cambia la medición a **Total events** (si dice *Unique users*) | *Unique users* cuenta personas; queremos contar **ocurrencias** |
| 04 | **+ Filter** → en el buscador escribe `event_original_name`, con guiones bajos y tal cual | Esta propiedad **no tiene display name**, aparece con su nombre técnico en *Event Properties* |
| 05 | Cambia el operador a **contains** (entra en *equals*) → valor **Viewed** → Enter | Respeta la mayúscula inicial |
| 06 | **+ Breakdown** → `event_original_name` | Abre una fila por cada nombre original distinto: lo que las apps están mandando |
| 07 | **+ Breakdown** otra vez → **Mixpanel Library** (busca «lib») | Ese es `mp_lib`, y **sí tiene display name**, por eso aparece como *Mixpanel Library*. El orden importa: nombre primero, SDK después |
| 08 | Selector de fechas arriba a la derecha → **Last 30 days** | |
| 09 | Tipo de gráfico → **Table** | Puedes ordenar por volumen desde el encabezado y exportar a CSV |
| 10 | Lee cada fila: *¿este nombre original ya trae «Viewed»?* Si sí, está mal — y la columna de SDK dice a qué equipo llamar | |
| 11 | **Save** → nómbralo y mándalo a un board | Para revisarlo cada sprint y ver si el volumen sube o baja |

### Si algo no sale

| Síntoma | Causa |
|---|---|
| Cero filas | El operador quedó en *equals*; debe ser *contains* |
| `event_original_name` no aparece en el buscador | Solo existe en eventos que pasan por Segment. Confirma proyecto **Vana** (3370988) y que el rango de fechas cubra días con tráfico |
| Los números salen muy bajos | La medición quedó en *Unique users*; cámbiala a *Total events* |
| Demasiadas filas | Ordena por volumen, o agrega un filtro por `product_context` para ver un producto a la vez |

**Repetir el mismo query cambiando el filtro a `Displayed`** para atrapar `Orders In Progress Displayed` y `Primary Offer Displayed`. Igual con `Shown`, `Screen`, `Page` si sospechas de otras convenciones.

### Variante: contar el daño de un patrón conocido

Si ya sabes el patrón y solo quieres el volumen y el SDK:

| Campo | Valor |
|---|---|
| Filter | `Event Name` **contains** `Viewed Viewed` |
| Breakdown | `Event Name` + `mp_lib` |

Reportes ya construidos:
- Doble prefijo por SDK: `https://mixpanel.com/project/3370988/view/3877396/app/insights#pvbqThFhnfMX`
- Sufijos redundantes por SDK: `https://mixpanel.com/project/3370988/view/3877396/app/insights#Aja9wQpAApmX`

---

## Método 3 — Interpretar: bug de pipeline vs. problema de convención

Aquí está el paso que la mayoría se salta. **Compara el nombre final contra `event_original_name`:**

| Si… | Significa | Dónde se arregla |
|---|---|---|
| final = `"Viewed " + original` | pasó por `page()` / `screen()` y el mapping le puso el prefijo | **Segment** (Insert Function) + la app que manda el nombre pre-formateado |
| final = original exacto | fue un `track()`; Segment no lo tocó | **el código de la app** (convención de naming) |

### Ejemplo real, con los dos casos lado a lado

| `event_original_name` | Evento final en Mixpanel | SDK | Veredicto |
|---|---|---|---|
| `VRM Viewed` | `Viewed VRM Viewed` | web | bug de pipeline — la web manda nombre pre-formateado a `page()` |
| `Notification Inbox` | `Viewed Notification Inbox` | Android + web | ✅ correcto |
| `Notification Inbox Viewed` | `Notification Inbox Viewed` | web | `track()` — **doble instrumentación**, no bug de pipeline |

**Hallazgo que salió de aquí:** la misma pantalla del inbox de notificaciones está instrumentada **dos veces desde la web**:
- `analytics.page("Notification Inbox")` → `Viewed Notification Inbox` (90,625 desde web + 617,809 desde Android)
- `analytics.track("Notification Inbox Viewed")` → `Notification Inbox Viewed` (775,022, solo web)

Son ~1.48 M eventos/mes para una sola vista de pantalla, contados dos veces con nombres distintos. Android solo emite el correcto. **Acción:** la web debe eliminar el `track()` redundante y quedarse con `page()`.

---

## Método 4 — Confirmar con un evento crudo

Para cerrar la investigación y darle al equipo de desarrollo la ubicación exacta:

1. Ir al explorador de **Events** (el stream de eventos crudos en el nav lateral).
2. Filtrar por el nombre del evento sospechoso.
3. Abrir una ocurrencia individual y revisar sus propiedades.
4. Las propiedades **`path`** y **`url`** te dicen la ruta exacta de la web que lo emitió — eso es lo que le pasas al equipo web para que encuentre la línea de código.

Ejemplo: el evento huérfano `Viewed` (3,995/30d) trae `path` = `/nps` y `/pay/nps` → son esas dos páginas las que llaman `analytics.page()` sin nombre.

---

## Receta reutilizable (para cualquier duplicado, no solo `Viewed`)

1. **Sospecha** → detecta el patrón en Lexicon (buscar / ordenar A–Z / exportar a Sheets con regex).
2. **Mide** → Insights con filtro `contains` sobre `Event Name`, breakdown por `Event Name`.
3. **Atribuye** → agrega breakdown por **`mp_lib`** (¿qué SDK?) y **`source_app`** / **`product_context`** (¿qué app y producto?).
4. **Diagnostica** → agrega breakdown por **`event_original_name`** y compara con el nombre final. Ahí se ve si el problema es del pipeline de Segment o del código de la app.
5. **Localiza** → explorador de Events → propiedades `path` / `url` del evento crudo.
6. **Documenta** → nombre canónico + dónde se arregla + cómo verificar que el volumen del malo cae a cero.

### Regla mental para no perderse

> `mp_lib` responde **quién lo mandó**.
> `event_original_name` responde **qué mandó**.
> La diferencia entre ese nombre y el final responde **quién lo ensució**.
