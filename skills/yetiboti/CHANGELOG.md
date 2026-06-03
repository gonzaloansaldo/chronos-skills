# Changelog - YetiBoti

## v1.3 - 2026-06-03
- Nuevo bloque WATERFALL: soporte para waterfall semanal y mensual de % <=2D en VIPs
- Query waterfall corregida: CTEs planos en lugar de nested WITH (fix de syntax error en BigQuery)
- Periodos relativos resueltos dentro de la query principal con CURRENT_DATE() (elimina round-trip extra)
- Formato de respuesta waterfall mejorado: headline con variacion total antes de la tabla
- Clasificacion de variacion: Ruido (<1pp), Moderada (1-2pp), Significativa (>2pp)
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