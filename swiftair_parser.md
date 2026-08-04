### Importador Webcal de Swiftair

#### 1. Objetivo

Sincronizar de forma segura, automática e idempotente la programación de Swiftair desde un Webcal privado generado mediante eCrew y un calendario dedicado.

#### 2. Fuente, seguridad y almacenamiento
- Aceptar `webcal://` y `https://`; convertir internamente cuando proceda.
- Validar esquema, longitud, host, puerto, destino resuelto y redirecciones.
- Bloquear destinos privados, loopback, link-local, metadata cloud y redirecciones inseguras para mitigar SSRF.
- Revalidar el destino después de cada resolución DNS y redirección, incluyendo IPv4 e IPv6.
- Limitar tamaño, tiempo y número de redirecciones.
- Guardar la URL solo en almacenamiento privado accesible por el servicio.
- La URL se envía desde el navegador únicamente durante el alta o sustitución de la fuente. Después no se devuelve al navegador ni se incluye en HTML, JavaScript, estado del cliente, respuestas API, logs, analítica, trazas o errores.
- Las amistades nunca pueden verla.
- RLS es obligatoria, pero no sustituye el aislamiento del secreto en servidor.

#### 3. Primera importación y sincronización
- Primera importación en servidor durante onboarding: eventos que se solapen con el mes natural en curso y el mes natural siguiente.
- Una primera sincronización válida pero sin eventos no completa el onboarding.
- Solo se almacenan eventos que se solapen con el mes natural anterior, el mes natural en curso y el mes natural siguiente, calculados en la zona horaria del perfil.
- Un evento se conserva completo si cualquier parte de su intervalo se solapa con la ventana.
- Cron global de Supabase una vez por minuto, sin solicitar todas las fuentes cada minuto.
- Seleccionar lotes de fuentes vencidas con control de concurrencia, leasing, timeout, backoff exponencial, jitter y circuit breaker.
- Usar `ETag` y `Last-Modified` cuando estén disponibles.
- Objetivo: cambios visibles normalmente en menos de cinco minutos.

Cuando se obtenga un cuerpo nuevo:
1. Calcular un hash del contenido completo sin registrar el contenido.
2. Si el hash coincide con el último contenido procesado, no modificar eventos.
3. Si el hash cambia, interpretar el contenido y reconciliar los eventos futuros.

Semántica del resultado de la obtención:
- Un cuerpo vacío representa un conjunto futuro vacío.
- Un iCalendar válido sin eventos representa un conjunto futuro vacío.
- Si el cuerpo contiene uno o más `VEVENT` completos interpretables, la fuente de verdad es el conjunto de esos eventos, aunque el documento esté parcial, truncado después de ellos o termine incorrectamente.
- Si se obtiene un cuerpo pero no contiene ningún `VEVENT` completo interpretable, representa un conjunto futuro vacío.
- Un error de conexión, timeout u obtención que no produzca cuerpo nuevo conserva temporalmente el estado anterior.

La obtención, interpretación, reconciliación y actualización del estado se harán de forma transaccional o con una estrategia equivalente que no deje un conjunto parcial visible.

#### 4. Parseo iCalendar

Procesar cuando existan:
- `UID`;
- `DTSTART`;
- `DTEND`;
- `SUMMARY`;
- `LOCATION`;
- `DESCRIPTION`;
- `SEQUENCE`;
- `LAST-MODIFIED`;
- `DTSTAMP`;
- `STATUS`;
- `RRULE`;
- `RDATE`;
- `EXDATE`;
- `RECURRENCE-ID`.

Reglas:
- Conservar zona original y normalizar instantes temporizados a UTC.
- Admitir `TZID`, UTC y `VALUE=DATE` con final exclusivo de iCalendar.
- Admitir medianoche, varios días y recurrencias dentro de la ventana acotada.
- Decodificar escapes y líneas plegadas.
- Usar `UID` como `source_uid`; si falta, crear huella determinista.
- Los límites temporales funcionales proceden de `DTSTART` y `DTEND`, no de las horas incluidas en `DESCRIPTION` o `LOCATION`.
- Un `VEVENT` con `STATUS:CANCELLED` no produce evento visible y se considera ausente del conjunto efectivo.
- Las recurrencias se expanden solo dentro de la ventana de almacenamiento. Se respetan `RDATE`, `EXDATE` y modificaciones individuales con `RECURRENCE-ID`.
- La identidad de una ocurrencia recurrente combinará la identidad de la serie con la fecha o instante de recurrencia de forma estable.

##### 4.1 Eventos de día completo y duración cero
- Los eventos `VALUE=DATE` conservan semántica de fecha civil y final exclusivo.
- Un evento Swiftair temporizado con `DTSTART` igual a `DTEND` se interpreta como evento de día completo correspondiente a la fecha civil de `DTSTART` en su `TZID` original.
- El evento normalizado guardará `is_all_day`, `start_date`, `end_date_exclusive` y zona original. Para una duración cero, `end_date_exclusive` será el día siguiente a `start_date`.
- Los eventos de día completo no se desplazan de fecha al cambiar la zona de visualización del usuario.
- Un evento temporizado puede solaparse visualmente con un evento de día completo al convertirse a otra zona. Ambos se muestran de forma independiente: el día completo en su banda de fecha civil y el temporizado en su hora convertida. No se recortan, fusionan ni desplazan entre sí.

#### 5. Hora de firma
- Extraer `Reporting time : HHMM` de `DESCRIPTION` cuando esté presente.
- La firma solo se conserva y muestra cuando el evento se clasifica como vuelo aéreo, sea operado o en situación.
- Si `DESCRIPTION` indica `All times in UTC`, interpretar la firma en UTC.
- Si indica `All times in Local Base (XXX)`, interpretar la firma en la zona IANA asociada al IATA `XXX` en `airports.csv`.
- Si el IATA de base no existe, no inventar una zona: generar advertencia no bloqueante y no mostrar firma hasta poder resolverla.
- Combinar la hora con la fecha correcta respecto del inicio del vuelo, contemplando que la firma pueda pertenecer al día civil anterior.
- Guardar la firma como instante normalizado y conservar la zona utilizada para interpretarla.
- La firma no sustituye el inicio del evento.
- Su ausencia no invalida el vuelo.
- No mostrar firma en actividades de catálogo, transporte terrestre, actividades desconocidas ni códigos descartados.

#### 6. Clasificación de `SUMMARY`

Aplicar este orden y detenerse en la primera regla aplicable:
1. Código descartado.
2. Actividad de catálogo.
3. Transporte terrestre `GRD`.
4. Número de vuelo y ruta.
5. Actividad desconocida.

La columna `ID` de `swiftair_codes.csv` debe ser única. Las pruebas o el despliegue deben fallar ante duplicados.

##### 6.1 Extracción y normalización del token
- Extraer el texto anterior al primer espacio de `SUMMARY`.
- Convertirlo a mayúsculas.
- Eliminar los caracteres de puntuación expresamente admitidos, incluidos los puntos finales, sin alterar letras ni números.
- Intentar primero una coincidencia exacta con el catálogo o lista correspondiente.
- Si no existe coincidencia exacta, buscar IDs que sean prefijo del token normalizado.
- Utilizar la coincidencia de mayor longitud.
- Ante varias coincidencias de la misma longitud, utilizar la primera fila según el orden original del catálogo o lista.

Ejemplos:
- `IM.` → `IM`.
- `N.` → `N`.
- `IMI` → `IMI`, no `IM`.
- `OLS1` → `OLS` si no existe `OLS1` y `OLS` es el prefijo válido más largo.

##### 6.2 Códigos descartados
- Mantener una lista versionada `ignored_summary_codes` propia del módulo Swiftair.
- La lista inicial incluye `CATB`.
- Aplicar a la lista descartada la misma normalización del token.
- Un evento descartado no genera evento ni advertencia de actividad desconocida.
- La lista se ampliará solo mediante cambio versionado y pruebas.

##### 6.3 Actividad de catálogo
Si el token normalizado coincide con un ID de `swiftair_codes.csv` mediante las reglas anteriores:
- `slab`: ID identificado.
- `description`: valor literal de la columna `DESCRIPTION` de `swiftair_codes.csv`.
- La ruta o texto restante de `SUMMARY` no impide la coincidencia.
- Los códigos y descripciones del catálogo no se traducen.
- El campo bruto `DESCRIPTION` del Webcal no se utiliza como descripción visible y nunca se comparte con amistades.

##### 6.4 Vuelos

Formatos equivalentes:
- `4646 HAJ-LGG`.
- `721P MAD-BCN`.
- `IB539 MAD-LIS`.
- `AEA1145 MAD-PMI`.

Reglas:
- Extraer ruta como dos IATA de tres letras.
- Tolerar únicamente variantes de separación expresamente probadas, como espacio o punto entre prefijo y números.
- Si el identificador contiene solo números, o números seguidos de letras, añadir `WT`: `4646` → `WT4646`; `721P` → `WT721P`.
- Si usa prefijo ICAO de tres letras, convertirlo a IATA mediante `airlines.csv` en dirección ICAO → IATA.
- Conservar identificador original y número normalizado.
- Ante varias coincidencias válidas en cualquier catálogo, utilizar la primera fila según el orden original del CSV.

##### 6.5 Operado y vuelo en situación
- Prefijos operados inicialmente: `WT` y `QY`.
- Todo otro prefijo IATA se clasifica como vuelo en situación.
- Para un vuelo en situación, consultar `airlines.csv` por IATA y obtener el nombre.
- Si hay varias filas, utilizar la primera en el orden original.
- Español: `Vuelo en situación {aerolínea} {número}`.
- Inglés: `Deadhead flight on {aerolínea} {número}`.
- Si no existe el IATA, conservar código y número, usar el código como nombre provisional y generar advertencia no bloqueante.
- La lista de operados debe ser configuración versionada del módulo Swiftair y estar probada.

##### 6.6 Transporte terrestre
- Prefijo `GRD`, por ejemplo `GRD1319 DUS-CGN`.
- No es vuelo operado ni vuelo en situación aéreo.
- Español: `En coche`.
- Inglés: `Ground transport`.
- Conservar la ruta si puede extraerse.

##### 6.7 Desconocido
- `slab`: token original anterior al primer espacio.
- `description`: texto posterior.
- Si no hay espacio, usar todo como slab y descripción vacía.
- Conservar `SUMMARY` original y generar advertencia no bloqueante.

#### 7. Reconciliación
- El hash se calcula sobre el cuerpo completo obtenido.
- Si el hash no cambia, no se escriben eventos.
- Si cambia, el conjunto nuevo sustituye todos los eventos Swiftair con `starts_at > now`.
- Para sustituir: eliminar los eventos futuros existentes y escribir todos los eventos futuros resultantes del nuevo contenido que entren en la ventana.
- No intentar emparejar eventos por proximidad ni depender de que el `UID` sea estable entre descargas.
- Los eventos con `starts_at <= now` no se eliminan ni actualizan por reconciliación, incluidos los eventos en curso.
- Un conjunto nuevo vacío elimina todos los eventos futuros de Swiftair.
- Un cuerpo parcial con `VEVENT` completos sustituye el futuro por los eventos completos interpretables.
- Un cuerpo recibido sin `VEVENT` completos interpretables sustituye el futuro por un conjunto vacío.
- Un fallo de conexión u obtención sin cuerpo nuevo conserva temporalmente el estado anterior.
- Solo se almacenan eventos que se solapen con el mes natural anterior, el actual y el siguiente.
- Un evento que se solape parcialmente con la ventana se conserva completo.
- Los eventos completamente fuera de la ventana se eliminan junto con sus datos dependientes.
- La operación es idempotente.
- Los eventos importados de Swiftair no se editan ni eliminan manualmente.
- Las actividades manuales independientes sí están permitidas y se rigen por su propia ventana.

#### 8. Criterios de aceptación
- La URL no es visible para clientes posteriores al alta ni amistades.
- SSRF, redirecciones, IPv4, IPv6, DNS rebinding y acceso cruzado están probados.
- Una sincronización con el mismo hash no escribe ni duplica eventos.
- Un hash distinto reemplaza todos los eventos con `starts_at > now` por el conjunto nuevo dentro de ventana.
- Un cuerpo vacío o un iCalendar sin eventos elimina todos los eventos futuros.
- Un cuerpo parcial utiliza únicamente sus `VEVENT` completos y reemplaza el futuro por ellos.
- Un cuerpo recibido sin `VEVENT` completos reemplaza el futuro por un conjunto vacío.
- Un fallo de obtención sin cuerpo nuevo conserva temporalmente el estado anterior.
- Los eventos con `starts_at <= now` no se eliminan ni actualizan.
- Se manejan DST, varios días, `VALUE=DATE`, cancelaciones y recurrencias acotadas.
- `STATUS:CANCELLED` no produce evento visible.
- Los eventos temporizados de duración cero se convierten en día completo de una fecha civil y no cambian de fecha al cambiar la zona de visualización.
- Un evento temporizado y uno de día completo pueden solaparse visualmente sin modificar sus datos.
- Se conservan completos los eventos que se solapen parcialmente con la ventana.
- Los IDs de `swiftair_codes.csv` son únicos.
- `IM.` se clasifica como `IM`, `N.` como `N` y `OLS1` como `OLS` mediante coincidencia exacta y después prefijo más largo.
- `IMI` se clasifica como `IMI`, no como `IM`.
- Los códigos descartados, incluido `CATB`, no generan eventos ni advertencias.
- La descripción visible de catálogo procede de `swiftair_codes.csv`, no del `DESCRIPTION` del Webcal.
- `WT` y `QY` son operados; los demás son vuelos en situación salvo `GRD`.
- Los vuelos en situación resuelven la aerolínea mediante `airlines.csv`.
- La firma se interpreta según `All times in UTC` o `All times in Local Base (XXX)` y solo aparece en vuelos.
- Ante varias coincidencias se utiliza la regla especializada y, en empate, la primera fila del catálogo según su orden original.
- Una primera sincronización válida pero vacía no completa onboarding.
