# indice-gansito-datos

Zona raw del proyecto [Índice Gansito](https://github.com/AldoMor00/indice-gansito).

**Este repositorio lo escriben procesos, no una persona**: la ingesta de GitHub Actions y, en
`publico/`, el export de gold desde Fabric. Fabric lee de aquí.

Los datos viven en git y no en Fabric porque la capacidad es una trial y va a desaparecer; con el
histórico afuera, el workspace se borra y se reconstruye sin perder un día. Es la
[decisión #1](https://github.com/AldoMor00/indice-gansito/blob/main/docs/decisiones.md) del
repositorio de código; las demás que explican este repo son la #2 (qué se guarda de Profeco), la
#9 (por qué cada fuente va por su lado), la #34 (de dónde se baja Profeco hoy) y la #6 (la copia
pública de gold).

## Estructura

Un directorio por fuente, cada uno con su manifiesto. No comparten índice.

```
profeco/precios/anio=YYYY/qqp_YYYY-MM_qN.parquet     filas del catálogo objetivo
profeco/tiendas/anio=YYYY/tiendas_YYYY-MM_qN.parquet tiendas del archivo completo
profeco/manifiesto.jsonl                             una línea por archivo procesado

conasami/salarios/*.csv                              salario mínimo, tal cual se sirve
conasami/manifiesto.jsonl                            una línea por versión

inpc/serie/inpc_quincenal.json                       INPC quincenal, tal cual lo sirve INEGI
inpc/manifiesto.jsonl                                una línea por versión

publico/*.parquet                                    las ocho tablas de gold
```

`profeco/`, `conasami/` e `inpc/` son **entrada** a Fabric. `publico/` es **salida**: lo exporta
`nb_50_export` desde gold, y lo lee, por URL anónima, el modelo import del reporte público.

## Lo de Profeco no es el archivo original

De cada CSV de Profeco se persisten dos cortes: las filas que cumplen el catálogo objetivo, y las
tuplas distintas de tienda. **El archivo íntegro no se guarda.** Es una concesión por el
presupuesto de un portafolio, no una buena práctica, y se mitiga con el manifiesto: guarda el
`sha256` y la URL de origen de cada archivo, así que cualquier corte puede rehacerse desde la
fuente de forma verificable.

Las líneas más viejas traen una `url_origen` de `repodatos.atdt.gob.mx`, que hoy contesta 503.
Su `sha256` sigue siendo válido: los bundles anuales del portal de Profeco traen los mismos
archivos byte por byte.

CONASAMI e INEGI se guardan enteros, sin cortar ni convertir. La `url_origen` de INEGI lleva
`{token}` en lugar del valor.

## Los manifiestos

Uno por fuente. Son el índice: GitHub no expone listado de directorio.

`profeco/manifiesto.jsonl`, una línea por quincena procesada:

```json
{
  "url_origen": "https://datos.profeco.gob.mx/datos_abiertos/file.php?t=...#QQP_2026/01-2026_Q1.csv",
  "sha256": "...",
  "crc32": 4049749088,
  "codificacion": "utf-8",
  "bytes": 162849302,
  "filas_leidas": 1284933,
  "filas_filtradas": 4118,
  "quincena": "2025-01_q1",
  "descargado_utc": "2026-01-06T13:04:11Z",
  "intento": 1
}
```

El `crc32` es el que el zip del portal guarda en su directorio central, para detectar sin
descomprimir que Profeco reescribió una quincena ya procesada. Cuando pasa, nada se sobrescribe:
la versión corregida entra como `intento` nuevo, con su parquet `_iN`, y la línea del intento más
alto es la que manda. `codificacion` dice cómo se leyó el CSV (`utf-8` o `cp1252`).

`conasami/manifiesto.jsonl` e `inpc/manifiesto.jsonl`, una línea por versión de cada archivo:

```json
{
  "url_origen": "https://repodatos.atdt.gob.mx/api_update/conasami/...",
  "sha256": "...",
  "bytes": 18360,
  "filas": 685,
  "archivo": "sm_real_indice",
  "descargado_utc": "2026-08-31T12:00:04Z",
  "version": 1
}
```

En las tres, el `sha256` es lo que detecta que la fuente republicó algo. Cambia en Profeco y la
quincena entra con un `intento` nuevo; cambia en CONASAMI o INEGI y entra una `version` nueva.
Nunca se pisa lo anterior.

## Fuentes

- **Profeco**, *Quién es Quién en los Precios*, del portal `datos.profeco.gob.mx`.
- **CONASAMI**, salario mínimo, de `repodatos.atdt.gob.mx`.
- **INEGI**, INPC quincenal, de su API de indicadores.

Qué trae cada una y qué no es obvio de ellas está en
[`docs/fuentes.md`](https://github.com/AldoMor00/indice-gansito/blob/main/docs/fuentes.md).
