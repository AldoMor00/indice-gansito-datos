# indice-gansito-datos

Zona raw del proyecto [Índice Gansito](https://github.com/AldoMor00/indice-gansito).

**Este repositorio lo escribe GitHub Actions, no una persona.** Fabric lee de aquí.

## Por qué los datos están en git

Porque la capacidad de Fabric es una trial y va a desaparecer, y con ella OneLake.
Con el histórico afuera, Fabric queda desechable: se borra el workspace y se
reconstruye sin perder un día de historia. El razonamiento completo está en
[`docs/decisiones.md`](https://github.com/AldoMor00/indice-gansito/blob/main/docs/decisiones.md)
del repositorio de código.

## Estructura

```
precios/anio=YYYY/qqp_YYYY-MM_qN.parquet     filas del catálogo objetivo
tiendas/anio=YYYY/tiendas_YYYY-MM_qN.parquet tiendas distintas del archivo completo
publico/                                      agregados de gold, salida del pipeline
manifiesto.jsonl                              procedencia, una línea por archivo
```

`precios/` y `tiendas/` son **entrada** a Fabric. `publico/` es **salida**: lo escribe
el último paso del pipeline y lo consume el reporte de la cuenta gratuita de Power BI.

## Esto no es el archivo original

De cada CSV de Profeco (~155 MB) se persisten sólo dos cortes: las filas que cumplen el
catálogo objetivo, y las tuplas distintas de tienda. **El archivo íntegro no se guarda.**

Es una concesión deliberada por el presupuesto de un portafolio, no una buena práctica.
Se mitiga con el manifiesto: guarda el `sha256` y la URL de origen de cada archivo, así
que cualquier corte puede rehacerse desde la fuente de forma verificable.

## manifiesto.jsonl

Una línea JSON por archivo procesado:

```json
{
  "url_origen": "https://repodatos.atdt.gob.mx/api_update/profeco/...",
  "sha256": "...",
  "bytes": 162849302,
  "filas_leidas": 1284933,
  "filas_filtradas": 4118,
  "quincena": "2025-01_q1",
  "descargado_utc": "2026-01-06T13:04:11Z",
  "intento": 1
}
```

El `sha256` es lo que detecta que Profeco republicó una quincena corregida: si cambia,
se vuelve a bajar con un `intento` nuevo y bronze conserva las dos versiones.

## Fuente

Profeco, *Quién es Quién en los Precios*, vía `repodatos.atdt.gob.mx`.
Datos abiertos del Gobierno de México.
