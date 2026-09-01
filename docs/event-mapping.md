# Mapeo de eventos de vana en Mixpanel — Fase 1

**Fecha de corte:** 2026-08-28 · **Ventana de volumen:** últimos 30 días · **Fuente:** Mixpanel (proyecto `Vana` 3370988 y `Vana Pay` 3449788)

---

## 1. Resumen ejecutivo

| Métrica | Valor |
|---|---|
| Eventos distintos en producción (proyecto Vana) | **544** (538 en Lexicon + 6 con tráfico que no aparecen en Lexicon) |
| Volumen total 30 días | **~54.0 M** de ocurrencias |
| Eventos verificados en Lexicon | **1 de 545 (0.2%)** |
| Eventos con descripción | **4 de 545 (0.7%)** |
| Eventos con tags/ownership | **0** |
| Eventos de tipo "pantalla vista" (`Viewed *`) | **177 eventos = 62% del volumen total** (33.4M) |
| Eventos con el bug de doble prefijo `Viewed Viewed *` | **23** (todos activos) |
| Eventos con <100 ocurrencias en 30d (candidatos a muertos) | **77** |
| Proyecto `Vana Pay` independiente (3449788) | **Muerto**: 24 eventos/30d, solo auto-tracking |

**Conclusión:** no hay un problema de *falta* de instrumentación — hay un problema de *gobernanza*. Existen al menos **9 clusters de eventos duplicados activos** (misma acción, distinto nombre, ambos recibiendo tráfico), 3+ convenciones de naming conviviendo, eventos con typos recibiendo millones de ocurrencias, y cero ownership documentado. Cualquier funnel construido hoy sobre estos datos subestima o duplica conversiones dependiendo de qué variante del evento se elija.

---

## 2. Arquitectura actual

```
Mixpanel
├── Vana (3370988)  ← TODO el tráfico vive aquí (~54M/30d)
│   ├── workspace: All Project Data (global)
│   ├── workspace: vana pay
│   ├── workspace: vana presta
│   └── workspace: vana card
├── Vana Pay (3449788)   ← LEGADO. 24 eventos/30d (solo $mp_web_page_view / session_record).
│   └── Quedan 3 eventos custom sin tráfico: "Venta Exitosa", "PIN Created/Authenticated", "Checkout Completed"
├── Vana QA (3371257)    ← ambiente de prueba
└── Vana Pay QA (3591636) ← ambiente de prueba
```

- La separación por producto se hace por **workspaces** dentro del proyecto Vana, no por proyectos. Bien. Pero el nombre del evento no lleva el producto de forma consistente (a veces sí: "Viewed Vana Pay Home", "Vana Pay Onboarding Completed"; a veces no).
- **Acción sugerida:** archivar el proyecto `Vana Pay` (3449788) para eliminar confusión — sus 3 eventos custom están muertos y su naming ("Venta Exitosa") ya es inconsistente con el resto.

---

## 3. Estado de gobernanza en Lexicon

| Señal | Estado |
|---|---|
| Verified | 1/545 — solo un evento está marcado como verificado |
| Descripciones | 4/545 — solo los eventos built-in de Mixpanel tienen descripción |
| Tags (dominio/squad) | 0/545 |
| Contactos/owners | 0/545 |
| Eventos con tráfico **fuera** de Lexicon | 6: `Document Capture Failed`, `Identity Verification Capture Error`, `Identity Verification Completed`, `Identity Verification Module Completed`, `Segment Upgrade Animation Displayed`, `Send Document Failed` |
| Eventos en Lexicon **sin** tráfico 30d | 8: `Comunicaciones Customer IO`, `Detractors`, `Promoters`, `Passive`, `Emai&Push&Inapp_Interaction`, `test`, `$session_start`, `$session_end` |

Nota: `Detractors` / `Promoters` / `Passive` son **segmentos de NPS creados como eventos** — señal clara de instrumentación ad-hoc. `test` en producción habla por sí solo.

---

## 4. Dominios funcionales

Clasificación de los 544 eventos activos del proyecto Vana por dominio (agrupación propia, pendiente de validar con cada squad):

| Dominio | Squad natural | Eventos (aprox.) | Ejemplos de alto volumen |
|---|---|---|---|
| Navegación / vistas de pantalla | Lifecycle | ~177 | `Viewed Home` (8.5M), `Viewed PIN Authentication` (3.5M), `Viewed User Profile` (1.2M) |
| Onboarding / registro / KYC / identidad | Identity | ~110 | `SignUp Attempt` (399k), `Phone Verified` (615k), `Liveness Verification Started` (639k), `ID Submitted` (315k) |
| Autenticación (login/logout/PIN/biometría) | Identity | ~45 | `Login Attempt` (541k), `PIN Authenticated` (266k), `Viewed PIN Authentication` (3.5M) |
| Préstamos (vana presta) | — | ~55 | `Start Loan Application Button Pressed` (836k), `Loan Application Approved` (406k), `Loan Released` (298k) |
| Pagos / checkout / payment sessions | — | ~70 | `Payment Approved` (299k), `Payment Created` (346k), `Viewed Payment Methods Selection` (749k) |
| Tarjeta (vana card) | — | ~20 | `Viewed Vana Card Landing` (572), mayoría de bajo volumen — producto joven |
| Wallet / VRM / vana cash | Growth | ~30 | `Viewed Wallet Onboarding` (229k), `Wallet Cash-In` (39k) |
| Rewards / referidos / retos | Growth | ~25 | `Viewed Rewards Home` (860k), `Rewards Challenge Information` (345k) |
| Mensajería (Email/Push/In-app/Webhook, CRM) | Lifecycle | ~35 | `In_app Sent` (2.0M), `Email Opened` (1.2M), `Push Dropped` (488k) |
| Backend / riesgo / originación | (data/riesgo) | ~28 | `Backend Payment Rejected` (100k), `Risk Analysis Score` (141k) |
| Cuenta / perfil / merge / delete | Identity | ~35 | `Notification Inbox Viewed` (698k), `Account Deleted` (7.7k) |
| Delivery / logística | — | ~10 | `Delivery Assigned` (342) |
| Ruido / sin clasificar | — | ~15 | `voicemail` (882k!), `connected`/`not_connected` (753k), `Viewed` (3.9k), `test` |

> `voicemail`, `connected`, `not_connected` parecen provenir de un dialer/contact-center bombeando a Mixpanel. Son el **tercer "evento" de mayor volumen** y contaminan cualquier métrica de "All Events". Prioridad alta: moverlos a un proyecto propio o taggearlos y ocultarlos.

---

## 5. Clusters de duplicados activos (misma acción, distinto nombre)

Estos son los casos donde **dos o más eventos con tráfico simultáneo** representan la misma acción. Volúmenes = últimos 30 días.

### 5.1 Login
| Evento | Vol 30d | Nota |
|---|---|---|
| `Login Attempt` | 541,259 | dominante |
| `Login Button Pressed` | 260,209 | duplicado (UI) |
| `Login Clicked` | 10,785 | duplicado (¿web?) |
| `Login Attempted` | 2,376 | duplicado (tiempo verbal) |
| `Login Succeeded` / `Login Failed` | 2,727 / 1,018 | resultado — volumen no cuadra con 541k intentos: instrumentación parcial |
| **Canónico propuesto** | | `Login Started`, `Login Completed`, `Login Failed` + prop `method` |

### 5.2 Logout
| Evento | Vol 30d |
|---|---|
| `Logged Out` | 64,993 |
| `Logout Button Pressed` | 56,068 |
| `Logout Attempted` / `Logout Succeeded` / `Logout Failed` | 467 / 439 / 23 |
| **Canónico propuesto** | `Logout Completed` (+ `Logout Started` si se necesita el intento) |

### 5.3 Registro / creación de cuenta
| Evento | Vol 30d | Nota |
|---|---|---|
| `SignUp Attempt` | 399,333 | |
| `Create Account Button Pressed` | 246,672 | duplicado UI |
| `Registration Completed` | 129,314 | outcome |
| `Registration Clicked` | 3,468 | duplicado |
| `Backend User Created` | 5,078 | servidor — volumen no cuadra con 129k |
| **Canónico propuesto** | | `Signup Started`, `Signup Completed` + prop `source` (client/backend se distingue por prop, no por nombre) |

### 5.4 PIN — duplicado por mayúsculas
| Evento | Vol 30d | Nota |
|---|---|---|
| `PIN Created` | 185,041 | dominante |
| `Pin Created` | 982 | **mismo evento, distinto casing** — Mixpanel es case-sensitive |
| `Viewed PIN Creation` vs `Viewed Pin Creation` | 231,644 vs 401 | ídem |
| `Viewed PIN Confirmation` vs `Viewed Pin Confirmation` | 185,239 vs 616 | ídem |
| `PIN Created/Authenticated` | 0 (proyecto legado) | mezcla dos acciones en un nombre |

### 5.5 Merge de cuentas — duplicado por typo
| Evento | Vol 30d | Nota |
|---|---|---|
| `Merge Account Initiated` | 9,438 | correcto |
| `Merge Account Iniciated` | 4,701 | **typo activo** — un tercio del tráfico se va al evento mal escrito |

### 5.6 Help Center — duplicado por espacio
| Evento | Vol 30d |
|---|---|
| `Viewed HelpCenter` | 371,689 |
| `Viewed Help Center` | 41,310 |

### 5.7 Información bancaria — 4 nombres para la misma zona
| Evento | Vol 30d |
|---|---|
| `Viewed Bank Account Details` | 420,221 |
| `Viewed Bank Account Confirmation` | 319,986 |
| `Viewed Bank Account Info` | 233,500 |
| `Viewed Bank Account Information` | 174,718 |
| `Viewed Banking Info` | 26 |
| `Viewed Bank Account Viewed` | 504 |

`Info` vs `Information` casi con seguridad son la misma pantalla instrumentada dos veces (o por dos equipos/plataformas).

### 5.8 Verificación de identidad — 4 instrumentaciones paralelas del mismo funnel
| Familia | Eventos | Vol 30d (suma aprox.) | Nota |
|---|---|---|---|
| `ID *` | `ID Submitted`, `ID Verified`, `ID Verification Failed/Completed/Method/...` | ~1.0M | dominante hoy |
| `Liveness *` | `Liveness Verification Started/Success/Failed/Created`, `Liveness Capture Completed` | ~970k | prueba de vida |
| `Incode *` | `Incode Flow/Module Started/Completed/Failed/Retry...` (11 eventos) | ~15k | **nombre del vendor en el evento** |
| `Identity Verification *` | `Identity Verification Started/Completed/Failed/Aborted/Flow Ready/Module...` (10 eventos) | ~1.2k | instrumentación nueva en rollout (6 de estos ni están en Lexicon) |
| `Verification.Provider.Finished` | | 717 | notación con puntos, única en todo el proyecto |
| **Canónico propuesto** | | | Una sola familia `Identity Verification *` + props `provider` (incode/…), `step` (id_front/id_back/selfie/liveness), `result` |

### 5.9 Checkout
| Evento | Vol 30d | Nota |
|---|---|---|
| `Checkout Attempted` | 8,359 | cliente |
| `Checkout Successful` | 3,177 | cliente |
| `Backend Checkout Completed` | 6,190 | servidor — ¿por qué 2x el del cliente? |
| `Viewed Checkout Completed` | 4,513 | pantalla |
| `Checkout Completed` | 0 (proyecto legado) | |
| **Canónico propuesto** | | `Checkout Started`, `Checkout Completed`, `Checkout Failed` + prop `source` |

### 5.10 Inbox de notificaciones — doble instrumentación confirmada
La misma pantalla se instrumenta **dos veces desde la app web**, con dos nombres distintos (confirmado vía `event_original_name` + `mp_lib`):

| Llamada | Evento resultante | Vol 30d | SDK |
|---|---|---|---|
| `analytics.page("Notification Inbox")` | `Viewed Notification Inbox` | 617,809 + 90,625 | Android + web |
| `analytics.track("Notification Inbox Viewed")` | `Notification Inbox Viewed` | 775,022 | **solo web** |

~1.48 M eventos/mes para una sola vista de pantalla, contados dos veces. Android solo emite el correcto. **Acción:** la web elimina el `track()` redundante y se queda con `page()`. A diferencia del bug `Viewed Viewed`, este no es un problema del pipeline de Segment — se arregla en el código de la web.

### 5.11 Otros duplicados menores
- `Loan 1 Application Approved/Rejected/Fulfilled/Released` vs `Loan Application Approved/...` — el número de préstamo va en el nombre en lugar de una propiedad `loan_sequence`.
- `Viewed Onboarding` (1.1M) vs `Viewed Onboarding 1/2/3/4` (374/14/7/5) — el paso va en el nombre; usar prop `step`.
- `Viewed Personal Information 1/2/3` y `Viewed Work Information 1/2/3` — ídem (y sus variantes `Viewed Viewed *`).

---

## 6. Problemas sistémicos de naming

1. **Doble prefijo `Viewed Viewed *` (23 eventos activos, ~700k/30d)** — **causa raíz confirmada** (ver [runbook](fix-viewed-viewed.md)): el mapping page/screen del destino Mixpanel (Actions) en Segment antepone `Viewed `, y la app web (`analytics.js`) le pasa nombres ya prefijados; Android (`analytics-kotlin`) los pasa limpios. Ej.: `Viewed Viewed Loan Details` (70,614, web) convive con `Viewed Loan Details` (435,805, Android). Fix: Insert Function en el destino de Segment + estandarizar los `page()` de la web.
2. **Sufijo redundante**: `Viewed Bank Account Viewed`, `Viewed Personal Info Viewed`, `Viewed VRM Viewed`, `Viewed Review Viewed`, y un evento llamado solo `Viewed` (3,882/30d).
3. **Tres convenciones de acción UI conviviendo**: `* Button Pressed` (37 eventos), `* Clicked` (20), `* Tapped` (1). La misma semántica, tres vocabularios — imposible hacer queries genéricas.
4. **Mezcla de idiomas**: `Viewed Detalle de orden`, `Viewed Orden generada`, `Viewed Formulario agregar vendedor`, `Viewed Historial`, `Viewed Tu negocio`, `Venta Exitosa`, `Comunicaciones Customer IO` conviven con inglés.
5. **Mezcla de casing/estilo**: Title Case dominante, pero existen `Payment calculator opened` (sentence case, 7 eventos), `In_app Opened` (snake), `created payment promise` / `expired payment promise` (minúsculas), `connected`, `voicemail`, `Verification.Provider.Finished` (dot notation).
6. **Typos activos**: `Merge Account Iniciated` (4.7k/30d), `Viewed Credit Aplication Confirmation` (4.6k), `Viewed Viewed Wallet Onborading`, `Viewed Bank Ticket  Details` (doble espacio, **358,851**/30d — el typo con más tráfico del proyecto).
7. **Metadata en el nombre en lugar de propiedades**: número de paso (`Onboarding 1/2/3`), secuencia de préstamo (`Loan 1 *`), vendor (`Incode *`), origen (`Backend *` — 28 eventos), producto (`Vana Pay *` a veces sí, a veces no).

---

## 7. Taxonomía canónica propuesta (v0 — para discusión)

Basada en el estándar *Object + Action (past tense)* que recomienda Segment, que es hacia donde queremos centralizar:

### Reglas
1. **Nombre = `Objeto Acción` en inglés, Title Case, acción en pasado.** Ej.: `Loan Application Submitted`, `Payment Completed`, `Signup Started`.
2. **Un solo evento de vistas: `Screen Viewed`** con propiedad `screen_name` (y `flow`, `step`). Esto elimina ~177 eventos (62% del volumen) de un plumazo y hace trivial el análisis de navegación. Los nombres de pantalla se estandarizan en un enum versionado.
3. **Un solo evento de interacción genérica: `Element Tapped`** (o `Button Tapped`) con `element_name`, para todo lo que hoy es `* Button Pressed`/`* Clicked` y no constituye un hito de negocio. Los hitos de negocio (submit de solicitud, aceptación de oferta) conservan evento propio.
4. **Resultado como sufijo estándar**: `Started` / `Submitted` / `Completed` / `Failed`. Nunca `Success`, `Successful`, `Succeeded`, `Approved` para lo mismo.
5. **Todo lo demás va en propiedades comunes obligatorias**:
   - `product`: `vana` | `vana_pay` | `vana_presta` | `vana_card`
   - `source`: `client_ios` | `client_android` | `client_web` | `backend`  (elimina el prefijo `Backend *`)
   - `flow` / `step` (elimina los `1/2/3` en nombres)
   - `provider` cuando aplique (elimina `Incode *`)
6. **Español prohibido en nombres de eventos**; el español vive en `screen_name` si el negocio lo requiere.
7. **Eventos de mensajería** (`Email/Push/In_app/Webhook *`) se consolidan como `Message Sent/Delivered/Opened/Clicked/Bounced/Failed` + prop `channel`.
8. **Todo evento nuevo nace en el tracking plan** (Segment cuando esté disponible; mientras tanto, Lexicon de Mixpanel con verified + descripción + tag de squad + owner).

### Ejemplo de migración (cluster login)
| Hoy | Canónico |
|---|---|
| `Login Attempt`, `Login Button Pressed`, `Login Clicked`, `Login Attempted` | `Login Started` |
| `Login Succeeded` | `Login Completed` |
| `Login Failed` | `Login Failed` |

---

## 8. Plan de acción propuesto

| Fase | Acción | Impacto |
|---|---|---|
| **1. Higiene inmediata (Lexicon, sin tocar código)** | Ocultar (`hidden`) los 77 eventos <100/30d que se confirmen muertos, `test`, `Detractors/Promoters/Passive`, y el ruido del dialer (`voicemail`, `connected`, `not_connected`). Taggear los 544 eventos por dominio/squad. Marcar `verified` los ~50 eventos núcleo de KPIs. | Lexicon usable en días |
| **2. Corregir bugs de instrumentación** | Wrapper que genera `Viewed Viewed *` (23 eventos); typos `Merge Account Iniciated`, `Bank Ticket  Details`, `Aplication`; casing `Pin Created`. | Detiene la fragmentación activa |
| **3. Definir tracking plan canónico** | Validar §7 con los tres squads; escribir el plan (este repo) evento por evento con props obligatorias. | Fuente única de verdad |
| **4. Migración dual-write** | Instrumentar nombres canónicos en paralelo, mantener alias en dashboards (Lexicon permite display names/merge), deprecar variantes viejas tras 1–2 ciclos de release. | Sin romper dashboards |
| **5. Centralizar en Segment** | Con acceso a la Public API: cargar el tracking plan a Segment Protocols, bloquear eventos fuera de plan, y atribuir cada source a su producto. | Gobernanza permanente |

---

## 9. Desglose por producto (`product_context`)

La dimensión de producto vive en la propiedad de evento **`product_context`**. Datos completos en [`data/mixpanel_event_by_product.csv`](../data/mixpanel_event_by_product.csv); visual en el [dashboard](dashboard/index.html).

| Valor | Volumen 30d | Eventos distintos | Nota |
|---|---|---|---|
| `presta` | 40,446,453 (75%) | 225 | La app principal: onboarding, auth, wallet, rewards y préstamos viven aquí |
| *(sin valor)* | 11,998,615 (22%) | 128 | CRM (Email/Push/In-app), ciclo de préstamo backend, dialer (`voicemail` etc.), deliveries |
| `pay` | 1,199,093 | 231 | vana pay: checkout, órdenes, merchants, calculadora de pagos |
| `vana_presta` | 397,018 | 7 | **Valor duplicado de `presta`** — solo lo emite el servicio KYC backend (ID Submitted, Document Verification…) |
| `card` | 18,042 | 49 | vana card (producto joven); aquí corre el rollout de la familia nueva `Identity Verification *` |

Hallazgos:
- **62 eventos aparecen en más de un producto** (auth, KYC, perfil): el mismo flujo se instrumentó por producto en lugar de compartir librería común.
- Existe una propiedad rival casi muerta, **`context_product`** (`Credit` 39k / `Pay` 3.6k) — otro caso de duplicación, ahora a nivel propiedad.
- La plataforma vive en **`source_app`** (`vana_android_app` 35.8M, `vana_web_app` 5.7M, `vana_pay_vrm`, `lms`, `casas_de_cobranza`, `internal_tools_automated_process`), con 12.4M sin valor.
- El bug `Viewed Viewed *` afecta tanto a `presta` como a `card` — el wrapper defectuoso es compartido.
- Estandarizar `product_context` (unificar `vana_presta`→`presta`, exigirlo en eventos backend/CRM) es prerequisito del tracking plan canónico.

## 10. Pendientes / limitaciones de esta fase

- **Sin acceso a Segment aún**: falta atribuir cada evento a su *source* de Segment y cargar el tracking plan a Protocols.
- La asignación de dominio (§4) es heurística y debe validarse con cada squad.
- Falta inventariar propiedades (siguiente fase: `List-Properties` por evento núcleo para detectar props duplicadas tipo `email_verified` vs `emailVerified` — ya hay dos confirmadas: `product_context` vs `context_product`).
