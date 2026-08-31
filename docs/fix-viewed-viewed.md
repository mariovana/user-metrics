# Runbook: el bug del doble prefijo `Viewed Viewed`

**Diagnóstico confirmado con datos de Mixpanel el 2026-08-31.** El doble prefijo **se agrega en Segment**, no en Mixpanel, y el origen del dato defectuoso es la **app web** (`analytics.js`).

## La mecánica del bug

El destino **Mixpanel (Actions)** en Segment tiene un mapping para las llamadas `page()`/`screen()` que construye el nombre del evento como `Viewed {{ name }}`. Eso está bien — es lo que convierte `screen("Loan Details")` de Android en el evento `Viewed Loan Details`. El problema es que cada plataforma le pasa el `name` en un formato distinto:

| Evento en Mixpanel | `mp_lib` (SDK de origen) | `event_original_name` (lo que envió la app) | Resultado |
|---|---|---|---|
| `Viewed Loan Details` ✅ | `Segment Actions: analytics-kotlin` (**Android**) | `Loan Details` | correcto |
| `Viewed Viewed Loan Details` ❌ | `Segment Actions: analytics.js` (**Web**) | `Viewed Loan Details` | doble prefijo |
| `Viewed Viewed Card Statement` ❌ | `Segment Actions: analytics.js` (**Web**) | `Viewed Card Statement` | doble prefijo |
| `Viewed Bank Account Viewed` ❌ | `Segment Actions: analytics.js` (**Web**) | `Bank Account Viewed` | sufijo redundante |
| `Viewed` (huérfano, 3.9k/30d) ❌ | `Segment Actions: analytics.js` (**Web**, páginas `/nps` y `/pay/nps`) | *(sin name)* | prefijo sin nombre |

- **Android** pasa el nombre de pantalla limpio → el mapping de Segment lo prefija una vez → correcto.
- **Web** pasa nombres que ya vienen en formato "evento" (`Viewed X`, `X Viewed`) → el mapping los prefija **otra vez**.
- La página de NPS llama `page()` **sin nombre** → el mapping produce el evento vacío `Viewed`.
- Mixpanel es solo el receptor: no renombra nada. La evidencia es la propiedad `event_original_name` que el propio pipeline adjunta.

Los ~23 eventos `Viewed Viewed *` (~700k ocurrencias/30d) y los 5 de sufijo redundante provienen todos de este mismo mecanismo.

## Cómo solventarlo (dos capas, complementarias)

### 1. Guard en Segment (inmediato, sin release de apps)

En el workspace de Segment: **Connections → Destinations → Mixpanel (Actions) → Functions/Mappings**. Dos opciones, de más a menos robusta:

**a) Destination Insert Function** (recomendada — normaliza para todas las sources de una vez):

```js
// Insert Function del destino Mixpanel (Actions)
function cleanScreenName(name, fallback) {
  let n = (name || "").replace(/\s+/g, " ").trim();
  n = n.replace(/^viewed\s+/i, "");      // quita prefijo ya incluido
  n = n.replace(/\s+viewed$/i, "");      // quita sufijo redundante
  return n || fallback || "Unknown";
}

async function onPage(event) {
  event.name = cleanScreenName(event.name, event.context?.page?.path);
  return event;
}

async function onScreen(event) {
  event.name = cleanScreenName(event.name);
  return event;
}
```

**b) Ajustar el mapping**: si no quieren una function, en el mapping de Page/Screen del destino se puede cambiar el template del Event Name, pero Handlebars no permite condicionales de este tipo — por eso la Insert Function es el camino.

> Nota: esto corrige el flujo **hacia adelante**. Los nombres viejos siguen en el histórico de Mixpanel; para reportes se pueden fusionar en Lexicon (merge de eventos, reversible, no toca datos) cuando decidamos hacerlo.

### 2. Fix de raíz en la app web (`analytics.js`)

El equipo web debe estandarizar sus llamadas para que el `name` sea el **nombre de pantalla limpio**, igual que Android:

```js
// ❌ hoy
analytics.page("Viewed Loan Details");
analytics.page("Bank Account Viewed");
analytics.page();                       // /nps — sin nombre

// ✅ correcto
analytics.page("Loan Details");
analytics.page("Bank Account");
analytics.page("NPS");
```

Con esto, la Insert Function queda como red de seguridad y no como parche permanente.

## Verificación post-fix

Query en Mixpanel (o vía este repo): volumen diario de los eventos `Viewed Viewed *` — debe caer a 0 tras el deploy de la Insert Function. Los eventos correctos (`Viewed Loan Details`, etc.) deben absorber ese volumen sin escalón en la suma total.

## Por qué este patrón aplica a más duplicados

`event_original_name` demuestra que **hay transformaciones de nombre corriendo en el pipeline de Segment**. Es el mismo punto de control donde se pueden atacar otros clusters sin tocar apps: normalizar casing (`Pin Created`→`PIN Created`), colapsar espacios (`Bank Ticket  Details`) y corregir typos conocidos (`Iniciated`→`Initiated`) con un pequeño mapa de alias en la misma function.
