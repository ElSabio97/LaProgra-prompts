# Importador Webcal de Swiftair

## 1. Objetivo

Sincronizar de forma segura, automática e idempotente la programación de Swiftair desde un Webcal privado generado mediante eCrew y un calendario dedicado.

## 2. Fuente, seguridad y almacenamiento

- Aceptar `webcal://` y `https://`; convertir internamente cuando proceda.
- Validar esquema, longitud, host, puerto, destino resuelto y redirecciones.
- Bloquear destinos privados, loopback, link-local, metadata cloud y redirecciones inseguras para mitigar SSRF.
- Limitar tamaño, tiempo y número de redirecciones.
- Guardar la URL solo en almacenamiento privado accesible por el servicio.
- No devolverla al navegador después del alta ni incluirla en logs, analítica, trazas o errores.
- Las amistades nunca pueden verla.
- RLS es obligatoria, pero no sustituye el aislamiento del secreto en servidor.

## 3. Primera importación y sincronización

- Primera importación en servidor durante onboarding: mes en curso y meses posteriores.
- Conservar históricos aunque eCrew deje de publicarlos.
- Cron global de Supabase una vez por minuto, sin solicitar todas las fuentes cada minuto.
- Seleccionar lotes de fuentes vencidas con control de concurrencia, leasing, timeout, backoff exponencial, jitter y circuit breaker.
- Usar `ETag` y `Last-Modified` cuando estén disponibles.
- Objetivo: cambios visibles normalmente en menos de cinco minutos.
- Un fallo externo o resultado parcial no borra datos existentes.

## 4. Parseo iCalendar

Procesar `UID`, `DTSTART`, `DTEND`, `SUMMARY`, `LOCATION`, `DESCRIPTION`, `SEQUENCE`, `LAST-MODIFIED` y `DTSTAMP` cuando existan.

- Conservar zona original y normalizar instantes a UTC.
- Admitir `TZID`, UTC y `VALUE=DATE` con final exclusivo de iCalendar.
- Admitir medianoche, varios días y recurrencias en ventana acotada.
- Decodificar escapes y líneas plegadas.
- Usar `UID` como `source_uid`; si falta, crear huella determinista.

## 5. Hora de firma

- Extraer `Reporting time : HHMM` de `DESCRIPTION` cuando esté presente.
- La hora está expresada en UTC.
- Guardarla separada del inicio del evento.
- Su ausencia no invalida el evento.

## 6. Clasificación de SUMMARY

Aplicar este orden:

1. Coincidencia exacta con `ID` de `swiftair_codes.csv`.
2. Transporte terrestre `GRD`.
3. Número de vuelo y ruta.
4. Actividad desconocida.

La columna `ID` del catálogo debe ser única. Las pruebas o el despliegue deben fallar ante duplicados.

### 6.1 Actividad de catálogo

- Slab: código identificado.
- Descripción: `DESCRIPTION` literal.
- Códigos y descripciones no se traducen.

### 6.2 Vuelos

Formatos equivalentes:

- `4646 HAJ-LGG`.
- `721P MAD-BCN`.
- `IB539 MAD-LIS`.
- `AEA1145 MAD-PMI`.

Reglas:

- Extraer ruta como dos IATA de tres letras.
- Tolerar únicamente variantes de separación expresamente probadas, como espacio o punto entre prefijo y números.
- Si el identificador contiene solo números, o números seguidos de letras, añadir `WT`: `4646` -> `WT4646`; `721P` -> `WT721P`.
- Si usa prefijo ICAO de tres letras, convertirlo a IATA mediante `airlines.csv` en dirección `ICAO -> IATA`.
- Conservar identificador original y número normalizado.

### 6.3 Operado y deadhead

- Prefijos operados inicialmente: `WT` y `QY`.
- Todo otro prefijo IATA se clasifica como vuelo en situación/deadhead.
- Para deadhead, consultar `airlines.csv` por IATA y obtener el nombre.
- Español: `Vuelo en situación {aerolínea} {número}`.
- Inglés: `Deadhead flight on {aerolínea} {número}`.
- Ejemplo: `FR2344` -> `Vuelo en situación Ryanair FR2344` / `Deadhead flight on Ryanair FR2344`.
- Si no existe el IATA, conservar código y número, usar el código como nombre provisional y generar advertencia no bloqueante.
- La lista de operados debe ser configuración versionada del módulo Swiftair y estar probada.

### 6.4 Transporte terrestre

- Prefijo `GRD`, por ejemplo `GRD1319 DUS-CGN`.
- No es vuelo operado ni deadhead aéreo.
- Español: `En coche`.
- Inglés: `Ground transport`.
- Conservar la ruta si puede extraerse.

### 6.5 Desconocido

- Slab: texto anterior al primer espacio.
- Descripción: texto posterior.
- Si no hay espacio, usar todo como slab y descripción vacía.
- Conservar `SUMMARY` original y generar advertencia no bloqueante.

## 7. Reconciliación

- Actualizar eventos futuros desde el instante actual.
- No borrar históricos.
- Si un evento futuro desaparece de una descarga válida y completa, eliminarlo.
- Si la descarga falla o es parcial, no eliminar por ausencia.
- La operación es idempotente.
- Los eventos importados de Swiftair no se editan ni eliminan manualmente.
- Las actividades manuales independientes sí están permitidas.
- Un UID modificado debe reconciliarse explícitamente y no producir duplicados silenciosos.

## 8. Criterios de aceptación

- La URL no es visible para clientes posteriores al alta ni amistades.
- SSRF y acceso cruzado están probados.
- Una sincronización repetida no duplica.
- Un fallo no borra datos.
- Se manejan DST, varios días, `VALUE=DATE`, cancelaciones y recurrencias acotadas.
- Los IDs de `swiftair_codes.csv` son únicos.
- `WT` y `QY` son operados; los demás son deadhead salvo `GRD`.
- Los deadhead resuelven la aerolínea mediante `airlines.csv` y generan la descripción bilingüe definida.