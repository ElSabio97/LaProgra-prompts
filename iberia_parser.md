### Importador CSV de Iberia

#### 1. Objetivo

Procesar en el navegador el CSV mensual exportado por Iberia y convertirlo en eventos normalizados de LaProgra, sin subir ni conservar el archivo original ni datos personales innecesarios.

Iberia no proporciona un nuevo CSV cuando modifica o cancela posteriormente un vuelo ya publicado. Esos cambios se reflejan manualmente en LaProgra por el propietario del evento.

#### 2. Privacidad
- Todo el análisis del CSV se realiza en el navegador.
- Solo se utilizan estas columnas:
  - `Asunto`;
  - `Fecha de comienzo`;
  - `Comienzo`;
  - `Fecha de finalización`;
  - `Finalización`.
- La columna `Todo el día` no se utiliza, aunque exista. Se descarta como cualquier otra columna no autorizada y no altera la semántica temporal.
- El resto de columnas se descarta y no se incluye en eventos normalizados, previsualizaciones, telemetría, logs, errores ni peticiones de red.
- Tras el análisis se eliminan las referencias al archivo original, filas originales y buffers temporales que no sean necesarios.
- El archivo original nunca se sube a un servidor ni se almacena en una base de datos.
- Solo se envían al servidor eventos normalizados confirmados por el usuario y metadatos técnicos mínimos.
- No se genera informe descargable de las filas originales o descartadas.

#### 3. Entrada admitida
- Archivo CSV mensual exportado por Iberia.
- Tamaño máximo inicial: 10 MB, configurable.
- Admitir UTF-8 y detectar de forma segura otras codificaciones habituales, incluida Windows-1252.
- Detectar el delimitador entre variantes expresamente admitidas.
- El MIME es orientativo y no sustituye la validación del contenido.
- Rechazar archivos vacíos, binarios, excesivamente grandes, con estructura desconocida o filas desproporcionadas.
- Las cinco cabeceras requeridas deben existir exactamente. Pueden existir otras columnas, pero se ignoran.
- Usar un parser CSV conforme con campos entrecomillados, comas dentro de campos, comillas escapadas y campos multilínea. No dividir el archivo por líneas o comas de forma manual.

#### 4. Zona horaria y fechas
- En el formato observado, las fechas usan `dd/MM/yyyy` y las horas `HH:mm`.
- Los valores están expresados en hora local de Madrid.
- La zona de origen es `Europe/Madrid`; no se utiliza un offset fijo.
- Para cada fila se combinan:
  - `Fecha de comienzo` + `Comienzo`;
  - `Fecha de finalización` + `Finalización`.
- Se conserva `Europe/Madrid` como zona original y se almacenan instantes normalizados en UTC.
- El final debe ser posterior al inicio.
- Una hora local inexistente o repetida por DST que no pueda resolverse se clasifica determinísticamente como ambigua; no se elige silenciosamente un offset.
- `Todo el día` y las horas contenidas en `Asunto` no intervienen en la determinación temporal.

#### 5. Flujo de procesamiento
1. Validar nombre, tamaño, MIME orientativo y firma de contenido.
2. Detectar codificación y delimitador admitidos.
3. Leer cabecera y comprobar las cinco columnas requeridas.
4. Proyectar cada fila a las cinco columnas autorizadas y descartar el resto.
5. Validar y normalizar fechas y horas con `Europe/Madrid`.
6. Clasificar `Asunto` siguiendo el orden de este documento.
7. Aplicar la ventana de almacenamiento para la previsualización.
8. Crear eventos normalizados o incidencias locales.
9. Mostrar previsualización con filas aceptadas, rechazadas, ambiguas y omitidas.
10. Mantener la previsualización hasta que el usuario la acepte o importe otro archivo.
11. Al confirmar, recalcular la ventana vigente en la zona del perfil.
12. Enviar solo los eventos confirmados que sigan dentro de la ventana.
13. Ejecutar un upsert idempotente.
14. Descartar el archivo y datos temporales locales.

#### 6. Clasificación de `Asunto`

Aplicar este orden y detenerse en la primera coincidencia:
1. Coincidencia exacta con `iberia_codes.csv`.
2. Fila de firma.
3. Vuelo y ruta.
4. Actividad desconocida.

##### 6.1 Coincidencia exacta con `iberia_codes.csv`
- Comparar el contenido completo y exacto de `Asunto` con la columna `ID`.
- No comparar únicamente el código anterior al guion.
- No realizar coincidencias parciales, por prefijo, similitud ni aproximación.
- Los errores ortográficos, espaciado y variantes históricas son literales y no se corrigen.
- Si coincide:
  - `slab` será `SLAB`;
  - `description` será `DESCRIPTION`.
- Las descripciones del catálogo son literales y no se traducen.
- Si hay varias coincidencias, se utiliza la primera fila según el orden original.

##### 6.2 Fila de firma

Formato observado: `Firma 13:55 Pairing 3010`.
- La hora de firma se obtiene de `Comienzo`; el texto de `Asunto` solo reconoce la fila.
- El número posterior a `Pairing` identifica la línea cuando esté presente.
- La firma se asocia exclusivamente a la fila física inmediatamente posterior cuando:
  - el final de la firma coincide exactamente con el inicio del vuelo;
  - la fila inmediatamente posterior se reconoce como vuelo.
- No se saltan filas inválidas, omitidas o desconocidas para buscar otro vuelo.
- La firma no crea evento ni slab independiente. Solo enriquece el primer vuelo contiguo.
- Español: `12:00 - 13:00 (firma 11:00)`.
- Inglés: `12:00 - 13:00 (report 11:00)`.
- La firma no sustituye el inicio del vuelo ni se añade a vuelos siguientes.
- Si no puede asociarse inequívocamente, se marca como ambigua y no se vincula.

##### 6.3 Vuelo y ruta

Formato observado: `IB415 MAD1255-BCN1415 / 32A A+`.

Extraer únicamente:
- número de vuelo;
- origen IATA;
- destino IATA.

Reglas:
- Inicio: exclusivamente `Fecha de comienzo` + `Comienzo`.
- Fin: exclusivamente `Fecha de finalización` + `Finalización`.
- Las horas incluidas en `Asunto` no se extraen, almacenan, convierten, comparan ni validan. Solo forman parte de la estructura textual para localizar vuelo, origen y destino.
- El texto adicional posterior a la ruta se descarta y no se muestra ni almacena.
- Consultar `airports.csv` para ciudades.
- Mostrar ciudades como `Madrid - Barcelona` cuando ambos IATA estén presentes.
- Si un aeropuerto no existe, conservar IATA y generar advertencia no bloqueante.
- El slab será la ruta IATA, por ejemplo `MAD-BCN`.

###### 6.3.1 Vuelo operado y vuelo en situación
- `IB`: vuelo operado por el tripulante para Iberia.
- `VS`: código interno de Iberia para vuelo en situación realizado en Iberia. No se busca en `airlines.csv` ni `iberia_codes.csv`; para presentación se sustituye por `IB`, conservando la parte numérica.
- Cualquier otro prefijo IATA: vuelo en situación de otra aerolínea, resuelto por IATA en `airlines.csv`.
- Si hay varias filas, usar la primera según el orden original.
- Español: `Vuelo en situación {aerolínea} {número de vuelo}`.
- Inglés: `Deadhead flight on {aerolínea} {número de vuelo}`.
- Mostrar además las ciudades de origen y destino.
- Si el prefijo no existe, conservar código y número, usar el código como nombre provisional y generar advertencia no bloqueante.
- La lógica admite futuras aerolíneas sin lista cerrada.

##### 6.4 Actividad desconocida
- Slab español: `N/D`.
- Slab inglés: `N/A`.
- Descripción: valor completo de `Asunto`, sin corregir ni traducir.
- Mostrar advertencia en la previsualización sin invalidar necesariamente el archivo.

#### 7. Identidad e idempotencia
- Cada evento tendrá `source_uid` estable.
- Si la fuente no proporciona identificador fiable, generar una huella determinista con compañía, fecha operativa, código o tipo, origen, destino, inicio y fin normalizados.
- Dos filas con huella idéntica se consideran duplicadas y generan un único evento.
- La huella reconoce una reimportación idéntica; no se usa para deducir actualizaciones posteriores.
- Importar dos veces el mismo archivo o programación no duplica eventos.

#### 8. Cambios posteriores

##### 8.1 Cambio de horario u otros datos
- El propietario edita manualmente los cambios posteriores.
- Las modificaciones se almacenan por campo en `event_overrides`.
- Campos editables:
  - fecha y hora de comienzo;
  - fecha y hora de finalización;
  - hora de firma;
  - número de pairing;
  - número de vuelo;
  - origen;
  - destino;
  - descripción.
- El origen importado se conserva sin mutarlo.
- Al cambiar origen o destino, recalcular slab y ciudades con `airports.csv`.
- Si la descripción tiene override, conservarla y no sobrescribirla al cambiar ruta.
- El evento efectivo mostrado resulta de aplicar overrides sobre el origen.

##### 8.2 Cancelación
- El propietario elimina manualmente el evento.
- La eliminación se guarda como tombstone.
- Deja de aparecer en calendario, vistas compartidas, comparaciones y PDF.

##### 8.3 Reimportación accidental
- No duplicar eventos.
- No sobrescribir overrides.
- No reactivar tombstones.
- No restaurar valores originales sobre valores editados.
- No emparejar por proximidad temporal un evento diferente.
- No deducir que un evento con horario, ruta o número distinto actualiza otro anterior.

#### 9. Previsualización y errores

Mostrar localmente:
- filas aceptadas;
- filas rechazadas;
- filas ambiguas;
- filas omitidas por ventana;
- asuntos desconocidos;
- errores de fecha y hora;
- aeropuertos o aerolíneas desconocidos;
- duplicados detectados.

Los mensajes serán accionables y no incluirán columnas descartadas ni datos personales. No habrá informe descargable.

#### 10. Ventana de almacenamiento
- Solo se almacenan eventos que se solapen con el mes anterior, actual o siguiente, calculados en la zona del perfil.
- Un evento que empiece fuera y termine dentro, o viceversa, se conserva completo.
- Se omite solo cuando queda completamente fuera.
- Una importación con todos los eventos fuera se rechaza antes de confirmar.
- Si mezcla filas dentro y fuera, se aceptan solo las que se solapan y las demás se muestran como omitidas.
- Al confirmar, recalcular la ventana vigente y volver a validar los eventos.
- La limpieza automática elimina también overrides y tombstones dependientes de eventos fuera de ventana.

#### 11. Criterios de aceptación
- El CSV original nunca sale del navegador ni se almacena en servidor o base de datos.
- Solo se usan las cinco columnas autorizadas.
- `Todo el día` y columnas adicionales no aparecen en red, logs ni errores.
- Un campo descartado con comas y saltos de línea no rompe el parseo ni la alineación de filas.
- La coincidencia usa `Asunto` completo y exacto.
- Una firma válida enriquece solo el vuelo de la fila física inmediatamente posterior.
- No se saltan filas para asociar una firma.
- `IB` es operado; `VS` es vuelo en situación en Iberia; otros prefijos resuelven aerolínea.
- Ante varias aerolíneas, se escoge la primera fila del catálogo.
- Las horas de `Asunto` no determinan, comprueban ni enriquecen el horario.
- El texto adicional del vuelo se descarta.
- `Todo el día=Verdadero` no transforma el intervalo ni corrige `00:01` o `23:59`.
- Dos huellas idénticas se consideran duplicado y generan un único evento.
- Una reimportación no duplica, no sobrescribe overrides y no reactiva tombstones.
- Solo son editables los campos definidos y un cambio de ruta recalcula slab y ciudades.
- No se detectan actualizaciones mediante tolerancias horarias.
- Las fechas DST válidas se convierten determinísticamente; las inexistentes o repetidas sin información suficiente se clasifican como ambiguas.
- Una fila inválida no invalida necesariamente todo el archivo.
- Se conservan completos los eventos con solapamiento parcial.
- La previsualización permanece hasta aceptar o reimportar; al aceptar se recalcula la ventana vigente.
- No se ofrece informe descargable.
