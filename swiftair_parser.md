### Importador Webcal de Swiftair

#### 1. Objetivo

Sincronizar de forma segura, automática e idempotente la programación de Swiftair desde un Webcal privado generado mediante eCrew y un calendario dedicado.

#### 2. Fuente, seguridad y almacenamiento
- El usuario aporta la URL Webcal desde el navegador, pero la descarga del calendario se realiza siempre en servidor.
- `progra_swiftair.txt` es solo un fixture de ejemplo del cuerpo descargado y nunca se utiliza como fuente productiva.
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
- Primera importación durante onboarding: capturar una sola vez `onboarding_cutoff` e importar solo eventos aún vigentes y dentro de la ventana ordinaria. Temporizados con `ends_at > onboarding_cutoff`; días completos con `end_date_exclusive` posterior a la fecha civil del corte en su zona original. Se incluyen eventos en curso y se excluyen los terminados.
- La primera importación completa onboarding únicamente cuando la transacción confirma y almacena al menos un evento efectivo y visible: individualmente válido, no descartado ni cancelado, vigente en `onboarding_cutoff` y solapado con la ventana. Una actividad desconocida temporalmente válida cuenta. Terminados, fuera de ventana, descartados, cancelados, inválidos, incompletos o no confirmados no cuentan.
- Solo se almacenan eventos que se solapen con el mes natural anterior, el mes natural en curso y el mes natural siguiente, calculados en la zona horaria del perfil.
- Un evento se conserva completo si cualquier parte de su intervalo se solapa con la ventana.
- Cron global de Supabase una vez por minuto, sin solicitar todas las fuentes cada minuto.
- Seleccionar lotes de fuentes vencidas con control de concurrencia, leasing, timeout, backoff exponencial, jitter y circuit breaker.
- Usar `ETag` y `Last-Modified` cuando estén disponibles.
- Objetivo: cambios visibles normalmente en menos de cinco minutos.

Cuando se obtenga un cuerpo nuevo:
- Calcular un hash del contenido completo sin registrarlo ni conservarlo.
- Solo omitir el reproceso cuando coincidan hash y ventana efectiva ya procesada.
- Si cambia la ventana, obtener un cuerpo completo y procesarlo aunque el hash sea igual.  
Semántica del resultado:
- La autoridad depende de la admisibilidad del cuerpo, no del estado HTTP. Cualquier respuesta HTTP con cuerpo admisible puede ser autoritativa aunque no sea `2xx`.
- `204 No Content` y un cuerpo de cero octetos correctamente obtenido son vacíos autoritativos.
- Un cuerpo reconocible como iCalendar es autoritativo aunque esté truncado por el origen.
- Solo se conservan `VEVENT` completos e individualmente válidos; los inválidos o incompletos se omiten. Si no queda ninguno, el futuro autoritativo es vacío.
- El estado HTTP puede afectar a observabilidad y reintentos, pero no anula la autoridad de un cuerpo admisible.
- HTML o contenido claramente ajeno a iCalendar no es autoritativo.
- Error de conexión, timeout, rechazo de seguridad, descarga interrumpida, descompresión fallida, límite excedido o respuesta sin cuerpo utilizable conserva temporalmente el estado anterior.
La obtención, interpretación, reconciliación y actualización del estado se harán de forma transaccional o con una estrategia equivalente que no deje un conjunto parcial visible.

#### 3.1 Frontera verificable de cuerpo admisible
La admisibilidad se decide antes de validar `VEVENT` y no depende de `Content-Type`, extensión, estado HTTP, `PRODID` ni de conservar el cuerpo bruto.

Un cuerpo es admisible exactamente en uno de estos casos:
1. **Vacío correcto:** después de completar transporte, redirecciones y decodificación de contenido, contiene cero octetos; `204 No Content` pertenece a este caso. Un cuerpo con solo espacios o saltos de línea no es vacío correcto.
2. **iCalendar reconocible:** la descarga termina limpiamente dentro de los límites; los octetos son UTF-8 válido, con BOM UTF-8 opcional; no contienen NUL ni controles prohibidos salvo tabulador, CR y LF; tras aceptar CRLF o LF y desplegar continuaciones, la primera línea lógica es exactamente `BEGIN:VCALENDAR`; y antes del primer componente o del fin del cuerpo aparece exactamente una línea `VERSION:2.0`. No se permite texto, HTML, JSON, XML ni otra envoltura antes de `BEGIN:VCALENDAR`.

Cumplida esa frontera, la ausencia de `END:VCALENDAR`, componentes incompletos o errores posteriores del contenedor no retira autoridad: se procesan solo los `VEVENT` cerrados por `END:VEVENT` e individualmente válidos. Un `BEGIN:VEVENT` aislado sin la envoltura y versión exigidas no hace reconocible el cuerpo. MIME y cabeceras son señales diagnósticas, nunca sustituyen esta prueba.

Se distingue un truncamiento recibido limpiamente desde el origen de una adquisición local incompleta. EOF limpio después de superar la frontera puede ser autoritativo; timeout, corte de conexión, fallo de descompresión, cancelación o límite local excedido significa que no se obtuvo cuerpo utilizable y conserva el futuro anterior. Los límites se aplican en streaming antes y después de descompresión.

#### 4. Parseo iCalendar

Procesar cuando existan:
- `UID`;
- `DTSTART`;
- `DTEND`;
- `DURATION`;
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
- La identidad de una ocurrencia recurrente combinará la identidad de la serie con la fecha o instante de recurrencia de forma estable. La protección y reconciliación operan por ocurrencia expandida, no sobre toda la serie.
- Si el cuerpo contiene duplicados de una misma identidad, conservar el más nuevo: mayor `SEQUENCE`; en empate, mayor `LAST-MODIFIED`; después mayor `DTSTAMP`; y, si todavía empatan o faltan esos valores, la última aparición física. Para una excepción recurrente, la identidad combina `UID` y `RECURRENCE-ID`.
- Cada evento conserva las versiones del adaptador, catálogo y configuración usadas al clasificarlo. Los cambios posteriores de catálogo no reclasifican eventos ya almacenados; solo afectan a eventos procesados después del cambio.

##### 4.1 `DURATION`
- `DTEND` y `DURATION` son mutuamente excluyentes dentro de un mismo `VEVENT`; si aparecen ambos, el elemento es inválido.
- `DURATION` debe ajustarse a la duración iCalendar y ser estrictamente positiva. Duraciones negativas son inválidas. Duración cero solo puede seguir la conversión excepcional `Full day` definida en §4.2; sin esa señal, es inválida.
- Con `DTSTART;VALUE=DATE`, solo se admiten semanas o días enteros, sin parte horaria. El resultado es un día completo con `end_date_exclusive = start_date + duración civil`.
- Con `DTSTART` DATE-TIME, semanas y días se suman primero como unidades civiles en la zona interpretativa de `DTSTART`, conservando la hora local a través de DST; después se suman horas, minutos y segundos como tiempo exacto. En UTC toda la suma es exacta. El extremo derivado debe ser posterior al inicio.
- Un inicio flotante con `DURATION` exige la misma zona general explícita y válida que cualquier otro flotante. El extremo derivado conserva la zona del inicio.
- La duración efectiva de una serie se hereda del maestro. Una excepción con `RECURRENCE-ID` puede sustituirla mediante su propio `DTEND` o `DURATION`; si no aporta ninguno, hereda la duración efectiva del maestro. Las combinaciones de clase DATE/DATE-TIME deben seguir siendo compatibles.
- `DURATION` participa en normalización, expansión, identidad temporal, ventana y validación, pero la huella usa el final normalizado derivado y no el texto bruto de la propiedad.

##### 4.2 Eventos de día completo y duración cero
- Los eventos `VALUE=DATE` conservan semántica de fecha civil y final exclusivo.
- Un evento temporizado con `DTSTART = DTEND` solo se convierte en día completo si una señal inequívoca, inicialmente `Full day` en `DESCRIPTION`, identifica esa semántica. `DESCRIPTION` solo se usa para detectarla y nunca como descripción visible o compartible. Sin esa señal, el evento es inválido.
- El evento normalizado guardará `is_all_day`, `start_date`, `end_date_exclusive` y zona original. Para una duración cero, `end_date_exclusive` será el día siguiente a `start_date`.
- Para convertir una duración cero con `Full day`, la fecha civil canónica procede de la representación explícita de `DTSTART`: con `TZID`, usar ese TZID; con sufijo `Z`, usar UTC como zona civil y su fecha UTC como `start_date`; si es flotante, exigir la zona general explícita y válida admitida por el adaptador. No deducirla del perfil, base, ruta, `LOCATION`, servidor ni zona de visualización.
- Los eventos de día completo no se desplazan de fecha al cambiar la zona de visualización del usuario.
- Para decidir si un día completo queda protegido, comparar `start_date` con la fecha civil del `reconciliation_cutoff` en su zona original. Si es igual o anterior, no se incorpora, actualiza ni elimina en esa reconciliación.
- Un evento temporizado puede solaparse visualmente con un evento de día completo al convertirse a otra zona. Ambos se muestran de forma independiente: el día completo en su banda de fecha civil y el temporizado en su hora convertida. No se recortan, fusionan ni desplazan entre sí.

#### 5. Hora de firma
- Extraer `Reporting time : HHMM` de `DESCRIPTION` cuando esté presente.
- La firma solo se conserva y muestra cuando el evento se clasifica como vuelo aéreo, sea operado o en situación.
- Si `DESCRIPTION` indica `All times in UTC`, interpretar la firma en UTC.
- Si indica `All times in Local Base (XXX)`, interpretar la firma en la zona IANA asociada al IATA `XXX` en `airports.csv`.
- Si el IATA de base no existe, no inventar una zona: generar advertencia no bloqueante y no mostrar firma hasta poder resolverla.
- Combinar la hora con la fecha correcta respecto del inicio del vuelo, contemplando que la firma pueda pertenecer al día civil anterior.
- Descartar candidatos posteriores a la salida. Si existen dos candidatos anteriores válidos, elegir el más alejado temporalmente de la salida, es decir, el más antiguo.
- Guardar la firma como instante normalizado y conservar la zona utilizada para interpretarla.
- La firma no sustituye el inicio del evento.
- Su ausencia no invalida el vuelo.
- No mostrar firma en actividades de catálogo, transporte terrestre, actividades desconocidas ni códigos descartados.

#### 6. Clasificación de `SUMMARY`

Aplicar este orden y detenerse en la primera regla aplicable:
- Código descartado por coincidencia exacta.
- Actividad de catálogo por coincidencia exacta.
- Transporte terrestre `GRD`.
- Número de vuelo y ruta.
- Actividad de catálogo por prefijo más largo.
- Actividad desconocida.
Las coincidencias exactas prevalecen primero; después una forma válida de transporte o vuelo prevalece sobre coincidencias parciales. El orden único es: descartado exacto, catálogo exacto, `GRD`, vuelo, catálogo por prefijo más largo y desconocido. `IB412`, `LH1121` y `VY1869` son vuelos; `N.` y `OLS1` pueden resolverse por prefijo.

La columna `ID` de `swiftair_codes.csv` debe ser única. Las pruebas o el despliegue deben fallar ante duplicados.

##### 6.1 Extracción y normalización del identificador
- Buscar en `SUMMARY` la primera ruta con forma `IATA-IATA`, formada por dos códigos de tres letras separados por guion y delimitados claramente respecto del texto adyacente.
- Si existe una ruta, extraer como identificador candidato todo el texto situado antes de la ruta y eliminar únicamente sus espacios exteriores.
- El identificador candidato puede contener espacios; no se vuelve a dividir por el primer espacio antes de consultar el catálogo.
- Si no existe ninguna ruta `IATA-IATA`, utilizar el `SUMMARY` completo, sin espacios exteriores, como identificador candidato.
- Convertir el candidato a mayúsculas.
- Eliminar los caracteres de puntuación expresamente admitidos, incluidos los puntos finales, sin alterar letras, números ni los espacios interiores necesarios para IDs válidos del catálogo.
- Intentar primero una coincidencia exacta con el catálogo o lista correspondiente.
- Si no existe coincidencia exacta, buscar IDs que sean prefijo del candidato normalizado.
- Utilizar la coincidencia de mayor longitud.
- Ante varias coincidencias de la misma longitud, utilizar la primera fila según el orden original del catálogo o lista.

Ejemplos:
- `IM. MAD-MAD` → candidato `IM.` → `IM`.
- `N. Positioning (Pending to be issued)`, sin ruta → se compara el `SUMMARY` completo y se resuelve como `N` por prefijo válido.
- `IMI MAD-MAD` → `IMI`, no `IM`.
- `OLS1 STR-STR` → `OLS` si no existe `OLS1` y `OLS` es el prefijo válido más largo.
- `CRE (training) MAD-MAD` → candidato `CRE (training)`, lo que permite la coincidencia exacta con ese ID del catálogo.

##### 6.2 Códigos descartados
- Mantener una lista versionada `ignored_summary_codes` propia del módulo Swiftair.
- La lista inicial incluye `CATB`.
- Aplicar a la lista descartada la misma normalización del identificador candidato.
- Un evento descartado no genera evento ni advertencia de actividad desconocida.
- La lista se ampliará solo mediante cambio versionado y pruebas.

##### 6.3 Actividad de catálogo
Si el identificador candidato normalizado coincide con un ID de `swiftair_codes.csv` mediante las reglas anteriores:
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
- Si se detectó una ruta, usar como `slab` el identificador candidato original y como `description` el texto funcional restante que no sea la ruta.
- Si no se detectó una ruta, usar como `slab` el primer fragmento identificador disponible y como `description` el texto restante; si no puede separarse de forma segura, usar el `SUMMARY` completo como `slab` y una descripción vacía.
- Conservar `SUMMARY` original y generar advertencia no bloqueante.

#### 7. Sustitución de la fuente
- El usuario puede sustituir su URL Webcal por otra fuente válida.
- La nueva URL se valida y obtiene siempre en servidor y nunca se devuelve posteriormente al navegador.
- Un Webcal válido sin eventos se acepta directamente: el conjunto futuro vacío es un estado normal de Swiftair y no requiere confirmación adicional.
- La sustitución incrementa una generación estable de la fuente. Los trabajos y leases de generaciones anteriores quedan invalidados y no pueden confirmar resultados.
- La activación de la nueva fuente, la retirada del secreto anterior y la reconciliación se realizan de forma atómica o mediante una estrategia equivalente sin estados parciales visibles.

#### 8. Reconciliación
- Capturar una sola vez un `reconciliation_cutoff`; no volver a consultar el reloj para mover la frontera.
- Temporizados con `starts_at <= reconciliation_cutoff` y días completos ya comenzados en la fecha civil del corte quedan protegidos: no se incorporan, actualizan ni eliminan.
- Los eventos que eran futuros en el corte siguen siendo reconciliables aunque comiencen durante el proceso.
- El conjunto autoritativo sustituye todos los eventos Swiftair reconciliables por sus eventos válidos dentro de ventana; si queda vacío, elimina todo el futuro reconciliable.
- No emparejar por proximidad ni depender de UID estable entre descargas.
- Las recurrencias se expanden dentro de ventana y se protegen, reconcilian y cancelan por ocurrencia. `EXDATE`, cancelaciones y excepciones no alteran una ocurrencia protegida.
- La limpieza general es independiente y puede retirar eventos comenzados completamente fuera de ventana.
- La operación es idempotente y no deja conjuntos parciales visibles.
- Antes del commit deben seguir vigentes cuenta activa, `account_generation`, generación de fuente, compañía, lease, versión de perfil, base, zona, catálogo, configuración y ventana efectiva.
- Swiftair no permite edición o eliminación manual de importados.

#### 9. Criterios de aceptación
- La URL no es visible para clientes posteriores al alta ni amistades.
- SSRF, redirecciones, IPv4, IPv6, DNS rebinding y acceso cruzado están probados.
- Una sincronización con el mismo hash no escribe ni duplica eventos.
- Cuando se procesa un cuerpo autoritativo, su conjunto efectivo sustituye todos los eventos Swiftair reconciliables dentro de la ventana. Son reconciliables los temporizados con `starts_at > reconciliation_cutoff` y los días completos aún no comenzados en la fecha civil del corte conforme a su zona original. La autoridad no modifica eventos protegidos.
- Un cuerpo de cero octetos correctamente obtenido, incluido `204 No Content`, o un iCalendar admisible sin eventos válidos elimina todos los eventos futuros reconciliables, con independencia del estado HTTP.
- Un iCalendar parcial o truncado utiliza únicamente sus `VEVENT` completos e individualmente válidos y reemplaza el futuro por ellos.
- Un iCalendar autoritativo sin ningún `VEVENT` válido reemplaza el futuro por un conjunto vacío.
- HTML, contenido claramente ajeno a iCalendar o un fallo de obtención sin cuerpo utilizable conserva temporalmente el estado anterior.
- La prueba de admisibilidad acepta vacío exacto o UTF-8 con primera línea lógica `BEGIN:VCALENDAR` y `VERSION:2.0` antes del primer componente; rechaza espacios solos, prefijos, HTML, JSON, XML, binario, UTF-8 inválido, NUL y `VEVENT` aislado.
- EOF limpio tras la frontera puede ser truncamiento autoritativo; timeout, corte local, descompresión fallida o límite excedido no lo es.
- `DTEND` y `DURATION` juntos invalidan el `VEVENT`; `DURATION` positiva deriva un final posterior determinista, respeta unidades civiles a través de DST y solo admite semanas o días enteros con `VALUE=DATE`.

- Los eventos ya comenzados en `reconciliation_cutoff` no se incorporan, actualizan ni eliminan; el mismo corte inmutable gobierna toda la operación.
- Un evento futuro en el corte sigue siendo reconciliable aunque comience durante el procesamiento o antes del commit.
- Un cambio de UID o identidad de serie no crea una copia nueva de una ocurrencia protegida.
- Las recurrencias se protegen y reconcilian por ocurrencia, sin congelar las futuras porque otra ocurrencia ya haya comenzado.
- La limpieza general puede eliminar un evento protegido frente a reconciliación si queda completamente fuera de ventana.
- Se manejan DST, varios días, `VALUE=DATE`, cancelaciones y recurrencias acotadas.
- `STATUS:CANCELLED` no produce evento visible.
- Un temporizado de duración cero se convierte en día completo únicamente con señal inequívoca `Full day`; sin ella es inválido. Con `Z`, UTC es la zona civil canónica y `start_date` es la fecha UTC; con `TZID` se usa ese TZID y, si es flotante, se exige zona general explícita. `DESCRIPTION` no se conserva como descripción visible.
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
- La primera importación solo completa onboarding cuando confirma y almacena al menos un evento efectivo y visible; un resultado vacío, descartado, cancelado, terminado, fuera de ventana, inválido, incompleto o no confirmado no lo completa.
- En la primera importación se excluyen los eventos ya terminados en `onboarding_cutoff`; uno todavía vigente se incorpora aunque haya comenzado en el mes anterior, si se solapa con la ventana.
- Un Webcal de sustitución válido pero vacío se acepta directamente y retira el futuro anterior conforme a la reconciliación.
- Los trabajos de una generación anterior de la fuente no pueden escribir después de sustituirla.
- Un VEVENT duplicado conserva determinísticamente la versión más nueva.
- Los cambios de catálogo no reclasifican eventos ya almacenados.
- Si una firma ofrece dos fechas anteriores válidas, se elige la más antigua y más alejada de la salida.


#### 10. Reglas consolidadas
##### 10.1 Identidad y temporalidad
- Sin UID, usar huella versionada con integración `swiftair`, clase temporal, inicio y fin normalizados, `SUMMARY`, `LOCATION` e identidad de recurrencia; excluir `DESCRIPTION`.
- Un temporizado requiere `DTSTART` y exactamente uno entre `DTEND` o `DURATION`. `DURATION` sigue §4.1; no se inventa duración temporizada.
- `DTSTART` y `DTEND` comparten clase: ambos `DATE` o ambos `DATE-TIME`; recurrencias y excepciones son compatibles. Mezclas son errores individuales.
- `DTSTART;VALUE=DATE` sin final representa un día civil.
- Extremos temporizados pueden usar zonas distintas; resolver ambos a UTC, conservar ambas y usar la zona de inicio como principal. No corregir intervalos invertidos.
- Una hora flotante solo se interpreta con una zona general explícita y válida declarada mediante una propiedad admitida y versionada por el adaptador; no deducirla de perfil, base, ruta, `LOCATION`, servidor ni último `TZID`.
- Modelo discriminado: temporizados con instantes UTC y fechas civiles nulas; días completos con fechas civiles y los instantes nulos.

##### 10.2 Reconciliación, hash y ventana
- Capturar un único `reconciliation_cutoff`. No incorporar, actualizar ni eliminar eventos ya comenzados en ese corte. Para día completo usar la fecha civil del corte en su zona original.
- Esta protección solo rige reconciliación; limpieza general puede retirar eventos comenzados fuera de ventana.
- Todo cuerpo autoritativo forma el nuevo futuro con sus `VEVENT` completos e individualmente válidos; los inválidos o incompletos se omiten y pueden retirar versiones anteriores. Si no queda ninguno, el nuevo futuro es vacío.
- La autoridad depende de la admisibilidad definida en §3.1 y no del estado HTTP. `204` es vacío autoritativo. Un fallo local de adquisición no produce un cuerpo y conserva el futuro anterior.
- Mismo hash evita escrituras solo con la misma ventana. Con ventana nueva, volver a descargar y procesar el cuerpo. No conservar el Webcal bruto.
- La limpieza del mes saliente puede ejecutarse aunque falle la descarga del entrante.
- Ventana igual permite petición condicional; ventana distinta exige cuerpo completo. Si un `304` coincide con una carrera de ventana, permitir un solo GET no condicional adicional tras revalidar generación, lease, compañía y ventana.

##### 10.3 Sustitución y estados
- La confirmación inicial de sustitución autoriza reemplazar o eliminar el futuro anterior. Una fuente válida vacía se acepta sin segunda confirmación.
- **correcta:** todos los elementos procesables se resolvieron; vacío genuino también es correcto.
- **parcial:** se almacenó al menos un evento válido y hubo errores individuales.
- **fallida:** no se almacena ningún evento y existen errores individuales. Si el cuerpo era autoritativo, el futuro se sustituye igualmente por el conjunto válido resultante, aunque sea vacío; el estado no altera la autoridad.
