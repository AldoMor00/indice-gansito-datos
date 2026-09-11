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

Un directorio por fuente, cada uno con su manifiesto. Las dos fuentes no se parecen y no
comparten índice: ver la decisión #9 del repositorio de código.

```
profeco/precios/anio=YYYY/qqp_YYYY-MM_qN.parquet     filas del catálogo objetivo
profeco/tiendas/anio=YYYY/tiendas_YYYY-MM_qN.parquet tiendas del archivo completo
profeco/manifiesto.jsonl                             una línea por archivo procesado

conasami/salarios/*.csv                              salario mínimo, tal cual se sirve
conasami/manifiesto.jsonl                            una línea por versión

publico/*.parquet                                    las ocho tablas de gold, salida
```

`profeco/` y `conasami/` son **entrada** a Fabric. `publico/` es **salida**: lo escribe
`nb_50_export` y lo lee, por URL anónima, el modelo import del reporte público.

Son las ocho tablas de gold tal cual, un parquet plano por tabla —sin particionar y sin
`_delta_log`—, 1.36 MB entre todas. Están aquí porque la capacidad de Fabric es una trial y
el reporte público necesita un origen que le sobreviva; bronze y silver no, porque nada
fuera de Fabric los lee. La decisión #6 del repositorio de código lo explica.

## Lo de Profeco no es el archivo original

De cada CSV de Profeco (~155 MB) se persisten sólo dos cortes: las filas que cumplen el
catálogo objetivo, y las tuplas distintas de tienda. **El archivo íntegro no se guarda.**

Es una concesión deliberada por el presupuesto de un portafolio, no una buena práctica.
Se mitiga con el manifiesto: guarda el `sha256` y la URL de origen de cada archivo, así
que cualquier corte puede rehacerse desde la fuente de forma verificable.

Las primeras 46 líneas traen una `url_origen` de `repodatos.atdt.gob.mx`, que hoy contesta
503. Su `sha256` sigue siendo válido: Profeco publica ahora por bundle anual en su portal
y esos bundles traen los mismos archivos, byte por byte —verificado sobre las 46—. La
decisión #34 del repositorio de código lo explica.

Lo de CONASAMI se guarda entero, sin cortar y sin convertir: son 40 KB entre los dos
archivos, así que no hay nada que ganar recortándolos.

## Los manifiestos

Uno por fuente. Son el índice: GitHub no expone listado de directorio, así que es la
única forma de saber qué hay aquí sin adivinar rutas.

`profeco/manifiesto.jsonl` — una línea por quincena procesada:

```json
{
  "url_origen": "https://datos.profeco.gob.mx/datos_abiertos/file.php?t=...#QQP_2026/01-2026_Q1.csv",
  "sha256": "...",
  "bytes": 162849302,
  "filas_leidas": 1284933,
  "filas_filtradas": 4118,
  "quincena": "2025-01_q1",
  "descargado_utc": "2026-01-06T13:04:11Z",
  "intento": 1
}
```

`conasami/manifiesto.jsonl` — una línea por versión de cada archivo:

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

En las dos, el `sha256` es lo que detecta que la fuente republicó algo. Cambia en
Profeco y la quincena se rebaja con un `intento` nuevo; cambia en CONASAMI y entra una
`version` nueva. En ninguna se pisa lo anterior: bronze conserva las dos.

## Fuentes

Datos abiertos del Gobierno de México, las dos vía `repodatos.atdt.gob.mx`:

- Profeco, *Quién es Quién en los Precios*
- CONASAMI, salario mínimo

Qué trae cada una y qué no es obvio de ellas está en
[`docs/fuentes.md`](https://github.com/AldoMor00/indice-gansito/blob/main/docs/fuentes.md).
