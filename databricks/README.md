# Pipeline Footloose en Databricks (medallion)

> Volver al [README principal](../README.md).

Este directorio contiene el pipeline de datos de Footloose implementado como notebook en Databricks Cloud, organizado en 4 secciones:

1. **Setup** — importación de librerías, definición de esquemas y rutas del catálogo `workspace`.
2. **01_bronce** — ingesta de datos brutos con Auto Loader (`cloudFiles`) y escritura en tablas Delta (clientes, productos, transacciones).
3. **02_silver** — limpieza, deduplicado, tipado y enriquecimiento (rango de edad calculado al 2025-12-31, sanitización de clientes y productos, validación de transacciones).
4. **03_gold** — joins entre silver, cálculo de importes (ingresos, costos, ganancia) y escritura de la tabla `gold_reporte_ventas` + 4 tablas agregadas para negocio (método de pago, canal, estado, top productos).

## Cómo ejecutar

Abre el notebook en **Databricks Cloud** y ejecuta las celdas en orden (1 → 2 → 3 → 4). Revisa el esquema y las rutas antes de ejecutar.

- Datos de entrada: `/Volumes/workspace/footloose_bronce/raw_data/` (`clientes/clientes.csv`, `productos/productos.csv`, `transacciones/transacciones_*.json`).
- Checkpoints: `/Volumes/workspace/footloose_bronce/checkpoints/`.
- Tablas Gold: catálogo `workspace` / esquema `footloose_gold`.

## Flujo de datos

```
raw_data/                 Databricks Cloud                            Power BI
  clientes/clientes.csv    ───────► bronze_clientes ───────────► silver_clientes
  productos/productos.csv  ───────► bronze_productos ──────────► silver_productos
  transacciones/*.json     ───────► bronze_transacciones ──────► silver_transacciones
                                                                    │
Databricks (gold_reporte_ventas) ◄────────────────────────────────┘
        └── 4 tablas agregadas (metodo_pago, canal,
            estado_venta, top_producto)
```

## Databricks + Power BI

El proyecto usa Databricks para el ELT y Power BI para el consumo:

- Databricks ejecuta el pipeline medallion (bronze → silver → gold) de forma incremental con Auto Loader y checkpoints.
- Power BI se conecta con el conector `DatabricksMultiCloud` y **importa** los datos (modo import), para que el dashboard responda sin latencia en los KPIs.
- El resultado importado es `gold_reporte_ventas` (tabla de hechos) más `silver_clientes` y `silver_productos` (dimensiones). Las tablas agregadas de gold quedan disponibles en Databricks pero no se importan en el `.pbix`; típicamente se refresca el informe al haber nuevos datos.
- Las escrituras de `silver_clientes`, `silver_productos`, `silver_transacciones` y `gold_reporte_ventas` usan `MERGE` por clave de negocio con `foreachBatch`: re-ejecutar el pipeline no duplica filas (idempotente ante reintentos).

## Scripts de prueba / reproducibilidad

`raw_data/` incluye datos sintéticos para pruebas (30 clientes, 50 productos, 150 transacciones distribuidas en 2023–2025). El pipeline los lee desde el volumen y escribe las tablas Gold correspondientes. Para reproducir localmente, sube los archivos a `/Volumes/.../raw_data/` y ejecuta el notebook.

## Dashboard de Power BI

El archivo [`powerbi/footloose-sales-dashboard.pbix`](../powerbi/footloose-sales-dashboard.pbix) consume `footloose_gold.gold_reporte_ventas`, `silver_clientes` y `silver_productos`, en modo importación. No requiere credenciales para visualizar; para actualizar desde el origen sí hace falta acceso al workspace de Databricks.

Vista previa sin Power BI: [`powerbi/exports/footloose-sales-dashboard.pdf`](../powerbi/exports/footloose-sales-dashboard.pdf).
