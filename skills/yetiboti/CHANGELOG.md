# Changelog - YetiBoti

## v1.4 - 2026-06-04
- Fast/Slow en VIPs: campos VISITS_SLOW, PROMISE_SLOW_LB, PROMISE_SLOW_UB con formulas
- Fast/Slow en Shipments: logica de diferencia (TOTAL-FAST) para metricas Slow
- Feriados nacionales MLB (Brasil) y MLC (Chile) y MCO (Colombia) 2026 para analisis de waterfall
- Tabla de BI (DM_SHP_TRACKS_VIP_CONVERTION): triggers, filtros obligatorios, regla COUNT_TRACK_VIP, campos disponibles y logica de convivencia de promesas

## v1.3 - 2026-06-03
- Nuevo bloque WATERFALL: soporte semanal y mensual de % <=2D en VIPs
- Query waterfall corregida: CTEs planos (fix syntax error BigQuery)
- Periodos relativos resueltos con CURRENT_DATE() dentro de la query principal
- Formato de respuesta: headline con variacion total antes de la tabla
- Clasificacion: Ruido (<1pp), Moderada (1-2pp), Significativa (>2pp)
- Deep Dive a pedido por dimension: DIM01 a DIM08

## v1.2 - 2026-06-03
- PROMISE_CVR: definiciones exactas de numeradores por metrica
- PROMISE_CVR: logica de fechas Domingo-Sabado, Top Cities, ambiguedad ciudad/estado
- PROMISE_CVR: regla de no agregar filtros no especificados
- Nuevo bloque IDEAL_PROMISE con metricas SD/ND/2D Ideal, Real y variante NBC

## v1.1 - 2026-06-03
- Auto-chequeo de version al usar la skill
- Comando /yetiboti actualizar para actualizar automaticamente desde GitHub

## v1.0 - 2026-06-03
- Lanzamiento inicial: Alarmas, Promesas/CVR, Shipments/Lead Time