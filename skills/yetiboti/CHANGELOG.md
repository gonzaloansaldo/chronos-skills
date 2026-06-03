# Changelog - YetiBoti

## v1.2 - 2026-06-03
- PROMISE_CVR: definiciones exactas de numeradores por metrica (% SD, % ND, % <=ND, % <=2D, etc.)
- PROMISE_CVR: logica de fechas Domingo-Sabado con BETWEEN y CURRENT_DATE(), prohibicion de VIP_DS_WEEK/VIP_DS_ISOYEAR
- PROMISE_CVR: soporte Top Cities con JOIN a LK_PROMISE_TOP_CITIES_FIXED y agrupacion Other Cities
- PROMISE_CVR: manejo de ambiguedad Ciudad vs. Estado (UNION ALL para Sao Paulo, Buenos Aires, etc.)
- PROMISE_CVR: regla de no agregar filtros no especificados
- Nuevo bloque IDEAL_PROMISE: tabla BT_SHP_IDEAL_PROMISE con metricas SD/ND/2D Ideal, Real y variante NBC

## v1.1 - 2026-06-03
- Auto-chequeo de version al usar la skill
- Comando /yetiboti actualizar para actualizar automaticamente desde GitHub

## v1.0 - 2026-06-03
- Lanzamiento inicial
- Soporte para Alarmas (BT_CRB_ALARMS)
- Soporte para Promesas / CVR (BT_SPEED_PROMISE_VIP_CVR)
- Soporte para Shipments / Lead Time (BT_SPEED_PROMISE_LT_SUMMARY)