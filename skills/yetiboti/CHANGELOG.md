# Changelog - YetiBoti

## v1.6 - 2026-06-04
- VIP_CVR schema: agregados 28 campos faltantes vs Excel (TRK_ORIGIN_ADDRESS_0_VALUE_ID, campos Slow/Fast, ventanas granulares WINDOW_1-4_X, WINDOW_VIPS_UB_MENOR_2, HT breakdown, pricing, PROMISE_LB/UB, VIP_DS_QUARTER)
- LT_SUMMARY schema: agregados 8 campos faltantes vs Excel (SHP_LT_REAL_3D/4D/5D/MENOR_5D, SHP_XD_TOTAL, SHP_XD_DO, SHP_ONTIME_VENTANA_UB, SHP_PROMISE_WEEKEND)

## v1.5 - 2026-06-04
- Waterfall: guia completa de interpretacion por dimension (DIM01-DIM09)
- Waterfall: contexto especifico por site para MLB, MLC y MCO (DIM02/03/04 con datos reales, eventos comerciales)

## v1.4 - 2026-06-04
- Fast/Slow en VIPs y Shipments
- Feriados nacionales MLB, MLC y MCO 2026
- Tabla de BI: triggers, filtros, COUNT_TRACK_VIP, campos y logica de convivencia
- Picking types: mapeo global de aliases

## v1.3 - 2026-06-03
- Waterfall: queries corregidas, headline formato, fechas con CURRENT_DATE()

## v1.2 - 2026-06-03
- PROMISE_CVR: numeradores exactos, fechas Dom-Sab, Top Cities, ciudad/estado, filtros

## v1.1 - 2026-06-03
- Auto-chequeo de version y comando /yetiboti actualizar

## v1.0 - 2026-06-03
- Lanzamiento inicial