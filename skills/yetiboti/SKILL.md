---
name: yetiboti
description: Asistente de datos de Promise (MercadoLibre). Responde consultas sobre alarmas, promesas/CVR, shipments/Lead Time, composición (Buffering, Custom Offsets), Promesa Ideal y Waterfall de VIP consultando BigQuery en meli-bi-data.
argument-hint: "<pregunta en lenguaje natural sobre alarmas, promesas, shipments, LT, buffering, CVR, promesa ideal o waterfall de VIP>"
allowed-tools: Bash, Write, PowerShell
---

version: 1.4
update-url: https://raw.githubusercontent.com/gonzaloansaldo/chronos-skills/main/version.json
skill-url: https://raw.githubusercontent.com/gonzaloansaldo/chronos-skills/main/skills/yetiboti/SKILL.md

Consulta del usuario: $ARGUMENTS

---

## Instrucciones de ejecución

### Paso 1 — Verificar entorno con PowerShell

Ejecutar:
```powershell
$bqOk = Get-Command bq -ErrorAction SilentlyContinue
$account = (gcloud auth list --filter=status:ACTIVE --format="value(account)" 2>$null | Select-Object -First 1)
"BQ_FOUND:$($bqOk -ne $null) ACCOUNT:$account"
```

- Si `BQ_FOUND:False` → responder y **detener**: `❌ bq CLI no encontrado. Instalá Google Cloud SDK.`
- Si `ACCOUNT:` vacío → responder y **detener**: `❌ No hay cuenta gcloud activa. Ejecutá: gcloud auth login`
- Si todo OK → continuar al Paso 2. El proyecto siempre es `meli-bi-data`.

---

### Paso 2 — Clasificar la consulta

Leer `$ARGUMENTS` y determinar el **tipo de consulta** según las palabras clave:

| Palabras clave | Tipo | Tabla principal |
|---|---|---|
| alarma, alarm, alerta, monitor, cerberus | ALARMS | `BT_CRB_ALARMS` |
| promesa, promise, VIP, visita, CVR, conversión, conversión, conversion | PROMISE_CVR | `BT_SPEED_PROMISE_VIP_CVR` |
| shipment, envío, envio, LT, lead time, delay, early, on time, ontime, buffering, buffer, custom offset, CO, ventana, window, composición, composicion | LT_SUMMARY | `BT_SPEED_PROMISE_LT_SUMMARY` |
| promesa ideal, ideal promise, SD ideal, ND ideal, 2D ideal, NBC, neto de buyers choice | IDEAL_PROMISE | `BT_SHP_IDEAL_PROMISE` |
| waterfall, water fall, descomposición, descomposicion, palancas, palanca, deep dive | WATERFALL | `BT_PROMISE_WATERFALL` |

Si la consulta mezcla promesas con shipments, usar **ambas tablas** y presentar los resultados side-by-side con labels explícitos.

Si no queda claro el tipo, preguntar al usuario antes de continuar.

---

### Mapeo de Picking Types — aplicar en todos los dominios

Cuando el usuario mencione cualquiera de los siguientes nombres o aliases, mapear al valor canónico usado en las tablas:

| Valor en tabla | Aliases que puede usar el usuario |
|---|---|
| `FULFILLMENT` | FF, FBM, FULL, Fulfillment, Full, Fulfillment by Meli |
| `CROSS_DOCKING` | XD, CROSSDOCK, Cross Docking, Cross-Docking |
| `XD_DROP_OFF` | XD_DO, XD Drop Off — a veces agrupado con CROSS_DOCKING |
| `DROP_OFF` | DS, Drop Shipping, Drop Off, Drop-Off, CC, Commercial Carrier |
| `SELF_SERVICE` | FLEX, Self Service, Self-Service |

**Notas de agrupación:**
- Si el usuario pide "XD" sin aclarar si incluye XD_DO → preguntar si quiere solo Cross Docking o también XD Drop Off, o agrupar ambos
- En `BT_SPEED_PROMISE_LT_SUMMARY` los picking types se filtran por `SHP_PICKING_TYPE`
- En `BT_SPEED_PROMISE_VIP_CVR` y Tabla de BI se filtran por `PICKING_TYPE`
- En la Tabla de BI el filtro base ya incluye `PICKING_TYPE IN ('FULFILLMENT','CROSS_DOCKING','XD_DROP_OFF','DROP_OFF','SELF_SERVICE')` — no agregar valores fuera de esta lista

---

### Paso 3 — Construir el SQL

Usar los schemas y reglas del tipo correspondiente para construir el SQL. Siempre usar **Standard SQL** (no legacy). Escribir el SQL en `/tmp/yetiboti_query.sql` con el Write tool.

---

#### TIPO: ALARMS — `meli-bi-data.WHOWNER.BT_CRB_ALARMS`

**Schema:**
| Columna | Tipo | Descripción |
|---|---|---|
| SEVERITY | STRING | Severidad de la alarma |
| PROCESS_ID | STRING | ID del proceso asociado |
| COST | NUMERIC | Costo de la alarma |
| ALARM_DATE | DATETIME | Fecha y hora de la alarma |
| DESCRIPTION | STRING | Descripción de la alarma |
| TITLE | STRING | Título de la alarma |
| SITE | STRING | Site de la alarma (MLA, MLB, MLM, etc.) |
| ENVIRONMENT | STRING | Entorno (prod, staging, etc.) |
| ALARM_IDENTIFIER | STRING | Identificador único |
| PROCESS_NAME | STRING | Nombre del proceso |
| AUD_UPD_DTTM | DATETIME | Fecha/hora de última actualización |
| ADDITIONAL_FIELDS | JSON | Campos adicionales en JSON (metric_dimension, date, mail_range_flag, etc.) |
| ALARM_DATE_DT | DATE | Fecha de la alarma (solo fecha) |
| CREATED_AT | DATETIME | Fecha de creación GMT-4 |
| UPDATED_AT | DATETIME | Fecha de actualización GMT-4 |
| STATE | STRING | Estado de la alarma |
| ALARM_TYPE | STRING | Agrupación por tipo de alarma |
| UNIQ_KEY | STRING | Hash único de la alarma |

**Filtros base obligatorios:**
```sql
ALARM_DATE >= '2025-03-15'
AND PROCESS_NAME IN (
  'Promise Offer Monitor - Receiver State',
  'Promise Offer Monitor - Logistic Type',
  'Promise Offer Monitor - Site',
  'Promise Offer Monitor - Last Mile Facility'
)
AND ENVIRONMENT = 'prod'
AND JSON_VALUE(ADDITIONAL_FIELDS.mail_range_flag) = 'MAIL_HOUR_RANGE'
```

**QUALIFY obligatorio** (para quedarse con el registro más reciente por dimensión):
```sql
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY PROCESS_NAME, JSON_VALUE(ADDITIONAL_FIELDS.metric_dimension),
               JSON_VALUE(ADDITIONAL_FIELDS.date), SITE
  ORDER BY AUD_UPD_DTTM DESC
) = 1
```

**Mapeo de dimensiones** (filtrar `JSON_VALUE(ADDITIONAL_FIELDS.metric_dimension)`):
- Si el usuario menciona **site, total, general** → dimensión: `site` o menor
- Si menciona **estado destino, estado, receiver state** → dimensión: `receiver_state`
- Si menciona **logistic type, picking type, PT, tipo logístico** → dimensión: `logistic_type`
- Si menciona **SVC, service center, last mile facility, facility** → dimensión: `last_mile_facility`

**Columnas útiles del JSON:**
- `JSON_VALUE(ADDITIONAL_FIELDS.metric_dimension)` — dimensión de la alarma
- `JSON_VALUE(ADDITIONAL_FIELDS.date)` — fecha de la métrica monitoreada
- `JSON_VALUE(ADDITIONAL_FIELDS.metric_value)` — valor actual de la métrica
- `JSON_VALUE(ADDITIONAL_FIELDS.threshold)` — umbral configurado

**Nota para las respuestas de alarmas:** un aumento en alarmas puede relacionarse con un aumento en el porcentaje de Buffering o Custom Offsets.

---

#### TIPO: PROMISE_CVR — `meli-bi-data.WHOWNER.BT_SPEED_PROMISE_VIP_CVR`

**Schema:**
| Columna | Tipo | Descripción |
|---|---|---|
| VIP_DS | DATE | Fecha de la visita |
| SITE_ID | STRING | Site del buyer (MLA, MLB, MLM, etc.) |
| PICKING_TYPE | STRING | Picking type (XD, FBM, etc.) |
| FLAG_LAST_VIP | BOOLEAN | True si es la última VIP del día del usuario |
| TRK_DESTINATION_STATE_NAME | STRING | Estado destino del envío |
| TRK_DESTINATION_CITY_NAME | STRING | Ciudad destino del envío |
| AVG_PROMISE | FLOAT | Promesa promedio en días |
| AVG_PROMISE_LW | FLOAT | Promesa promedio (ventana con UB > 2 días) |
| VISITS | INTEGER | Cantidad de visitas |
| VIP_CONGRATS | INTEGER | Cantidad de órdenes generadas |
| SAMEDAYS_VIPS | INTEGER | Visitas con promesa Same Day |
| ONEDAYS_VIPS | INTEGER | Visitas con promesa Next Day |
| TWODAYS_VIPS | INTEGER | Visitas con promesa 2 días |
| THREEDAYS_VIPS | INTEGER | Visitas con promesa 3 días |
| FOURDAYS_VIPS | INTEGER | Visitas con promesa 4 días |
| FIVEDAYS_VIPS | INTEGER | Visitas con promesa 5 días |
| MASFIVEDAYS_VIPS | INTEGER | Visitas con promesa más de 5 días |
| WINDOW_VIPS | INTEGER | Visitas con promesa con ventana |
| FIXED_VIPS | INTEGER | Visitas con promesa fija |
| SW_VIPS | INTEGER | Visitas Short Window (UB ≤ 3 días) |
| LW_VIPS | INTEGER | Visitas Long Window (UB > 3 días) |
| AGENCY_VIPS | INTEGER | Visitas con destino agencia |
| CO_FLAG_VIPS | INTEGER | Visitas con custom offset |
| SAMEDAYS_VIP_CONGRATS | INTEGER | Órdenes SD |
| ONEDAYS_VIP_CONGRATS | INTEGER | Órdenes ND |
| TWODAYS_VIP_CONGRATS | INTEGER | Órdenes 2D |
| THREEDAYS_VIP_CONGRATS | INTEGER | Órdenes 3D |
| WINDOW_VIP_CONGRATS | INTEGER | Órdenes con ventana |
| FIXED_VIP_CONGRATS | INTEGER | Órdenes con promesa fija |
| VIP_MDD | INTEGER | Visitas Meli Delivery Day |
| VIP_SLOW_FLEXIBLE | INTEGER | Visitas Slow Flexible |
| VIP_TURBO | INTEGER | Visitas Turbo |
| VIP_FDS | INTEGER | Visitas fin de semana |
| FREE_SHP_VIPS | INTEGER | Visitas con Free Shipping |
| NON_FREE_SHP_VIPS | INTEGER | Visitas sin Free Shipping |
| PROMISE_UB | INTEGER | Upper Bound de la promesa |
| PROMISE_LB | INTEGER | Lower Bound de la promesa |
| VIP_DS_WEEK | INTEGER | Número de semana |
| VIP_DS_MONTH | INTEGER | Número de mes |
| VIP_DS_YEAR | INTEGER | Año |
| VIP_DS_ISOYEAR | INTEGER | Año ISO |
| FLAG_HNB | INTEGER | Flag Heavy & Bulky |

**Reglas de cálculo — métricas de distribución de promesa (CRÍTICO):**

Siempre dividir el numerador por `NULLIF(SUM(VISITS), 0)` o `SAFE_DIVIDE`. Construir el numerador según la métrica pedida:

| Métrica | Numerador |
|---|---|
| % SD | `SUM(SAMEDAYS_VIPS)` |
| % ND | `SUM(ONEDAYS_VIPS)` |
| % <=ND | `SUM(SAMEDAYS_VIPS) + SUM(ONEDAYS_VIPS)` |
| % <=2D | `SUM(SAMEDAYS_VIPS) + SUM(ONEDAYS_VIPS) + SUM(TWODAYS_VIPS) + SUM(WINDOW_VIPS_UB_MENOR_2)` |
| % 2D | `SUM(TWODAYS_VIPS)` |
| % 3D | `SUM(THREEDAYS_VIPS)` |
| % 4D | `SUM(FOURDAYS_VIPS)` |
| % 5D | `SUM(FIVEDAYS_VIPS)` |
| % >5D | `SUM(MASFIVEDAYS_VIPS)` |
| CVR / Conversión | `SUM(VIP_CONGRATS) / SUM(VISITS)` — agregar filtro `FLAG_LAST_VIP = TRUE` |
| Promesa promedio | `SUM(AVG_PROMISE * VISITS) / SUM(VISITS)` (ponderado) |

**Lógica de fechas — semana Domingo a Sábado (OBLIGATORIO):**
- Siempre filtrar usando `VIP_DS` con `BETWEEN`. La semana va de Domingo a Sábado.
- **Semana pasada:** `VIP_DS BETWEEN DATE_SUB(DATE_TRUNC(CURRENT_DATE(), WEEK), INTERVAL 1 WEEK) AND DATE_SUB(DATE_TRUNC(CURRENT_DATE(), WEEK), INTERVAL 1 DAY)`
- **2 semanas atrás:** reemplazar con `INTERVAL 2 WEEK` y `INTERVAL 8 DAY`
- **NUNCA usar** `VIP_DS_WEEK`, `VIP_DS_ISOYEAR` ni `VIP_DS_YEAR` para filtrar por semana.
- No conocés la fecha actual — siempre delegarla a `CURRENT_DATE()` en BigQuery. Nunca fabricar ni mencionar fechas exactas en la respuesta; explicar la lógica en términos de "última semana completa (Domingo a Sábado)".

**Top Cities — cuando el usuario menciona "top cities", "top ciudades" o "top 10":**

Agregar este LEFT JOIN a la query:
```sql
FROM `meli-bi-data.WHOWNER.BT_SPEED_PROMISE_VIP_CVR` VIP
LEFT JOIN `meli-bi-data.SBOX_NETWORKD.LK_PROMISE_TOP_CITIES_FIXED` TC
  ON VIP.SITE_ID = TC.SITE_ID
 AND VIP.TRK_DESTINATION_STATE_NAME = TC.TRK_DESTINATION_STATE_NAME
 AND VIP.TRK_DESTINATION_CITY_NAME = TC.TRK_DESTINATION_CITY_NAME
```

Reglas de agrupación:
- Usar `CASE WHEN TC.METRO_CITY_GROUP LIKE 'Other Cities%' THEN 'Other Cities' ELSE COALESCE(TC.METRO_CITY_GROUP, 'Other Cities') END AS CITY_GROUP`
- La respuesta debe incluir (en orden): hasta 10 grupos por `METRO_CITY_GROUP`, una fila `Other Cities`, y una fila `Total Site` (con `UNION ALL`)
- Ordenar: ciudades por volumen DESC, `Other Cities` penúltimo, `Total Site` último
- En la tabla de respuesta mostrar SOLO dos columnas: label de ciudad y el porcentaje pedido. No mostrar volúmenes crudos salvo que el usuario los pida explícitamente.

**Ambigüedad Ciudad vs. Estado:**
Si el usuario pregunta por una ubicación que comparte nombre con estado/provincia (ej: São Paulo, Rio de Janeiro, Buenos Aires), mostrar AMBAS vistas con `UNION ALL`:
- Un SELECT filtrando `TRK_DESTINATION_CITY_NAME = 'Nombre'` → label `'Nombre (City)'`
- Un SELECT filtrando `TRK_DESTINATION_STATE_NAME = 'Nombre'` → label `'Nombre (State)'`

Para cualquier otra ciudad sin ambigüedad, filtrar directamente por `TRK_DESTINATION_CITY_NAME`.

**Diferenciación Fast / Slow en VIPs:**

Las métricas genéricas de VIPs (sin aclaración) siempre hacen referencia a la promesa más rápida disponible (Fast). En una misma VIP puede haber hasta dos ofertas de promesa; la oferta Slow corresponde a los métodos `slow` y `slow_meli`.

- Si el usuario **no menciona** slow ni fast → usar las métricas genéricas (comportamiento estándar)
- Si el usuario menciona **"fast"** → aclarar que las métricas estándar ya son Fast, no hay campos adicionales
- Si el usuario menciona **"slow"** → usar los campos específicos de Slow:

| Métrica Slow | Fórmula |
|---|---|
| % Visitas con oferta Slow | `SUM(VISITS_SLOW) / SUM(VISITS)` |
| Promesa Slow LB promedio (días) | `SUM(PROMISE_SLOW_LB) / SUM(VISITS_SLOW)` |
| Promesa Slow UB promedio (días) | `SUM(PROMISE_SLOW_UB) / SUM(VISITS_SLOW)` |

Nota: no existe CVR específico de Slow en el schema. Si el usuario lo pide, aclararlo explícitamente.

**IMPORTANTE:** Para consultas de promesas, usar SIEMPRE esta tabla como fuente autoritativa. Nunca dividir un numerador de VIPs por un denominador de SHIPMENTS.

**Regla de filtros:** Bajo ninguna circunstancia agregar filtros que no estén especificados explícitamente en las instrucciones o en el pedido del usuario.

---

#### Tabla de BI — `meli-bi-data.WHOWNER.DM_SHP_TRACKS_VIP_CONVERTION`

Usar esta tabla SOLO cuando la tabla intermedia (`BT_SPEED_PROMISE_VIP_CVR`) no puede responder la consulta. Es una tabla muy pesada — una fila por VIP con toda la granularidad — y queries mal acotadas pueden tardar horas.

**Cuándo ir a la Tabla de BI (triggers):**
- El usuario pide granularidad por **zip code** (solo disponible para MLA)
- El usuario pide análisis de **convivencia de promesas** — comparar las dos ofertas simultáneas (fast vs slow, métodos, bounds crudos)
- El usuario pide desglose por **método de entrega raw** (`PROMISE_METHOD_TYPE_ADDRESS_0/1`)
- El usuario pide campos de **origen granular** por facility o estado de origen por promesa
- El usuario pide datos por **seller o buyer específico**
- La métrica requerida **no existe en la intermedia** (ej: `SPAN_SLOW_FAST_LB`, campos de agency con detalle)

**Advertencia obligatoria antes de ejecutar — siempre mostrar y esperar confirmación:**
> ⚠️ *Esta consulta requiere ir a la Tabla de BI (`DM_SHP_TRACKS_VIP_CONVERTION`), que es muy pesada. Para rangos de más de 7 días puede tardar varios minutos. ¿Querés continuar, o preferís ver esto a nivel de [ciudad / picking type / semana] desde la tabla optimizada?*

Si el usuario elige la alternativa → resolver con `BT_SPEED_PROMISE_VIP_CVR`. Si confirma ir a la Tabla de BI → ejecutar con los filtros obligatorios de abajo.

**Filtros OBLIGATORIOS en toda query a la Tabla de BI:**

Siempre filtrar primero por `VIP_DS` (la tabla está particionada por este campo — sin este filtro la query escanea todo):

```sql
FROM `meli-bi-data.WHOWNER.DM_SHP_TRACKS_VIP_CONVERTION`
WHERE VIP_DS BETWEEN {fecha_inicio} AND {fecha_fin}   -- SIEMPRE primero, es la partición
  AND PICKING_TYPE IN ('FULFILLMENT','CROSS_DOCKING','XD_DROP_OFF','DROP_OFF','SELF_SERVICE')
  AND USER_BUYER_ID IS NOT NULL
-- Agregar en HAVING (después del GROUP BY):
HAVING FLAG_CBT = FALSE
  AND SHIPPING_MODE = 'ME2'
  AND VERTICAL_TRACK = 'CORE'
  AND FLAG_TRACK_VIP_PROMISE = TRUE
  AND BU = 'mercadolibre'
  AND TYPE_VIP <> 'VIP_PROXIMITY'
```

Para consultas de **CVR** agregar además: `AND FLAG_LAST_VIP = TRUE` en el WHERE.

**Regla crítica de agregación — `COUNT_TRACK_VIP`:**
Las filas de la Tabla de BI NO son 1:1 con visitas. Siempre ponderar por `COUNT_TRACK_VIP`:
- **Visitas** = `SUM(COUNT_TRACK_VIP)` (nunca `COUNT(*)`)
- **Cualquier métrica** = `SUM(campo * COUNT_TRACK_VIP)` o `SUM(CASE WHEN ... THEN COUNT_TRACK_VIP ELSE 0 END)`
- **CVR** = `SUM(CASE WHEN ORDER_PATH IS NOT NULL THEN 1 ELSE 0 END) / SUM(COUNT_TRACK_VIP)`

**Campos disponibles en la Tabla de BI que NO están en la intermedia:**

| Campo | Descripción |
|---|---|
| `PROMISE_DESTINATION_ZIPCODE` | Zip code destino (solo MLA) |
| `PROMISE_METHOD_TYPE_ADDRESS_0` | Método de la primera promesa a domicilio |
| `PROMISE_METHOD_TYPE_ADDRESS_1` | Método de la segunda promesa a domicilio (slow/fast alternativa) |
| `PROMISE_LOWER_BOUND_ADDRESS_0/1` | Lower bound crudo de cada promesa |
| `PROMISE_UPPER_BOUND_ADDRESS_0/1` | Upper bound crudo de cada promesa |
| `PROMISE_ADDRESS_METHOD_TYPE_ADJ` | Método de la promesa mostrada al usuario (la elegida) |
| `PROMISE_ORIGINS_ADDRESS_0/1_VALUE_ID` | ID de facility de origen por promesa |
| `PROMISE_ORIGINS_ADDRESS_0/1_STATE_NAME` | Estado de origen por promesa |
| `PROMISE_HANDLING_DAYS_ADDRESS_0` | Handling time de la promesa principal |
| `PROMISE_CUSTOM_OFFSET_ID_ADDRESS_0` | ID del custom offset aplicado |
| `PROMISE_PICKING_TYPE_0/1` | Picking type por promesa |
| `USER_BUYER_ID` | ID del comprador |
| `ORDER_PATH` | No nulo si la VIP convirtió en orden |
| `SPAN_SLOW_FAST_LB` | Diferencia en días entre promesa slow y fast |
| `PROMISE_METHOD_TYPE_AGENCY_0` | Método de promesa a agencia |
| `ITEM_TYPES` | Tipos de ítem (para flag HNB) |
| `COUNT_TRACK_VIP` | Peso de la fila — usar siempre como ponderador |

**Lógica de convivencia de promesas (caso de uso principal de la Tabla de BI):**
- `PROMISE_METHOD_TYPE_ADDRESS_1 IS NOT NULL` → hay dos ofertas de promesa en la VIP
- La promesa mostrada al usuario es `PROMISE_ADDRESS_METHOD_TYPE_ADJ`
- Fast = la que NO es `slow` ni `slow_meli`
- Slow = la que es `slow` o `slow_meli`
- Para identificar cuál es cuál: comparar `PROMISE_METHOD_TYPE_ADDRESS_0` y `PROMISE_METHOD_TYPE_ADDRESS_1` contra `('slow','slow_meli')`

---

#### TIPO: LT_SUMMARY — `meli-bi-data.WHOWNER.BT_SPEED_PROMISE_LT_SUMMARY`

**Schema:**
| Columna | Tipo | Descripción |
|---|---|---|
| SHP_FIRST_VISIT_DELIVERED_DATE_TZ | DATE | Fecha de primera visita entregada |
| SIT_SITE_ID | STRING | Site (MLA, MLB, MLM, etc.) |
| SHP_PICKING_TYPE | STRING | Picking type |
| GEO_SND_STATE_NAME | STRING | Estado origen |
| GEO_RCV_STATE_NAME | STRING | Estado destino |
| SHP_CARRIER_AJUS | STRING | Carrier comercial |
| SHP_LOGISTIC_CENTER | STRING | Centro logístico de origen |
| LG_FACILITY | STRING | Facility |
| FVD_DATE_YEAR | INTEGER | Año |
| FVD_DATE_QUARTER | STRING | Quarter |
| FVD_DATE_MONTH | STRING | Mes |
| FVD_DATE_WEEK | STRING | Semana |
| ISO_FVD_DATE_YEAR | INTEGER | Año ISO |
| ISO_FVD_DATE_WEEK | STRING | Semana ISO |
| ISO_FVD_YEAR_WEEK | STRING | Año-Semana ISO (ej: "2025-W12") |
| FVD_YEAR_MONTH | STRING | Año-Mes |
| SHIPMENTS | INTEGER | Total de envíos |
| SHIPMENTS_FAST | INTEGER | Envíos Fast |
| SHP_LT_DELAY | INTEGER | Envíos con delay |
| SHP_LT_EARLY | INTEGER | Envíos early |
| SHP_LT_ONTIME | INTEGER | Envíos on time |
| SHP_LT_DELAY_FAST | INTEGER | Envíos Fast con delay |
| SHP_LT_EARLY_FAST | INTEGER | Envíos Fast early |
| SHP_LT_ONTIME_FAST | INTEGER | Envíos Fast on time |
| SHP_LT_REAL_SD | INTEGER | Envíos entregados Same Day |
| SHP_LT_REAL_ND | INTEGER | Envíos entregados Next Day |
| SHP_LT_REAL_2D | INTEGER | Envíos entregados en 2 días |
| SHP_LT_REAL_SD_FAST | INTEGER | Envíos Fast SD |
| SHP_LT_REAL_ND_FAST | INTEGER | Envíos Fast ND |
| SHP_LT_REAL_2D_FAST | INTEGER | Envíos Fast 2D |
| SHP_BUFFERING | INTEGER | Envíos con buffering total |
| SHP_BUFFERING_OPERACIONAL | INTEGER | Buffering operacional |
| SHP_BUFFERING_OPERACIONAL_FBM | INTEGER | Buffering operacional FBM |
| SHP_BUFFERING_OPERACIONAL_XD | INTEGER | Buffering operacional XD |
| SHP_BUFFERING_OPERACIONAL_SVC | INTEGER | Buffering operacional SVC |
| SHP_BUFFERING_MIDDLE_MILE | INTEGER | Buffering Middle Mile |
| SHP_BUFFERING_LAST_MILE | INTEGER | Buffering Last Mile |
| SHP_BUFFERING_SELLER | INTEGER | Buffering Seller |
| SHP_BUFFERING_FIRST_MILE | INTEGER | Buffering First Mile |
| SHP_BUFFERING_FAST | INTEGER | Buffering Fast total |
| SHP_BUFFERING_OPERACIONAL_FAST | INTEGER | Buffering operacional Fast |
| SHP_BUFFERING_OPERACIONAL_FBM_FAST | INTEGER | Buffering op. FBM Fast |
| SHP_BUFFERING_OPERACIONAL_XD_FAST | INTEGER | Buffering op. XD Fast |
| SHP_BUFFERING_OPERACIONAL_SVC_FAST | INTEGER | Buffering op. SVC Fast |
| SHP_BUFFERING_MIDDLE_MILE_FAST | INTEGER | Buffering MM Fast |
| SHP_BUFFERING_LAST_MILE_FAST | INTEGER | Buffering LM Fast |
| SHP_BUFFERING_SELLER_FAST | INTEGER | Buffering Seller Fast |
| SHP_BUFFERING_FIRST_MILE_FAST | INTEGER | Buffering FM Fast |
| SHP_CO_ST | INTEGER | Envíos con Custom Offset ST |
| SHP_CO_ST_SHIFT | INTEGER | Envíos con CO ST Shift |
| SHP_CO_ST_EXPAND | INTEGER | Envíos con CO ST Expand |
| SHP_CO_HT | INTEGER | Envíos con Custom Offset HT |
| SHP_ONLY_CO_ST | INTEGER | Solo CO ST (sin buffering) |
| SHP_ONLY_CO_HT | INTEGER | Solo CO HT (sin buffering) |
| SHP_ONLY_BUFFERING | INTEGER | Solo buffering (sin CO) |
| SHP_CO_ST_AND_BUFFERING | INTEGER | CO ST y buffering simultáneo |
| SHP_PROMISE_WINDOW | INTEGER | Envíos con promesa con ventana |
| SHP_PROMISE_WINDOW_FAST | INTEGER | Envíos Fast con promesa con ventana |
| SHP_LT_DELAY_WINDOW | INTEGER | Delay en promesas con ventana |
| SHP_LT_EARLY_WINDOW | INTEGER | Early en promesas con ventana |
| SHP_LT_ON_TIME_WINDOW | INTEGER | On time en promesas con ventana |
| SHP_FBM | INTEGER | Envíos FBM |
| SHP_XD | INTEGER | Envíos XD |
| SHP_DS | INTEGER | Envíos Drop Shipping |
| SHP_FLEX | INTEGER | Envíos Flex |
| SHP_BULKY | INTEGER | Envíos Bulky |
| SHP_GROUPING | INTEGER | Envíos agrupados |
| SHP_MELI_DELIVEY_DAY | INTEGER | Envíos Meli Delivery Day |
| SHP_NO_RUSH_SLOW_FLEXIBLE | INTEGER | Envíos No Rush / Slow Flexible |

**Diferenciación Fast / Slow en Shipments:**

En shipments hay una sola promesa por envío. Fast = métodos que NO son `slow` ni `slow_meli`. Slow = métodos `slow` y `slow_meli`. Las columnas `_FAST` cubren exclusivamente los envíos Fast; Slow se obtiene siempre por diferencia.

- Si el usuario **no menciona** slow ni fast → usar métricas totales (comportamiento estándar)
- Si el usuario menciona **"fast"** → usar columnas `_FAST` con denominador `SUM(SHIPMENTS_FAST)`
- Si el usuario menciona **"slow"** → calcular por diferencia: `(TOTAL - FAST)` con denominador `SUM(SHIPMENTS) - SUM(SHIPMENTS_FAST)`

**Reglas de cálculo — Total (sin aclaración de fast/slow):**
- **% Delay** = `SAFE_DIVIDE(SUM(SHP_LT_DELAY), SUM(SHIPMENTS))`
- **% Early** = `SAFE_DIVIDE(SUM(SHP_LT_EARLY), SUM(SHIPMENTS))`
- **% On Time** = `SAFE_DIVIDE(SUM(SHP_LT_ONTIME), SUM(SHIPMENTS))`
- **% Buffering** = `SAFE_DIVIDE(SUM(SHP_BUFFERING), SUM(SHIPMENTS))` (ídem para cualquier subtipo)
- **% CO ST** = `SAFE_DIVIDE(SUM(SHP_CO_ST), SUM(SHIPMENTS))`
- **% CO ST Shift** = `SAFE_DIVIDE(SUM(SHP_CO_ST_SHIFT), SUM(SHIPMENTS))`

**Reglas de cálculo — Fast:**
- Reemplazar numerador con columna `_FAST` y denominador con `SUM(SHIPMENTS_FAST)`
- Ejemplo: `% Buffering Fast` = `SAFE_DIVIDE(SUM(SHP_BUFFERING_FAST), SUM(SHIPMENTS_FAST))`

**Reglas de cálculo — Slow (siempre por diferencia):**
- Denominador Slow: `SUM(SHIPMENTS) - SUM(SHIPMENTS_FAST)`
- Ejemplo: `% Buffering Slow` = `SAFE_DIVIDE(SUM(SHP_BUFFERING) - SUM(SHP_BUFFERING_FAST), SUM(SHIPMENTS) - SUM(SHIPMENTS_FAST))`
- Aplicar el mismo patrón para cualquier métrica: `(SUM(CAMPO_TOTAL) - SUM(CAMPO_FAST)) / (SUM(SHIPMENTS) - SUM(SHIPMENTS_FAST))`
- Métricas Slow disponibles por diferencia: Delay, Early, On Time, Buffering (total y subtipos), LT Real SD/ND/2D, Promise Window

**Por semana**: usar `ISO_FVD_DATE_WEEK` + `ISO_FVD_DATE_YEAR` (o `ISO_FVD_YEAR_WEEK` como label)
Usar `SAFE_DIVIDE` en todos los cálculos para evitar división por cero.

---

#### TIPO: IDEAL_PROMISE — `meli-bi-data.WHOWNER.BT_SHP_IDEAL_PROMISE`

**Reglas de cálculo:**

| Métrica | Numerador | Denominador |
|---|---|---|
| % SD Ideal | `SUM(LT_EST_SD_IDEAL)` | `SUM(SHIPMENTS)` |
| % ND Ideal | `SUM(LT_EST_ND_IDEAL)` | `SUM(SHIPMENTS)` |
| % 2D Ideal | `SUM(LT_EST_2D_IDEAL)` | `SUM(SHIPMENTS)` |
| % SD Real | `SUM(LT_EST_SD_REAL)` | `SUM(SHIPMENTS)` |
| % ND Real | `SUM(LT_EST_ND_REAL)` | `SUM(SHIPMENTS)` |
| % 2D Real | `SUM(LT_EST_2D_REAL)` | `SUM(SHIPMENTS)` |

**Variante NBC** (neto de Buyers Choice / without Buyers Choice): si el usuario menciona "NBC", "neto de buyers choice" o "sin buyers choice", reemplazar la columna del numerador por su equivalente `_NBC`:
- `LT_EST_SD_IDEAL_NBC`, `LT_EST_ND_IDEAL_NBC`, `LT_EST_2D_IDEAL_NBC`

**Lógica de fechas — por semana:** usar `ISO_CREATED_WEEK` e `ISO_CREATED_YEAR`.

Usar `SAFE_DIVIDE` en todos los cálculos.

---

#### TIPO: WATERFALL — `meli-bi-data.SBOX_NETWORKD.BT_PROMISE_WATERFALL`

**Parámetros obligatorios — preguntar si no se especifican todos:**
- `site_id`: MLA, MLB, MLM, MLC, MCO, MLU, MPE
- `period1`: período más reciente (el que se analiza)
- `period2`: período base (contra el que se compara)
- `mode`: `WEEK` o `MONTH` — inferir del tipo de período que menciona el usuario

**Reglas de resolución de períodos:**

| Lo que dice el usuario | Resolución |
|---|---|
| "semana pasada" | ISO week actual - 1 |
| "semana anterior" | ISO week actual - 2 |
| "semana 20 del 2026" / "W20 2026" | year=2026, week=20 |
| "mes pasado" | month actual - 1 |
| "enero 2026" / "enero del 2026" | year=2026, month=1 |

Si un período es semanal y el otro mensual → preguntar al usuario cuál modo prefiere antes de continuar.

**Resolución de períodos relativos — NUNCA correr una query separada para resolver fechas.** Para períodos relativos ("semana pasada", "mes pasado") usar subqueries de BigQuery dentro del WHERE de la query principal:

- Semana pasada (WEEK): `ISO_YEAR_VIP_DS = EXTRACT(ISOYEAR FROM DATE_SUB(CURRENT_DATE(), INTERVAL 1 WEEK)) AND ISO_WEEK_VIP_DS = EXTRACT(ISOWEEK FROM DATE_SUB(CURRENT_DATE(), INTERVAL 1 WEEK))`
- Semana anterior (WEEK): reemplazar `INTERVAL 1 WEEK` por `INTERVAL 2 WEEK`
- Mes pasado (MONTH): `YEAR_VIP_DS = EXTRACT(YEAR FROM DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 DAY)) AND MONTH_VIP_DS = EXTRACT(MONTH FROM DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 DAY))`
- Mes anterior (MONTH): `YEAR_VIP_DS = EXTRACT(YEAR FROM DATE_SUB(DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 MONTH), MONTH), INTERVAL 1 DAY)) AND MONTH_VIP_DS = EXTRACT(MONTH FROM DATE_SUB(DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 MONTH), MONTH), INTERVAL 1 DAY))`

**Dimensiones del waterfall:**

| Código | Nombre legible |
|---|---|
| DIM01 | Día semana + Hora |
| DIM02 | Estado destino |
| DIM03 | Picking type |
| DIM04 | FC origen |
| DIM05 | Handling time |
| DIM06 | Shipping time |
| DIM07 | Flag offset |
| DIM08 | Efecto calendario/buffering |
| DIM09 | Residuo tasa pura |

**NUNCA mencionar DIM01, DIM02, etc. en la respuesta — siempre usar el nombre legible.**

---

##### Query waterfall — válida para WEEK y MONTH

La misma estructura de CTEs planos sirve para ambos modos. Solo cambia el bloque de filtros en el SELECT interno según el modo.

**Modo WEEK** — usar `{W1_FILTER}` y `{W2_FILTER}` como expresiones booleanas completas, por ejemplo:
- Semana pasada: `(ISO_YEAR_VIP_DS = EXTRACT(ISOYEAR FROM DATE_SUB(CURRENT_DATE(), INTERVAL 1 WEEK)) AND ISO_WEEK_VIP_DS = EXTRACT(ISOWEEK FROM DATE_SUB(CURRENT_DATE(), INTERVAL 1 WEEK)))`
- Semana explícita: `(ISO_YEAR_VIP_DS = 2026 AND ISO_WEEK_VIP_DS = 22)`

**Modo MONTH** — usar `{M1_FILTER}` y `{M2_FILTER}`:
- Mes pasado: `(YEAR_VIP_DS = EXTRACT(YEAR FROM DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 DAY)) AND MONTH_VIP_DS = EXTRACT(MONTH FROM DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 DAY)))`
- Mes explícito: `(YEAR_VIP_DS = 2026 AND MONTH_VIP_DS = 5)`

Para el PERIOD label en modo MONTH usar: `CONCAT(CAST(YEAR_VIP_DS AS STRING),'_',LPAD(CAST(MONTH_VIP_DS AS STRING),2,'0'))`

```sql
WITH tabla1 AS (
  SELECT
    MAX(CASE WHEN IS_P1 THEN CONCAT(CAST(ISO_YEAR_VIP_DS AS STRING),'_',CAST(ISO_WEEK_VIP_DS AS STRING)) ELSE NULL END) AS PERIOD1,
    MAX(CASE WHEN IS_P2 THEN CONCAT(CAST(ISO_YEAR_VIP_DS AS STRING),'_',CAST(ISO_WEEK_VIP_DS AS STRING)) ELSE NULL END) AS PERIOD2,
    SITE_ID, DIM01 AS DIM01_3, DIM01_2, CONCAT(DIM01,DIM01_2) AS DIM01,
    DIM02, DIM03, DIM04, DIM05, DIM06, DIM07, DIM08,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID) AS VIP_VISITS_TOTAL_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2)) AS VIP_VISITS_DIM01_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02) AS VIP_VISITS_DIM02_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03) AS VIP_VISITS_DIM03_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04) AS VIP_VISITS_DIM04_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05) AS VIP_VISITS_DIM05_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06) AS VIP_VISITS_DIM06_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07) AS VIP_VISITS_DIM07_1,
    SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07,DIM08) AS VIP_VISITS_DIM08_1,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID) AS VIP_VISITS_TOTAL_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2)) AS VIP_VISITS_DIM01_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02) AS VIP_VISITS_DIM02_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03) AS VIP_VISITS_DIM03_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04) AS VIP_VISITS_DIM04_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05) AS VIP_VISITS_DIM05_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06) AS VIP_VISITS_DIM06_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07) AS VIP_VISITS_DIM07_2,
    SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07,DIM08) AS VIP_VISITS_DIM08_2,
    SUM(VIP_VISITS_1) AS VIP_VISITS_1,
    SUM(VIP_VISITS_2) AS VIP_VISITS_2,
    SUM(MENOR2_1) AS MENOR2_1,
    SUM(MENOR2_2) AS MENOR2_2
  FROM (
    SELECT *,
      {P1_FILTER} AS IS_P1,
      {P2_FILTER} AS IS_P2,
      CASE WHEN {P1_FILTER} THEN VIP_VISITS ELSE 0 END AS VIP_VISITS_1,
      CASE WHEN {P2_FILTER} THEN VIP_VISITS ELSE 0 END AS VIP_VISITS_2,
      CASE WHEN {P1_FILTER} THEN MENOR2 ELSE 0 END AS MENOR2_1,
      CASE WHEN {P2_FILTER} THEN MENOR2 ELSE 0 END AS MENOR2_2
    FROM `meli-bi-data.SBOX_NETWORKD.BT_PROMISE_WATERFALL`
    WHERE SITE_ID = '{SITE}'
      AND ({P1_FILTER} OR {P2_FILTER})
    GROUP BY ALL
  )
  GROUP BY ALL
),
tabla2 AS (
  SELECT *,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM01_1, VIP_VISITS_TOTAL_1), 0) AS VIP_VISITS_DIM01_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM02_1, VIP_VISITS_DIM01_1), 0) AS VIP_VISITS_DIM02_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM03_1, VIP_VISITS_DIM02_1), 0) AS VIP_VISITS_DIM03_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM04_1, VIP_VISITS_DIM03_1), 0) AS VIP_VISITS_DIM04_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM05_1, VIP_VISITS_DIM04_1), 0) AS VIP_VISITS_DIM05_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM06_1, VIP_VISITS_DIM05_1), 0) AS VIP_VISITS_DIM06_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM07_1, VIP_VISITS_DIM06_1), 0) AS VIP_VISITS_DIM07_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM08_1, VIP_VISITS_DIM07_1), 0) AS VIP_VISITS_DIM08_1_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM01_2, VIP_VISITS_TOTAL_2), 0) AS VIP_VISITS_DIM01_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM02_2, VIP_VISITS_DIM01_2), 0) AS VIP_VISITS_DIM02_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM03_2, VIP_VISITS_DIM02_2), 0) AS VIP_VISITS_DIM03_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM04_2, VIP_VISITS_DIM03_2), 0) AS VIP_VISITS_DIM04_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM05_2, VIP_VISITS_DIM04_2), 0) AS VIP_VISITS_DIM05_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM06_2, VIP_VISITS_DIM05_2), 0) AS VIP_VISITS_DIM06_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM07_2, VIP_VISITS_DIM06_2), 0) AS VIP_VISITS_DIM07_2_SHARE,
    IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM08_2, VIP_VISITS_DIM07_2), 0) AS VIP_VISITS_DIM08_2_SHARE,
    IFNULL(SAFE_DIVIDE(MENOR2_1, VIP_VISITS_1), 0) AS MENOR2_1_PERC,
    IFNULL(SAFE_DIVIDE(MENOR2_2, VIP_VISITS_2), 0) AS MENOR2_2_PERC
  FROM tabla1
),
tabla3 AS (
  SELECT *,
    VIP_VISITS_DIM01_1_SHARE * IF(VIP_VISITS_DIM02_2_SHARE<>0,VIP_VISITS_DIM02_2_SHARE,VIP_VISITS_DIM02_1_SHARE) * IF(VIP_VISITS_DIM03_2_SHARE<>0,VIP_VISITS_DIM03_2_SHARE,VIP_VISITS_DIM03_1_SHARE) * IF(VIP_VISITS_DIM04_2_SHARE<>0,VIP_VISITS_DIM04_2_SHARE,VIP_VISITS_DIM04_1_SHARE) * IF(VIP_VISITS_DIM05_2_SHARE<>0,VIP_VISITS_DIM05_2_SHARE,VIP_VISITS_DIM05_1_SHARE) * IF(VIP_VISITS_DIM06_2_SHARE<>0,VIP_VISITS_DIM06_2_SHARE,VIP_VISITS_DIM06_1_SHARE) * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM01_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST1,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * IF(VIP_VISITS_DIM03_2_SHARE<>0,VIP_VISITS_DIM03_2_SHARE,VIP_VISITS_DIM03_1_SHARE) * IF(VIP_VISITS_DIM04_2_SHARE<>0,VIP_VISITS_DIM04_2_SHARE,VIP_VISITS_DIM04_1_SHARE) * IF(VIP_VISITS_DIM05_2_SHARE<>0,VIP_VISITS_DIM05_2_SHARE,VIP_VISITS_DIM05_1_SHARE) * IF(VIP_VISITS_DIM06_2_SHARE<>0,VIP_VISITS_DIM06_2_SHARE,VIP_VISITS_DIM06_1_SHARE) * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM02_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST2,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * IF(VIP_VISITS_DIM04_2_SHARE<>0,VIP_VISITS_DIM04_2_SHARE,VIP_VISITS_DIM04_1_SHARE) * IF(VIP_VISITS_DIM05_2_SHARE<>0,VIP_VISITS_DIM05_2_SHARE,VIP_VISITS_DIM05_1_SHARE) * IF(VIP_VISITS_DIM06_2_SHARE<>0,VIP_VISITS_DIM06_2_SHARE,VIP_VISITS_DIM06_1_SHARE) * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM03_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST3,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * VIP_VISITS_DIM04_1_SHARE * IF(VIP_VISITS_DIM05_2_SHARE<>0,VIP_VISITS_DIM05_2_SHARE,VIP_VISITS_DIM05_1_SHARE) * IF(VIP_VISITS_DIM06_2_SHARE<>0,VIP_VISITS_DIM06_2_SHARE,VIP_VISITS_DIM06_1_SHARE) * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM04_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST4,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * VIP_VISITS_DIM04_1_SHARE * VIP_VISITS_DIM05_1_SHARE * IF(VIP_VISITS_DIM06_2_SHARE<>0,VIP_VISITS_DIM06_2_SHARE,VIP_VISITS_DIM06_1_SHARE) * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM05_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST5,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * VIP_VISITS_DIM04_1_SHARE * VIP_VISITS_DIM05_1_SHARE * VIP_VISITS_DIM06_1_SHARE * IF(VIP_VISITS_DIM07_2_SHARE<>0,VIP_VISITS_DIM07_2_SHARE,VIP_VISITS_DIM07_1_SHARE) * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM06_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST6,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * VIP_VISITS_DIM04_1_SHARE * VIP_VISITS_DIM05_1_SHARE * VIP_VISITS_DIM06_1_SHARE * VIP_VISITS_DIM07_1_SHARE * IF(VIP_VISITS_DIM08_2_SHARE<>0,VIP_VISITS_DIM08_2_SHARE,VIP_VISITS_DIM08_1_SHARE) * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM07_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST7,
    VIP_VISITS_DIM01_1_SHARE * VIP_VISITS_DIM02_1_SHARE * VIP_VISITS_DIM03_1_SHARE * VIP_VISITS_DIM04_1_SHARE * VIP_VISITS_DIM05_1_SHARE * VIP_VISITS_DIM06_1_SHARE * VIP_VISITS_DIM07_1_SHARE * VIP_VISITS_DIM08_1_SHARE * (CASE WHEN PERIOD2 IS NULL AND VIP_VISITS_DIM08_2_SHARE=0 THEN MENOR2_1_PERC ELSE MENOR2_2_PERC END) AS EST8
  FROM tabla2
)
SELECT
  SITE_ID,
  ROUND(IFNULL(SAFE_DIVIDE(SUM(MENOR2_2), SUM(VIP_VISITS_2)), 0) * 100, 2) AS METRIC2,
  ROUND((SUM(EST1) - IFNULL(SAFE_DIVIDE(SUM(MENOR2_2), SUM(VIP_VISITS_2)), 0)) * 100, 2) AS DIM01_VAR,
  ROUND((SUM(EST2) - SUM(EST1)) * 100, 2) AS DIM02_VAR,
  ROUND((SUM(EST3) - SUM(EST2)) * 100, 2) AS DIM03_VAR,
  ROUND((SUM(EST4) - SUM(EST3)) * 100, 2) AS DIM04_VAR,
  ROUND((SUM(EST5) - SUM(EST4)) * 100, 2) AS DIM05_VAR,
  ROUND((SUM(EST6) - SUM(EST5)) * 100, 2) AS DIM06_VAR,
  ROUND((SUM(EST7) - SUM(EST6)) * 100, 2) AS DIM07_VAR,
  ROUND((SUM(EST8) - SUM(EST7)) * 100, 2) AS DIM08_VAR,
  ROUND((IFNULL(SAFE_DIVIDE(SUM(MENOR2_1), SUM(VIP_VISITS_1)), 0) - SUM(EST8)) * 100, 2) AS DIM09_VAR,
  ROUND(IFNULL(SAFE_DIVIDE(SUM(MENOR2_1), SUM(VIP_VISITS_1)), 0) * 100, 2) AS METRIC1,
  MAX(PERIOD1) AS PERIOD1,
  MAX(PERIOD2) AS PERIOD2
FROM tabla3
GROUP BY 1
```

-- Y el WHERE:
WHERE SITE_ID = '{SITE}'
  AND ((YEAR_VIP_DS = {YEAR1} AND MONTH_VIP_DS = {MONTH1})
    OR (YEAR_VIP_DS = {YEAR2} AND MONTH_VIP_DS = {MONTH2}))

-- Y los MAX de PERIOD en el SELECT final:
MAX(CASE WHEN IS_P1 THEN CONCAT(YEAR_VIP_DS,'_',LPAD(CAST(MONTH_VIP_DS AS STRING),2,'0')) ELSE NULL END) AS PERIOD1,
MAX(CASE WHEN IS_P2 THEN CONCAT(YEAR_VIP_DS,'_',LPAD(CAST(MONTH_VIP_DS AS STRING),2,'0')) ELSE NULL END) AS PERIOD2,
```

⚠️ **Advertencia mensual:** siempre agregar al final de la respuesta:
> *Comparación mensual: los resultados pueden estar influenciados por diferencias en la cantidad de semanas o distribución de días entre meses.*

---

##### Deep Dive — a pedido del usuario para una dimensión específica

Correr solo cuando el usuario pregunta por una palanca específica. Usa los mismos `{SITE}`, `{YEAR1}`, `{WEEK1}/{MONTH1}`, `{YEAR2}`, `{WEEK2}/{MONTH2}` del waterfall anterior — no volver a pedirlos.

Mapear la palanca mencionada por el usuario a la columna correspondiente:

| Palanca mencionada | DIM a usar en GROUP BY |
|---|---|
| Día semana / Hora / día y hora | DIM01 |
| Estado destino / estado / provincia | DIM02 |
| Picking type / logística / tipo de picking | DIM03 |
| FC origen / fulfillment center / centro logístico | DIM04 |
| Handling time / HT / tiempo de manejo | DIM05 |
| Shipping time / tiempo de envío / tránsito | DIM06 |
| Flag offset / offset / ajuste | DIM07 |
| Calendario / buffering / feriado | DIM08 |

**Query Deep Dive** (reemplazar `{DIM_COL}` por la dimensión correcta, ej: `DIM04`):

```sql
WITH tabla3 AS (
WITH tabla2 AS (
WITH tabla1 AS (
SELECT
  MAX(CASE WHEN IS_P1 THEN CONCAT(ISO_YEAR_VIP_DS,'_',ISO_WEEK_VIP_DS) ELSE NULL END) AS PERIOD1,
  MAX(CASE WHEN IS_P2 THEN CONCAT(ISO_YEAR_VIP_DS,'_',ISO_WEEK_VIP_DS) ELSE NULL END) AS PERIOD2,
  SITE_ID, DIM01 AS DIM01_3, DIM01_2, CONCAT(DIM01,DIM01_2) AS DIM01,
  DIM02, DIM03, DIM04, DIM05, DIM06, DIM07, DIM08,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID) AS VIP_VISITS_TOTAL_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2)) AS VIP_VISITS_DIM01_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02) AS VIP_VISITS_DIM02_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03) AS VIP_VISITS_DIM03_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04) AS VIP_VISITS_DIM04_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05) AS VIP_VISITS_DIM05_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06) AS VIP_VISITS_DIM06_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07) AS VIP_VISITS_DIM07_1,
  SUM(SUM(VIP_VISITS_1)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07,DIM08) AS VIP_VISITS_DIM08_1,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID) AS VIP_VISITS_TOTAL_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2)) AS VIP_VISITS_DIM01_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02) AS VIP_VISITS_DIM02_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03) AS VIP_VISITS_DIM03_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04) AS VIP_VISITS_DIM04_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05) AS VIP_VISITS_DIM05_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06) AS VIP_VISITS_DIM06_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07) AS VIP_VISITS_DIM07_2,
  SUM(SUM(VIP_VISITS_2)) OVER(PARTITION BY SITE_ID,CONCAT(DIM01,DIM01_2),DIM02,DIM03,DIM04,DIM05,DIM06,DIM07,DIM08) AS VIP_VISITS_DIM08_2,
  SUM(VIP_VISITS) AS VIP_VISITS,
  SUM(VIP_VISITS_1) AS VIP_VISITS_1,
  SUM(VIP_VISITS_2) AS VIP_VISITS_2,
  SUM(MENOR2_1) AS MENOR2_1,
  SUM(MENOR2_2) AS MENOR2_2
FROM (
  SELECT *,
    (ISO_YEAR_VIP_DS = {YEAR1} AND ISO_WEEK_VIP_DS = {WEEK1}) AS IS_P1,
    (ISO_YEAR_VIP_DS = {YEAR2} AND ISO_WEEK_VIP_DS = {WEEK2}) AS IS_P2,
    CASE WHEN (ISO_YEAR_VIP_DS = {YEAR1} AND ISO_WEEK_VIP_DS = {WEEK1}) THEN VIP_VISITS ELSE 0 END AS VIP_VISITS_1,
    CASE WHEN (ISO_YEAR_VIP_DS = {YEAR2} AND ISO_WEEK_VIP_DS = {WEEK2}) THEN VIP_VISITS ELSE 0 END AS VIP_VISITS_2,
    CASE WHEN (ISO_YEAR_VIP_DS = {YEAR1} AND ISO_WEEK_VIP_DS = {WEEK1}) THEN MENOR2 ELSE 0 END AS MENOR2_1,
    CASE WHEN (ISO_YEAR_VIP_DS = {YEAR2} AND ISO_WEEK_VIP_DS = {WEEK2}) THEN MENOR2 ELSE 0 END AS MENOR2_2
  FROM `meli-bi-data.SBOX_NETWORKD.BT_PROMISE_WATERFALL`
  WHERE SITE_ID = '{SITE}'
    AND ((ISO_YEAR_VIP_DS = {YEAR1} AND ISO_WEEK_VIP_DS = {WEEK1})
      OR (ISO_YEAR_VIP_DS = {YEAR2} AND ISO_WEEK_VIP_DS = {WEEK2}))
  GROUP BY ALL
)
GROUP BY ALL
),

SELECT
  SITE_ID, DIM01, DIM02, DIM03, DIM04, DIM05, DIM06, DIM07, DIM08,
  VIP_VISITS_TOTAL_1, VIP_VISITS_DIM01_1, VIP_VISITS_DIM02_1, VIP_VISITS_DIM03_1,
  VIP_VISITS_DIM04_1, VIP_VISITS_DIM05_1, VIP_VISITS_DIM06_1, VIP_VISITS_DIM07_1, VIP_VISITS_DIM08_1,
  VIP_VISITS_TOTAL_2, VIP_VISITS_DIM01_2, VIP_VISITS_DIM02_2, VIP_VISITS_DIM03_2,
  VIP_VISITS_DIM04_2, VIP_VISITS_DIM05_2, VIP_VISITS_DIM06_2, VIP_VISITS_DIM07_2, VIP_VISITS_DIM08_2,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_TOTAL_1, VIP_VISITS_TOTAL_1), 0) AS VIP_VISITS_TOTAL_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM01_1, VIP_VISITS_TOTAL_1), 0) AS VIP_VISITS_DIM01_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM02_1, VIP_VISITS_DIM01_1), 0) AS VIP_VISITS_DIM02_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM03_1, VIP_VISITS_DIM02_1), 0) AS VIP_VISITS_DIM03_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM04_1, VIP_VISITS_DIM03_1), 0) AS VIP_VISITS_DIM04_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM05_1, VIP_VISITS_DIM04_1), 0) AS VIP_VISITS_DIM05_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM06_1, VIP_VISITS_DIM05_1), 0) AS VIP_VISITS_DIM06_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM07_1, VIP_VISITS_DIM06_1), 0) AS VIP_VISITS_DIM07_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM08_1, VIP_VISITS_DIM07_1), 0) AS VIP_VISITS_DIM08_1_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM01_2, VIP_VISITS_TOTAL_2), 0) AS VIP_VISITS_DIM01_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM02_2, VIP_VISITS_DIM01_2), 0) AS VIP_VISITS_DIM02_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM03_2, VIP_VISITS_DIM02_2), 0) AS VIP_VISITS_DIM03_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM04_2, VIP_VISITS_DIM03_2), 0) AS VIP_VISITS_DIM04_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM05_2, VIP_VISITS_DIM04_2), 0) AS VIP_VISITS_DIM05_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM06_2, VIP_VISITS_DIM05_2), 0) AS VIP_VISITS_DIM06_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM07_2, VIP_VISITS_DIM06_2), 0) AS VIP_VISITS_DIM07_2_SHARE,
  IFNULL(SAFE_DIVIDE(VIP_VISITS_DIM08_2, VIP_VISITS_DIM07_2), 0) AS VIP_VISITS_DIM08_2_SHARE,
  IFNULL(SAFE_DIVIDE(MENOR2_1, VIP_VISITS_1), 0) AS MENOR2_1_PERC,
  IFNULL(SAFE_DIVIDE(MENOR2_2, VIP_VISITS_2), 0) AS MENOR2_2_PERC,
  MENOR2_1, MENOR2_2, VIP_VISITS_1, VIP_VISITS_2, PERIOD1, PERIOD2
FROM tabla1
),

SELECT
  SITE_ID,
  {DIM_COL} AS DIM_VALUE,
  SUM(VIP_VISITS_1) AS VIP_VISITS_1,
  SUM(VIP_VISITS_2) AS VIP_VISITS_2,
  SUM(MENOR2_1) AS MENOR2_1,
  SUM(MENOR2_2) AS MENOR2_2,
  SAFE_DIVIDE(SUM(MENOR2_1), SUM(VIP_VISITS_1)) AS PERC_LTE2D_P1,
  SAFE_DIVIDE(SUM(MENOR2_2), SUM(VIP_VISITS_2)) AS PERC_LTE2D_P2,
  SAFE_DIVIDE(SUM(VIP_VISITS_1), MAX(VIP_VISITS_TOTAL_1)) AS SHARE_P1,
  SAFE_DIVIDE(SUM(VIP_VISITS_2), MAX(VIP_VISITS_TOTAL_2)) AS SHARE_P2,
  SAFE_DIVIDE(SUM(MENOR2_1), MAX(VIP_VISITS_TOTAL_1)) - SAFE_DIVIDE(SUM(MENOR2_2), MAX(VIP_VISITS_TOTAL_2)) AS CONTRIBUCION_VAR_PP,
  MAX(PERIOD1) AS PERIOD1,
  MAX(PERIOD2) AS PERIOD2
FROM tabla3
GROUP BY 1, 2
ORDER BY ABS(CONTRIBUCION_VAR_PP) DESC
```

Para Deep Dive mensual: aplicar el mismo reemplazo de filtros que en Modo MONTH.

---

**Calendario de feriados nacionales — 🇧🇷 MLB (Brasil) 2026:**

Usar esta tabla para comentar automáticamente si alguno de los períodos comparados contiene feriados, especialmente cuando DIM08 (Efecto Calendario/Buffering) muestra variación significativa.

| Fecha | ISO Week | Feriado |
|---|---|---|
| 01/01/2026 | 2026_1 | Confraternização Universal |
| 16/02/2026 | 2026_8 | Carnaval (lunes) |
| 17/02/2026 | 2026_8 | Carnaval (martes) |
| 18/02/2026 | 2026_8 | Quarta-feira de Cinzas |
| 03/04/2026 | 2026_14 | Sexta-feira Santa |
| 21/04/2026 | 2026_17 | Tiradentes |
| 01/05/2026 | 2026_18 | Dia do Trabalho |
| 04/06/2026 | 2026_23 | Corpus Christi |
| 07/09/2026 | 2026_37 | Independência do Brasil |
| 12/10/2026 | 2026_42 | Nossa Senhora Aparecida |
| 02/11/2026 | 2026_45 | Finados |
| 15/11/2026 | 2026_46 | Proclamação da República |
| 20/11/2026 | 2026_47 | Dia da Consciência Negra |
| 25/12/2026 | 2026_52 | Natal |

**Calendario de feriados nacionales — 🇨🇱 MLC (Chile) 2026:**

| Fecha | ISO Week | Feriado |
|---|---|---|
| 01/01/2026 | 2026_1 | Año Nuevo |
| 03/04/2026 | 2026_14 | Viernes Santo |
| 04/04/2026 | 2026_14 | Sábado Santo |
| 01/05/2026 | 2026_18 | Día Nacional del Trabajo |
| 21/05/2026 | 2026_21 | Día de las Glorias Navales |
| 21/06/2026 | 2026_25 | Día Nacional de los Pueblos Indígenas |
| 29/06/2026 | 2026_27 | San Pedro y San Pablo |
| 16/07/2026 | 2026_29 | Día de la Virgen del Carmen |
| 15/08/2026 | 2026_33 | Asunción de la Virgen |
| 18/09/2026 | 2026_38 | Independencia Nacional (Fiestas Patrias) |
| 19/09/2026 | 2026_38 | Día de las Glorias del Ejército |
| 12/10/2026 | 2026_42 | Encuentro de Dos Mundos |
| 31/10/2026 | 2026_44 | Día de las Iglesias Evangélicas y Protestantes |
| 01/11/2026 | 2026_44 | Día de Todos los Santos |
| 08/12/2026 | 2026_50 | Inmaculada Concepción |
| 25/12/2026 | 2026_52 | Navidad |

**Calendario de feriados nacionales — 🇨🇴 MCO (Colombia) 2026:**

| Fecha | ISO Week | Feriado |
|---|---|---|
| 01/01/2026 | 2026_1 | Año Nuevo |
| 06/01/2026 | 2026_2 | Reyes Magos |
| 24/03/2026 | 2026_13 | Día de San José (trasladado) |
| 17/04/2026 | 2026_16 | Jueves Santo |
| 18/04/2026 | 2026_16 | Viernes Santo |
| 01/05/2026 | 2026_18 | Día del Trabajo |
| 02/06/2026 | 2026_23 | Ascensión del Señor (trasladado) |
| 23/06/2026 | 2026_26 | Corpus Christi (trasladado) |
| 30/06/2026 | 2026_27 | Sagrado Corazón de Jesús (trasladado) |
| 07/07/2026 | 2026_28 | San Pedro y San Pablo (trasladado) |
| 20/07/2026 | 2026_30 | Grito de Independencia |
| 07/08/2026 | 2026_32 | Batalla de Boyacá |
| 18/08/2026 | 2026_34 | Asunción de la Virgen (trasladado) |
| 13/10/2026 | 2026_42 | Día de la Raza (trasladado) |
| 03/11/2026 | 2026_45 | Todos los Santos (trasladado) |
| 17/11/2026 | 2026_47 | Independencia de Cartagena (trasladado) |
| 08/12/2026 | 2026_50 | Inmaculada Concepción |
| 25/12/2026 | 2026_52 | Navidad |

**Regla de uso:** al presentar cualquier resultado de waterfall para MLB, MLC o MCO, cruzar PERIOD1 y PERIOD2 contra esta tabla. Si algún período contiene feriados, mencionarlo en la respuesta con un comentario del estilo:
> 📅 *La semana 2026_8 incluye Carnaval (lunes, martes y Quarta-feira de Cinzas) — esto puede explicar variaciones en DIM01 (Día semana + Hora) y DIM08 (Efecto Calendario/Buffering).*

---

---

#### Guía de interpretación del Waterfall

**Regla crítica — MIX vs TASA:**
El waterfall mide EXCLUSIVAMENTE el efecto del cambio de SHARE (composición). NO mide si el % <=2D interno de un segmento mejoró o empeoró. Para saber si hubo degradación interna de tasa, hay que mirar el Deep Dive: comparar el % <=2D del segmento entre PERIOD1 y PERIOD2.

Ejemplo: DIM03_VAR = -0.46pp significa que el cambio en el mix de picking types explicó -0.46pp de la caída total. Puede ser porque FF perdió share (y FF tiene mejor promesa que XD), o porque XD ganó share (y XD tiene peor promesa). El Deep Dive muestra cuál ocurrió, y además si el % <=2D interno de FF o XD también cambió.

**Verificación matemática obligatoria:** METRIC2 + DIM01_VAR + ... + DIM09_VAR = METRIC1. Si no cierra, hay un error en los datos o en la lectura.

**Consideraciones de análisis:**
- **MIX vs TASA:** para confirmar si hubo degradación interna de tasa además del efecto mix, siempre mirar el Deep Dive
- **Puntual vs Estructural:** si el VAR vs Best Week es < 0.5pp el movimiento es puntual; si es > 1pp hay brecha estructural que viene de antes
- **Contribución vs % interno:** `CONTRIBUCION_VAR_PP` en el Deep Dive = efecto combinado mix + tasa sobre el total. `PERC_LTE2D_P1` vs `PERC_LTE2D_P2` = si ese segmento mejoró o empeoró internamente. Un segmento puede tener gran contribución por volumen aunque su tasa apenas se movió, y viceversa
- **Regla fundamental:** nunca comparar sites entre sí. Cada site se analiza de forma completamente independiente
- **Deep Dive Cruzado DIM04 × DIM02:** correr solo cuando DIM04 está en el top 3 de palancas. Muestra combinaciones FC origen × destino; identificar si la combinación perdió share (mix), mantuvo share pero bajó % <=2D (tasa), o ganó share con bajo % <=2D (mix negativo)

---

#### Interpretación de dimensiones — aplica a todos los sites

**DIM01 — Día de semana + Hora del día**
Mide el cambio en el mix de cuándo se hicieron las visitas. Los valores son combinaciones día+hora (ej: Monday08, Saturday22).
- VAR negativo: más visitas en fines de semana, noche o fuera del horario de corte de recolección (promesas peores)
- VAR positivo: más visitas en días hábiles, mañana o mediodía
- Hipótesis: evento o campaña que concentró tráfico en fin de semana / cambio en patrón de navegación

**DIM02 — Estado/provincia/departamento de destino**
Mide el cambio en la distribución geográfica de la demanda. Ver sección por site para los valores de cobertura específicos.
- VAR negativo: creció participación de destinos con promesas más lentas (zonas remotas, sin cobertura <=2D)
- VAR positivo: creció participación de destinos con buena cobertura
- Hipótesis: campañas en regiones específicas / cambio de cobertura de red / efecto estacional

**DIM03 — Picking type**
Mide el cambio en el mix de tipos logísticos. Ver sección por site para % <=2D típico y share por picking type.
- VAR negativo: creció XD o DS a costa de FF, o FF perdió share hacia XD
- VAR positivo: creció FF o FLEX a costa de XD o DS
- Hipótesis: cambio en mix de sellers activos / migración entre logistic types / degradación interna de FF

**DIM04 — FC de origen (FF) / Estado origen (XD)**
Mide el cambio en qué centros o hubs concentraron el volumen. Ver sección por site para FCs principales y su % <=2D.
- Para FF: qué FC ganó o perdió participación
- Para XD: qué estado de origen cambió su peso
- VAR negativo: más volumen desde FCs con peor promesa / pérdida de share de FCs buenos
- Si DIM04 en top 3 → correr también Deep Dive Cruzado DIM04 × DIM02
- Hipótesis: cambios operacionales en algún FC / activación de Custom Offsets / cambio de volumen de seller grande

**DIM05 — Handling time**
Mide el cambio en el mix de tiempos de preparación de los sellers (SD=0hs, ND=24hs, 2D=48hs, MAS2=+48hs).
- VAR negativo: más volumen en sellers con HT alto (MAS2 o 2D)
- VAR positivo: más volumen en sellers con HT bajo (SD o ND)
- Hipótesis: nuevos sellers con HT alto / seller grande cambió su SLA / HT degradado por stock o capacidad

**DIM06 — Shipping time**
Mide el cambio en el mix de tiempos de tránsito de las rutas (SD=0hs, ND=24hs, 2D=48hs, MAS2=+48hs).
- VAR negativo: más rutas con TT alto activas en el período
- VAR positivo: más rutas con TT bajo
- Hipótesis: rutas más lentas activadas hacia nuevas zonas / cambios en configuración de TT / expansión geográfica

**DIM07 — Flag offset**
Mide promesas donde handling + shipping + offset cabe dentro del UB, pero el UB es > 2 días. Son visitas que podrían haber sido <=2D pero el offset las empujó fuera del rango. Deep Dive: OFFSET vs OTHER.
- VAR negativo: más visitas afectadas por offsets que las sacaron del <=2D
- Hipótesis: nuevos Custom Offsets en nodos clave / aumento de offsets en FCs o hubs principales

**DIM08 — Efecto calendario / buffering / feriados**
Mide el gap entre días operativos y delivery bounds. Este componente es SIEMPRE efecto de mix puro — el % <=2D interno de TWODAYS es 100%, de FDS y OTHER es 0%, no hay degradación de tasa posible.
- TWODAYS: promesas con UB ≤ 2 días → contribuyen al % <=2D
- FDS: promesas que podrían haber sido <=2D pero no lo fueron por fines de semana, feriados o buffering
- OTHER: promesas que nunca podrían haber sido <=2D (estructuralmente fuera del rango)
- VAR negativo: creció FDS u OTHER a costa de TWODAYS
- Si creció FDS: verificar tabla de feriados y eventos. Un feriado en PERIOD1 que no existía en PERIOD2 explica un VAR negativo estructural — no es un problema operacional, mencionarlo explícitamente
- Si creció OTHER: puede indicar buffering de capacidad activado o crecimiento de rutas/destinos estructuralmente lentos

**DIM09 — Residuo (tasa pura)**
Variación que NO se explica por cambios de mix en ninguna dimensión. Es el cambio "puro" en la calidad de la promesa.
- |DIM09| > 0.3pp: movimiento estructural — degradación o mejora real de la red (cambios en algoritmo, ajustes masivos de TT/HT, política global de offsets)
- DIM09 ≈ 0: la variación se explica casi completamente por efectos de mix; la red en sí no cambió

---

#### Contexto por site — valores de referencia para interpretación

**🇧🇷 MLB — Brasil**

*DIM02 — Cobertura geográfica:*
- Mejor cobertura <=2D: SP, RJ, MG, PR (zona sureste)
- Peor cobertura: regiones Norte y Nordeste (AM, PA, MA, CE, etc.)

*DIM03 — % <=2D típico y share por picking type:*
| Picking Type | % <=2D típico | Share típico |
|---|---|---|
| FLEX (Self Service) | ~90-96% | ~3% |
| FULFILLMENT (FF) | ~58-62% | ~38% |
| CROSS_DOCKING (XD) | ~9-11% | ~52% |
| DROP_OFF (DS) | ~0% | ~7% |

XD es dominante en volume (~52%) pero con muy bajo % <=2D → cualquier ganancia de share de XD sobre FF impacta negativamente el agregado.

*DIM04 — FCs principales y % <=2D aproximado:*
| FC | Ubicación | % <=2D aprox. |
|---|---|---|
| BRRJ02 | Rio de Janeiro | ~75-80% |
| BRSP02 | São Paulo | ~65-68% |
| BRSP04 | São Paulo 2 | ~63-66% |
| BRSC02 | Santa Catarina | ~60-63% |
| BRBA01 | Bahia | ~55-60% |
| BRPE01 | Pernambuco | ~50-60% |

Si DIM04 en top 3 → Deep Dive Cruzado: combinaciones críticas son FCs del Nordeste (BRPE01, BRBA01) hacia estados remotos del Norte, o FCs de SP hacia estados fuera del eje SP-RJ-MG. Combinaciones con % <=2D < 30% que ganen share son alerta roja.

*Eventos comerciales MLB:*
| Mes | Evento | Efecto esperado |
|---|---|---|
| Mayo | Hot Sale | Caída por demanda y buffering |
| Mayo | Dia das Maes | Pico de demanda, posible caída |
| Junio | Dia dos Namorados | Pico moderado |
| Noviembre | Black Friday | Caída por demanda y buffering |
| Noviembre | Cyber Week | Caída sostenida |
| Diciembre | Navidad / Fin de año | Caída por saturación logística |

---

**🇨🇱 MLC — Chile**

*DIM02 — Cobertura geográfica:*
- Mejor cobertura <=2D: RM (77.39%), Valparaíso (58.89%), Libertador B. O'Higgins (57.57%), Biobío (56.04%), Ñuble (55.79%)
- Peor cobertura: Aysén (10.39%), Magallanes (12.71%), Tarapacá (18.92%), Antofagasta (18.95%), Arica y Parinacota (19.12%)
- La geografía de Chile impacta estructuralmente: zona central concentra la mejor cobertura, extremos norte y sur enfrentan desafíos logísticos

*DIM03 — % <=2D típico y share por picking type:*
| Picking Type | % <=2D típico | Share típico |
|---|---|---|
| SELF_SERVICE (FLEX) | ~97% | ~27% |
| FULFILLMENT (FF) | ~86% | ~41% |
| XD_DROP_OFF | ~59% | ~16% |
| CROSS_DOCKING (XD) | ~58% | ~14% |
| DROP_OFF (DS) | ~42% | ~1% |

FF es el tipo dominante por share (~41%) y tiene muy alto % <=2D → su variación de share o tasa interna tiene gran impacto en el agregado.

*DIM04 — Centros/hubs principales y % <=2D aproximado:*
| Hub | Ubicación | % <=2D aprox. |
|---|---|---|
| SRM1 | Santiago RM | ~86-89% |
| SRM2 | Santiago RM | ~84-85% |
| SVP3 | Valparaíso | ~81-82% |
| SBB1 | Bío Bío | ~80-81% |
| STC1 | Temuco/Araucanía | ~78-79% |

Si DIM04 en top 3 → Deep Dive Cruzado: combinaciones críticas son centros del norte y extremo sur hacia destinos remotos (SAF1, SAR1 hacia regiones extremas; SPU1, SXI1 hacia Magallanes y Aysén). Combinaciones con % <=2D < 45% que ganen share son alerta.

*Eventos comerciales MLC:*
| Mes | Evento | Efecto esperado |
|---|---|---|
| Mayo | Hot Sale | Caída por demanda y buffering |
| Mayo | Día de la Madre | Pico de demanda, posible caída |
| Junio | Día de los Enamorados | Pico moderado |
| Noviembre | Black Friday | Caída por demanda y buffering |
| Noviembre | Cyber Week | Caída sostenida |
| Diciembre | Navidad / Fin de año | Caída por saturación logística |

---

**🇨🇴 MCO — Colombia**

*DIM02 — Cobertura geográfica:*
- Mejor cobertura <=2D: Bogotá D.C. (64.81%), Cundinamarca (42.10%), Antioquia (38.70%), Tolima (32.76%), Boyacá (32.70%)
- Peor cobertura: Chocó (0%), Guainía (0%), San Andrés y Providencia (0%), Vichada (0%), Putumayo (0%)
- Brecha de ~65pp entre mejor y peor región. Los departamentos con 0% hacen que cualquier crecimiento de demanda ahí impacte directamente el % <=2D agregado sin posibilidad de compensación

*DIM03 — % <=2D típico y share por picking type:*
| Picking Type | % <=2D típico | Share típico |
|---|---|---|
| SELF_SERVICE (FLEX) | ~94% | ~13% |
| FULFILLMENT (FF) | ~64% | ~24% |
| CROSS_DOCKING (XD) | ~37% | ~23% |
| XD_DROP_OFF | ~31% | ~33% |
| DROP_OFF (DS) | ~0% | ~7% |

⚠️ ATENCIÓN MCO: XD_DROP_OFF es el picking type DOMINANTE (33% de share) pero con solo 31% de <=2D → es el principal factor estructural de arrastre hacia abajo del agregado. Cualquier ganancia de share de XD_DO sobre FF o SELF_SERVICE tiene impacto negativo visible e inmediato.

*DIM04 — Hubs principales y % <=2D aproximado:*
| Hub | % <=2D aprox. | Notas |
|---|---|---|
| SBO1 | ~81% | Mejor hub, CO ST bajo (1.63%), buffering bajo (2.43%) |
| SCI1 | ~73% | CO ST bajo (1.35%), buffering bajo (2.67%) |
| SAN1 | ~73% | CO ST 4.53%, buffering 3.67% |
| SRI1 | ~72% | CO ST 5.48%, buffering 4.40% |
| SVA1 | ~66% | CO ST 4.68%, buffering 3.17% |
| SAT1, SBL1, SNS1, SCS1, SCR1, SCA1 | 24-32% | Hubs de bajo desempeño — alto riesgo si ganan share |
| SNA1, SAN2 | 10-16% | Hubs críticos — impacto negativo inmediato si crecen |

Hubs con mayor CO ST (impacto en DIM07): SCR1 (23.14%), SCS1 (19.16%), SHU2 (8.11%), SSA1 (8.09%)
Hubs con mayor Buffering (impacto en DIM08): SSA1 (8.04% LM), SNA1 (6.84%), SCA1 (6.30%)

Si DIM04 en top 3 → Deep Dive Cruzado: combinaciones críticas son hubs de bajo desempeño (SAT1, SBL1, SNA1, SAN2, SCR1, SCS1, SCA1) hacia cualquier destino, o SBO1/SCI1 hacia departamentos remotos de Amazonía u Orinoquía. Combinaciones con % <=2D < 45% que ganen share son alerta.

*Nota especial MCO — festivos trasladados:* los feriados trasladados siempre caen en lunes, afectando de forma predecible el share de FDS en DIM08.

*Eventos comerciales MCO:*
| Mes | Evento | Efecto esperado |
|---|---|---|
| Febrero | Día de San Valentín | Pico moderado |
| Mayo | Día de la Madre | Pico de demanda, posible caída |
| Junio | Hot Sale | Caída por demanda y buffering |
| Junio | Día del Padre | Pico moderado |
| Septiembre | Amor y Amistad | Pico significativo (equivalente a San Valentín) |
| Octubre | Días sin IVA | ⚠️ Caída crítica por pico de demanda y buffering |
| Noviembre | Black Friday | Caída por demanda y buffering |
| Noviembre | Cyber Week | Caída sostenida |
| Diciembre | Navidad / Fin de año | Caída por saturación logística |

---

**Formato de respuesta del Waterfall:**

Presentar siempre en este orden:

1. **Headline** (antes de la tabla):
   `% <=2D [P2]: XX.XX% → [P1]: XX.XX% = [signo][VAR] pp [✅/❌] — [Ruido/Moderada/Significativa]`
   - Ruido: |var| < 1 pp · Moderada: 1–2 pp · Significativa: > 2 pp

2. **Tabla de waterfall completa**

3. **Línea de cierre** con la palanca más positiva y más negativa (solo si |var| > 0.1 pp)

```
🌊 VIP Promise % <=2D Waterfall — 🏳️ SITE | P2 → P1
% <=2D P2: XX.XX% → P1: XX.XX% = +/-X.XX pp ✅/❌ (Ruido/Moderada/Significativa)

| Palanca                  | Contribución   |   |
|--------------------------|----------------|---|
| % <=2D base (P2)         | XX.XX%         |   |
| Día semana + Hora        | +/-X.XX pp     | ✅/❌ |
| Mix Geográfico           | +/-X.XX pp     | ✅/❌ |
| Picking Type             | +/-X.XX pp     | ✅/❌ |
| FC Origen                | +/-X.XX pp     | ✅/❌ |
| Handling Time            | +/-X.XX pp     | ✅/❌ |
| Shipping Time            | +/-X.XX pp     | ✅/❌ |
| Flag Offset              | +/-X.XX pp     | ✅/❌ |
| Efecto Cal/Buffering     | +/-X.XX pp     | ✅/❌ |
| Residuo tasa pura        | +/-X.XX pp     | 🔵 |
| % <=2D resultado (P1)    | XX.XX%         |   |
```

**Formato de respuesta del Deep Dive:**

Presentar tabla con los valores de la dimensión ordenados por `CONTRIBUCION_VAR_PP` DESC (mayor impacto primero), mostrando % <=2D en cada período y contribución en pp. Top 15 filas máximo.

---

### Paso 4 — Ejecutar el SQL

Construir el SQL como string en PowerShell y escribirlo en `$env:TEMP\yb.sql` **sin BOM** (UTF-8 encoding sin BOM es obligatorio para que bq lo acepte):

```powershell
$q = @"
<SQL aqui, con backticks dobles `` para los backticks de BigQuery>
"@
$enc = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText("$env:TEMP\yb.sql", $q, $enc)
cmd /c "bq.cmd query --project_id=meli-bi-data --use_legacy_sql=false --format=prettyjson --max_rows=500 < $env:TEMP\yb.sql" 2>&1
"BQ_EXIT:$LASTEXITCODE"
```

**Notas importantes:**
- Usar `New-Object System.Text.UTF8Encoding $false` para escribir sin BOM (si hay BOM, bq rechaza el archivo con "Illegal input character")
- En el here-string de PowerShell, los backticks de BigQuery (`) se escriben como doble backtick (``)
- Ejecutar via `cmd /c "bq.cmd ..."` con redirección `<` (no pipe `|`)

- `BQ_EXIT:0` con datos → continuar al Paso 5
- `BQ_EXIT:0` con `[]` → responder "No se encontraron datos para esa consulta."
- `BQ_EXIT:` distinto de 0 → mostrar el error de bq e intentar corregir el SQL una vez. Si falla de nuevo, mostrar el SQL y el error explicando el problema.

---

### Paso 5 — Formatear y responder

#### Reglas de respuesta obligatorias

**Flags por site** (usar en títulos y menciones de sitios):
- 🇦🇷 MLA — Argentina
- 🇧🇷 MLB — Brasil
- 🇲🇽 MLM — México
- 🇨🇱 MLC — Chile
- 🇨🇴 MCO — Colombia
- 🇺🇾 MLU — Uruguay
- 🇵🇪 MPE — Perú

**Indicadores de variación** (comparar períodos o valores contra umbral):
Consultar la tabla de métricas abajo para saber si "low" o "high" es positivo, y señalizar:
- Variación en dirección positiva → ✅
- Variación en dirección negativa → ❌

**Tabla de métricas — dirección positiva:**
| Métrica | Positivo |
|---|---|
| % CO ST | Low ↓ |
| % CO ST Shift | Low ↓ |
| % CO ST Expand | Low ↓ |
| % Buffering | Low ↓ |
| % Buffering Operacional | Low ↓ |
| % Buffering Operacional FBM | Low ↓ |
| % Buffering Middle Mile | Low ↓ |
| % Buffering Last Mile | Low ↓ |
| % Buffering Seller | Low ↓ |
| % Buffering First Mile | Low ↓ |
| % Delay | Low ↓ |
| % Early | High ↑ |
| % On Time | High ↑ |
| CVR / Conversión | High ↑ |
| Promesa promedio (días) | Low ↓ |
| Alarmas activas | Low ↓ |

**Formato de respuesta:**
- Usar emojis para hacer la respuesta más visual
- Presentar tablas markdown para comparaciones
- Incluir porcentajes formateados a 2 decimales (ej: 12.34%)
- Si hay comparación entre períodos, mostrar el delta y el indicador ✅/❌
- Para alarmas: agrupar por PROCESS_NAME y dimensión
- Para métricas mixtas (promesas + shipments): presentar side-by-side con labels explícitos "(VIPs)" y "(Shipments)"

**Regla crítica:** Nunca mezclar numeradores de VIPs con denominadores de Shipments ni viceversa.

**Si no entendés la consulta:** No inferir ni inventar. Decir qué parte no queda clara y hacer una sola pregunta concisa para desbloquearse.
