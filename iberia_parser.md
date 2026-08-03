# Importador CSV de Iberia

## 1. Objetivo

Procesar en el navegador el CSV mensual exportado por Iberia y convertirlo en eventos normalizados de LaProgra, sin subir ni conservar el archivo original ni datos personales innecesarios.

Iberia no proporciona un nuevo CSV cuando modifica o cancela posteriormente un vuelo ya publicado. Esos cambios se reflejan manualmente en LaProgra por el propietario del evento.

## 2. Privacidad

- Todo el análisis del CSV se realiza en el navegador.
- Solo se utilizan estas columnas:
  - `Asunto`
  - `Fecha de comienzo`
  - `Comienzo`
  - `Fecha de finalización`
  - `Finalización`
- El resto de columnas puede contener datos personales y debe descartarse inmediatamente. No se incluirá en eventos normalizados, previsualizaciones técnicas, telemetría, logs, informes de error ni peticiones de red.
- La columna `Todo el día` no se utiliza, aunque exista en el CSV. Se descarta como cualquier otra columna no autorizada y no altera la semántica temporal del evento.
- Tras el análisis se descartarán el archivo original, las filas originales y cualquier buffer que contenga columnas no autorizadas.
- Solo se enviarán al servidor eventos normalizados y metadatos técnicos mínimos.

## 3. Entrada admitida

- Archivo CSV mensual exportado por Iberia.
- Tamaño máximo inicial: 10 MB, configurable.
- Admitir UTF-8 y detectar de forma segura otras codificaciones habituales del exportador, incluida Windows-1252.
- Detectar el delimitador entre las variantes explícitamente admitidas.
- El MIME es orientativo y no sustituye la validación del contenido.
- Rechazar archivos vacíos, binarios, excesivamente grandes, con estructura desconocida o con filas desproporcionadas.
- Las cinco cabeceras requeridas deben existir exactamente. Pueden existir columnas adicionales, pero se ignorarán.

## 4. Zona horaria y fechas

- En el formato observado, las fechas usan `dd/MM/yyyy` y las horas `HH:mm`.
- Los valores están expresados en hora local de Madrid.
- La zona horaria de origen es `Europe/Madrid`; no se utilizará un offset fijo.
- Para cada fila se combinarán:
  - `Fecha de comienzo` + `Comienzo`
  - `Fecha de finalización` + `Finalización`
- Se conservará `Europe/Madrid` como zona horaria original y se almacenarán los instantes normalizados en UTC.
- El final debe ser posterior al inicio.
- Una hora local inexistente o ambigua por un cambio DST que no pueda resolverse con los datos disponibles se marcará como ambigua; no se elegirá silenciosamente un offset.

## 5. Flujo de procesamiento

1. Validar nombre, tamaño, MIME orientativo y firma de contenido.
2. Detectar una codificación y un delimitador admitidos.
3. Leer la cabecera y comprobar las cinco columnas requeridas.
4. Proyectar inmediatamente cada fila a las cinco columnas autorizadas y descartar el resto.
5. Validar y normalizar fechas y horas con `Europe/Madrid`.
6. Clasificar el valor de `Asunto` siguiendo el orden indicado en este documento.
7. Crear eventos normalizados o incidencias de importación.
8. Mostrar una previsualización con filas aceptadas, rechazadas y ambiguas.
9. Enviar al servidor únicamente los eventos confirmados por el usuario.
10. Ejecutar un upsert idempotente.
11. Descartar el archivo y los datos temporales locales.

## 6. Clasificación del campo Asunto

Aplicar este orden y detenerse en la primera coincidencia:

1. Coincidencia exacta con `iberia_codes.csv`.
2. Fila de firma.
3. Vuelo y ruta.
4. Actividad desconocida.

### 6.1 Coincidencia exacta con iberia_codes.csv

- Comparar el contenido completo y exacto de `Asunto` con la columna `ID` de `iberia_codes.csv`.
- No comparar únicamente el código anterior al guion.
- No realizar coincidencias parciales, por prefijo, por similitud ni aproximadas.
- Los aparentes errores ortográficos, diferencias de espaciado y variantes históricas del catálogo son literales y no deben corregirse ni fusionarse.
- Si existe coincidencia exacta:
  - `slab` será el valor de `SLAB`.
  - `description` será el valor de `DESCRIPTION`.
- Las descripciones del catálogo son literales y no se traducen.

### 6.2 Fila de firma

Formato observado: `Firma 13:55 Pairing 3010`.

- La hora de firma se obtiene de `Comienzo`; el texto de `Asunto` se usa como comprobación cruzada.
- El número posterior a `Pairing` identifica la línea cuando esté presente.
- La firma se asocia al primer vuelo inmediatamente posterior cuando:
  - el final de la firma coincide exactamente con el inicio del vuelo; y
  - la fila posterior se reconoce como vuelo.
- La firma no crea evento ni slab independiente. Sirve exclusivamente para enriquecer el horario del primer vuelo del pairing.
- Español: `12:00 - 13:00 (firma 11:00)`.
- Inglés: `12:00 - 13:00 (report 11:00)`.
- La firma no sustituye el inicio del vuelo y no se añade a los vuelos siguientes del pairing.
- Si no puede asociarse inequívocamente al vuelo posterior, se marca como ambigua y no se vincula por aproximación.

### 6.3 Vuelo y ruta

Formato observado: `IB415  MAD1255-BCN1415 / 32A A+`.

Extraer como mínimo:

- número de vuelo;
- origen IATA;
- destino IATA;

Reglas:

- La fecha y hora de inicio se obtienen exclusivamente de `Fecha de comienzo` + `Comienzo`.
- La fecha y hora de finalización se obtienen exclusivamente de `Fecha de finalización` + `Finalización`.
- Las horas incluidas dentro de `Asunto` no se extraen, almacenan, convierten, comparan ni validan. Solo pueden formar parte de la estructura textual necesaria para reconocer el formato y localizar el número de vuelo, el origen y el destino.
- Consultar `airports.csv` para obtener las ciudades de origen y destino.
- La descripción incluirá las ciudades con el formato `Madrid - Barcelona` cuando ambos IATA estén presentes.
- Si un aeropuerto no existe en el catálogo, conservar el IATA y generar una advertencia no bloqueante.
- El slab del vuelo será la ruta IATA, por ejemplo `MAD-BCN`.

#### 6.3.1 Vuelo operado y vuelo en situación

La clasificación se determina por el prefijo del número de vuelo:

- `IB`: vuelo operado por el tripulante para Iberia.
- `VS`: código interno de Iberia que indica un vuelo en situación o deadhead realizado en Iberia. `VS` no se busca en `airlines.csv` ni en `iberia_codes.csv`. Para presentación se sustituye por `IB`, conservando la parte numérica.
- Cualquier otro prefijo IATA: vuelo en situación o deadhead realizado en otra aerolínea. Resolver el nombre mediante búsqueda por `IATA` en `airlines.csv`.
- Si la búsqueda por IATA devuelve varias filas, utilizar siempre la primera coincidencia según el orden original de `airlines.csv`.

Descripción del deadhead:

- Español: `Vuelo en situación {aerolínea} {número de vuelo}`.
- Inglés: `Deadhead flight on {aerolínea} {número de vuelo}`.

Ejemplos:

- `VS2344` se presenta como `Vuelo en situación Iberia IB2344` / `Deadhead flight on Iberia IB2344`.
- `FR2344` se presenta como `Vuelo en situación Ryanair FR2344` / `Deadhead flight on Ryanair FR2344`.

Si el prefijo no existe en `airlines.csv`, conservar el código y el número, usar el código como nombre provisional de aerolínea y generar una advertencia no bloqueante. La lógica debe admitir futuras aerolíneas sin una lista cerrada.

### 6.4 Actividad desconocida

Si el `Asunto` no coincide con el catálogo y no es una firma ni un vuelo:

- slab español: `N/D`;
- slab inglés: `N/A`;
- descripción: valor completo de `Asunto`, sin corregirlo ni traducirlo;
- advertencia en la previsualización, sin invalidar necesariamente el archivo.

## 7. Identidad e idempotencia

- Cada evento tendrá un `source_uid` estable.
- Si la fuente no proporciona un identificador fiable, se generará una huella determinista con compañía, fecha operativa, código o tipo, origen, destino, inicio y fin normalizados.
- La huella sirve para reconocer una reimportación idéntica y evitar duplicados; no se usará para deducir actualizaciones posteriores proporcionadas por Iberia, porque Iberia no facilita un CSV actualizado cuando cambia o cancela un vuelo.
- Importar dos veces el mismo archivo o la misma programación no debe duplicar eventos.

## 8. Cambios posteriores a la importación

### 8.1 Cambio de horario u otros datos

- Si Iberia cambia posteriormente el horario u otro dato del evento, el propietario lo edita manualmente en LaProgra.
- Las modificaciones se almacenan por campo en `event_overrides`.
- El registro de origen importado se conserva sin mutarlo.
- El evento efectivo que se muestra en calendario, amistades y PDF resulta de aplicar los overrides sobre el origen.

### 8.2 Cancelación

- Si Iberia cancela posteriormente un evento, el propietario lo elimina manualmente en LaProgra.
- La eliminación se guarda como tombstone.
- El evento deja de aparecer en calendario, vistas compartidas, comparaciones y PDF.

### 8.3 Reimportación accidental

Si el usuario vuelve a importar el mismo CSV o la misma programación:

1. no duplicar eventos;
2. no sobrescribir overrides;
3. no reactivar tombstones;
4. no restaurar los valores originales sobre los valores editados;
5. no intentar emparejar por proximidad temporal un evento diferente.

No se intentará deducir automáticamente que un evento con horario, ruta o número de vuelo distinto es una actualización de otro evento previamente importado.

## 9. Previsualización y errores

Mostrar:

- filas aceptadas;
- filas rechazadas;
- filas ambiguas;
- asuntos desconocidos;
- errores de fecha y hora;
- aeropuertos o aerolíneas desconocidos.

Los mensajes serán accionables y no incluirán columnas descartadas ni datos personales. Se podrá descargar un informe saneado.

## 10. Criterios de aceptación

- El CSV original nunca sale del navegador.
- Solo se usan las cinco columnas autorizadas.
- Las columnas adicionales no aparecen en red, logs o errores.
- La coincidencia con `iberia_codes.csv` usa el `Asunto` completo y es exacta.
- Una firma válida enriquece el primer vuelo contiguo y no crea un evento independiente.
- `IB` es operado; `VS` es deadhead en Iberia; cualquier otro prefijo es deadhead y resuelve la aerolínea mediante `airlines.csv`.
- Importar dos veces el mismo archivo no duplica eventos.
- Un cambio manual sobrevive a una reimportación idéntica.
- Una eliminación manual sobrevive a una reimportación idéntica.
- No se intenta detectar actualizaciones posteriores de Iberia mediante tolerancias horarias.
- Las fechas alrededor de DST se procesan de forma determinista.
- Los horarios incluidos en `Asunto` no se usan para determinar, comprobar ni enriquecer los horarios del evento.
- Una fila inválida no invalida necesariamente todo el archivo.