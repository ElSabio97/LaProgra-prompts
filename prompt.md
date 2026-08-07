### LaProgra: especificación maestra del producto

#### 1. Objetivo

Construye **LaProgra**, una PWA para que pilotos y TCP importen, visualicen y comparen sus programaciones laborales con las de sus amistades.

#### 2. Instrucciones para el agente
- Lee este documento y todos los documentos especializados antes de implementar.
- No inicies una funcionalidad si contiene un requisito `[PENDIENTE BLOQUEANTE]`.
- Si hay contradicción, este documento prevalece en producto y reglas transversales; el documento especializado prevalece en su funcionalidad. Si persiste la contradicción, registra la decisión antes de programar.
- No inventes reglas de negocio. Documenta las decisiones técnicas relevantes mediante ADR.
- Implementa pruebas automatizadas y completa los criterios de aceptación.

#### 3. Documentación complementaria
- `swiftair_parser.md`: importación Webcal de Swiftair.
- `iberia_parser.md`: importación y anonimización del CSV de Iberia.
- `upload_instructions.md`: instrucciones para usuarios.
- `iberia_codes.csv` y `swiftair_codes.csv`: catálogos literales, no traducibles.
- `download_pdf.md`: exportación de calendarios.
- `new_event.md`: actividades manuales.
- `friendships.md`: amistades, visibilidad y colores.
- `airports.csv`: `IATA,City,Timezone`.
- `airlines.csv`: `ICAO,IATA,Name`.
- `technical_adrs.md`: decisiones técnicas que concretan estas reglas sin introducir negocio.

Los CSV se guardarán en UTF-8, con cabecera, coma como delimitador y campos entre comillas cuando sea necesario. `Timezone` será una zona IANA válida.

Cuando una consulta a un catálogo produzca varias coincidencias válidas, se usará siempre la primera fila según el orden original del CSV, sin reordenar previamente el catálogo. Cuando una regla especializada disponga escoger la coincidencia de mayor longitud, esta se aplicará antes de resolver empates por orden original.

#### 4. Glosario
- **Programación o progra:** eventos laborales mensuales.
- **Compañía:** aerolínea empleadora.
- **Tripulante:** piloto o TCP.
- **TCP:** tripulante de cabina de pasajeros.
- **Base:** aeropuerto de sede operativa, identificado por IATA.
- **Evento:** vuelo, simulador, curso, día libre u otra actividad.
- **Slab:** bloque visual de un evento.
- **Actividad manual:** evento creado por el usuario.
- **Línea o pairing:** sucesión relacionada de vuelos.
- **Evento de día completo:** evento con semántica de fecha civil, sin desplazamiento de fecha por la zona de visualización.

#### 5. Alcance de la primera versión

Incluye autenticación, onboarding, Iberia, Swiftair, calendario mensual, actividades manuales, amistades, español e inglés, configuración y PDF. Quedan fuera pagos, mensajería, bloqueo de usuarios y otras aerolíneas.

Iberia y Swiftair son las integraciones iniciales, pero la arquitectura no asumirá una lista cerrada de compañías. Debe permitir añadir en el futuro nuevas aerolíneas mediante la misma interfaz común, sin rediseñar el núcleo del producto.

#### 6. Arquitectura
- Producción: `laprogra.app`, Vercel y Supabase.
- PWA responsive con prioridad móvil.
- Stack recomendado: Next.js estable, TypeScript estricto, Supabase y un sistema de i18n.
- Las integraciones de aerolínea implementarán una interfaz común y estarán desacopladas de UI, calendario y persistencia.
- Cada integración encapsulará sus formatos, catálogos, clasificación, identificadores, reconciliación, configuración versionada, códigos descartados y política de edición.
- La interfaz común declarará capacidades como adquisición, sincronización, reconciliación, edición, cancelaciones, recurrencias, secretos, previsualización y versionado de catálogos.
- Cada evento importado conservará las versiones del adaptador, catálogo y configuración usadas al clasificarlo. Un cambio posterior de catálogo no reclasifica eventos existentes; solo afecta a eventos procesados después del cambio.
- El núcleo no contendrá condicionales de negocio específicos de Iberia o Swiftair.
- Migraciones Supabase versionadas, legibles y reproducibles.
- Entornos local, preview y producción con secretos separados.
- Documentación de instalación, configuración, pruebas, mantenimiento, recuperación y despliegue.

#### 7. Modelo mínimo de datos

Entidades mínimas: `profiles`, `events`, `friendships`, `friend_preferences`, `sharing_preferences`, `event_visibility_exclusions` e `import_sources`.

Cada evento tendrá `owner_id`, integración o fuente, zona original, datos normalizados y las versiones del adaptador, catálogo y configuración que determinen su semántica.  
La representación temporal será estrictamente discriminada:
- **Evento temporizado:** `starts_at` y `ends_at` estarán presentes; `start_date` y `end_date_exclusive` serán nulos. `ends_at` deberá ser posterior a `starts_at`.
- **Evento de día completo:** `start_date` y `end_date_exclusive` estarán presentes; `starts_at` y `ends_at` serán nulos. `end_date_exclusive` deberá ser posterior a `start_date`.  
Las dos formas serán mutuamente excluyentes. Los instantes derivados de fechas civiles no forman parte de la representación canónica de un día completo, no se almacenan como `starts_at` o `ends_at` y no gobiernan presentación, identidad, ventana, reconciliación ni limpieza.

Los valores importados se conservan como datos de origen. La política depende de la fuente:
- **Swiftair:** los eventos importados no pueden editarse ni eliminarse manualmente por el usuario. La sincronización automática actualiza y elimina eventos futuros conforme a `swiftair_parser.md`.
- **Iberia:** Iberia no proporciona un CSV actualizado cuando cambia o cancela posteriormente un evento. El propietario refleja esos cambios manualmente: las ediciones se guardan como overrides por campo y las eliminaciones como tombstones. Una reimportación idéntica no duplica eventos, no sobrescribe overrides y no reactiva tombstones. No se intentará deducir por proximidad horaria que un evento distinto es una actualización.
- **Actividades manuales:** son eventos independientes y permiten creación, edición y eliminación por su propietario.

##### 7.1 Ventana de almacenamiento

Para eventos importados de Iberia, Swiftair y futuras aerolíneas, solo se almacenan eventos que se solapen con esta ventana:
- mes natural anterior;
- mes natural en curso;
- mes natural siguiente.

Para actividades manuales, se almacenan eventos que se solapen con esta ventana:
- mes natural anterior;
- mes natural en curso;
- cualquier fecha futura, sin límite funcional.

Las ventanas se calculan en la zona horaria del perfil del propietario. Sus límites se convierten a UTC para comparar eventos temporizados. Un evento se conserva si cualquier parte de su intervalo se solapa con la ventana aplicable, aunque comience antes o termine después. Solo se elimina cuando queda completamente fuera.

Los eventos de día completo se comparan por sus fechas civiles respecto de la ventana calculada en la zona del propietario.

En una importación de Iberia, la ventana se vuelve a calcular al confirmar la previsualización. La confirmación aplica la ventana vigente en ese momento.

La limpieza será automática e idempotente y eliminará también overrides, tombstones, exclusiones y otros datos dependientes que hayan quedado sin evento asociado. Al cambiar el mes natural en la zona del perfil, se elimina el mes que acaba de salir de la ventana: por ejemplo, el 31 de julio se conserva junio y el 1 de agosto junio queda fuera y se elimina. La limpieza también se reevalúa al cambiar la base o zona del perfil, confirmar una importación y completar una sincronización.

#### 8. Seguridad y privacidad
- Supabase Auth para correo y contraseña. También permitir acceso con Google.
- RLS obligatoria en todas las tablas con datos de usuario.
- Ningún secreto ni Webcal se entregará al navegador después de guardarse.
- Durante el alta inicial, la URL Webcal debe enviarse al servidor; después no se devolverá en HTML, JavaScript, estado del cliente, respuestas API, logs del navegador ni consultas posteriores.
- El CSV original de Iberia se procesa en el navegador, no se sube ni se almacena en servidor o base de datos.
- Solo se enviarán al servidor los eventos normalizados confirmados por el usuario.
- Los logs no contendrán Webcals, CSV originales, tokens ni datos personales.
- Validar formato de archivo CSV antes de procesarlo.
- Aplicar rate limiting razonable a búsqueda, importaciones y sincronizaciones.
- LaProgra no realiza ni conserva copias de seguridad propias de los datos de usuario.

##### 8.0.1 Datos locales y retenciones administradas
Los datos locales controlados por LaProgra se aíslan por cuenta y versión y tienen caducidad. Previsualizaciones, eventos normalizados pendientes, preferencias cacheadas, respuestas, blobs, descargas, PDF temporales, IndexedDB, Cache Storage, almacenamiento local y cualquier caché del Service Worker se eliminan al dejar de ser necesarios y, como mínimo, al aceptar o sustituir una importación, cerrar sesión, cambiar de compañía cuando corresponda y borrar la cuenta. Tras cerrar sesión o iniciar el borrado, una reapertura offline no puede volver a mostrar datos de la cuenta anterior. Si una limpieza local no puede completarse, se bloquea la lectura de esos datos y se reintenta sin reactivar la cuenta.
LaProgra no crea copias de seguridad propias de datos de usuario. Las copias o retenciones inevitables administradas por proveedores se inventarían y documentan con proveedor, categoría de datos, finalidad, plazo o criterio de expiración y mecanismo de supresión. Quedan fuera de los flujos ordinarios de lectura y restauración del producto, no pueden usarse para reactivar una cuenta borrada y conservan acceso mínimo, cifrado y auditoría durante su retención.

##### 8.1 Eliminación de cuenta

Antes de eliminar la cuenta se requiere confirmación explícita mediante el texto correspondiente al idioma de la interfaz:
- Español: **Por favor, confirma. Esta acción borrará permanentemente tu cuenta y los datos almacenados en LaProgra.**
- Inglés: **Please confirm. This will permanently delete your account and all data stored in LaProgra.**

Tras la confirmación se eliminan la cuenta y absolutamente todos los datos asociados controlados por LaProgra, incluidos perfil, eventos importados, actividades manuales, overrides, tombstones, fuentes, secretos Webcal, amistades, solicitudes, preferencias, exclusiones, estados de importación, incidencias, trabajos pendientes, leases y registros técnicos vinculados. La relación de amistad se elimina sin borrar datos propios de la otra persona. La pérdida de acceso para las amistades es inmediata. La operación es irreversible e idempotente ante reintentos.

Durante la barrera de eliminación, el cliente invalida inmediatamente el espacio local de la cuenta, revoca URLs de objetos y blobs, elimina previews y PDF temporales, purga IndexedDB, almacenamiento local y Cache Storage, y ordena al Service Worker retirar entradas asociadas a la cuenta. Las claves locales incluyen identidad de cuenta y versión para impedir acceso cruzado. El servidor no considera concluida la eliminación por el mero éxito de la limpieza local; ambas rutas son idempotentes e independientes.
Las retenciones administradas por proveedores no retrasan la barrera ni permiten acceso funcional: la identidad y los datos activos se eliminan conforme a la saga y cualquier copia retenida expira únicamente según la política documentada del proveedor.

#### 9. Autenticación e identidad
- Una persona no debe obtener dos perfiles funcionales con el mismo correo.
- La vinculación de identidades seguirá el comportamiento seguro documentado por Supabase. No se confiará solo en mensajes de interfaz.
- Nombre de usuario: 3 a 20 caracteres visibles; letras Unicode, números, punto, guion y guion bajo; sin espacios ni controles.
- Normalizar con NFKC y minúsculas a `username_normalized`, con índice único.
- Reservar nombres del sistema como `admin`, `support`, `laprogra` y equivalentes.

#### 10. Onboarding

Modal obligatorio hasta completar:
- nombre de usuario válido;
- compañía, inicialmente Iberia o Swiftair;
- base IATA válida, mostrando ciudad como comprobación;
- importación inicial correcta.

El proceso será reanudable e idempotente. Un fallo no debe crear datos parciales irreparables. La primera importación Swiftair completa onboarding únicamente cuando la transacción confirma y almacena al menos un evento efectivo y visible. Debe ser individualmente válido, no descartado ni cancelado, vigente en `onboarding_cutoff` y solapado con la ventana. Una actividad desconocida temporalmente válida cuenta; duplicados y recurrencias cuentan por los eventos u ocurrencias efectivos almacenados. Terminados, fuera de ventana, descartados, cancelados, inválidos, incompletos o no confirmados no completan onboarding.

#### 11. Importaciones

##### Swiftair

Webcal privado. El usuario aporta la URL desde el navegador, pero la primera importación, la descarga del contenido y las sincronizaciones posteriores se realizan siempre en servidor. `progra_swiftair.txt` es solo un fixture de ejemplo y no una fuente productiva. Los cambios deben aparecer normalmente en menos de cinco minutos. La planificación debe usar colas o lotes, control de concurrencia, timeout, backoff, jitter, caché condicional y registro de estado. No lanzar una petición externa independiente por usuario cada minuto.

Se intentará revisar cada fuente vencida con una cadencia objetivo de un minuto, distribuyendo las fuentes mediante lotes. Se usarán `ETag` y `Last-Modified` cuando existan. Cuando se obtenga contenido, se calculará el hash del contenido completo:
- si coinciden hash y ventana efectiva, no se modifican eventos; si cambia la ventana, se obtiene un cuerpo completo y se reevalúa aunque el hash sea igual;
- si cambia, se aplica la reconciliación definida en `swiftair_parser.md`.

La autoridad depende del cuerpo correctamente obtenido y de su admisibilidad, no del estado HTTP. Cualquier respuesta HTTP con cuerpo admisible puede ser autoritativa aunque no sea `2xx`; `204 No Content` es un vacío correctamente obtenido y autoritativo. Un cuerpo autoritativo conserva solo sus `VEVENT` completos e individualmente válidos y, si no queda ninguno, el futuro reconciliable es vacío. El estado HTTP puede afectar a observabilidad y reintentos, pero no anula la autoridad del cuerpo. Error de conexión, timeout, rechazo de seguridad, descarga interrumpida, límite excedido o respuesta sin cuerpo utilizable conserva temporalmente el futuro anterior. HTML o contenido claramente ajeno a iCalendar no es autoritativo. La frontera determinista de admisibilidad se define en `swiftair_parser.md`. El usuario puede sustituir su Webcal. Una fuente de sustitución válida pero sin eventos se acepta directamente y representa un conjunto futuro vacío. La fuente tendrá una generación estable para impedir que trabajos o leases anteriores escriban después de la sustitución.

Cada reconciliación captura una sola vez un `reconciliation_cutoff`, inmutable durante adquisición, interpretación, expansión, reconciliación y commit. No se vuelve a consultar el reloj para desplazar esa frontera.  
Quedan protegidos los temporizados con `starts_at <= reconciliation_cutoff` y los días completos cuyo `start_date` sea igual o anterior a la fecha civil del corte en su zona original. Un evento protegido no se incorpora, actualiza ni elimina. Un evento futuro en el corte sigue siendo reconciliable aunque comience durante el procesamiento o antes del commit.  
La protección se aplica por ocurrencia expandida. Un cambio de UID, identidad de serie o representación no puede incorporar un duplicado de una ocurrencia protegida. La limpieza general es independiente y puede retirar eventos, incluso comenzados, completamente fuera de la ventana.  
Si aparecen VEVENT duplicados de una misma identidad, se conserva determinísticamente el más nuevo conforme a `swiftair_parser.md`. Los eventos importados no se editan ni eliminan manualmente, salvo correcciones administrativas documentadas.

##### Iberia

CSV procesado y anonimizado en navegador. Las horas de `Asunto` no gobiernan el horario salvo la excepción estricta de una hora repetida por DST definida en `iberia_parser.md`, donde pueden usarse temporalmente sus horas UTC informativas; si no resuelven, se aplica la alternativa conservadora. Usar `source_uid` o una huella determinista para garantizar idempotencia. Dos filas con huellas idénticas se consideran duplicadas y generan un único evento.

Iberia no proporciona un nuevo CSV cuando cambia horarios o cancela eventos ya publicados. El propietario editará manualmente esos cambios, almacenados como overrides por campo, y eliminará manualmente las cancelaciones, almacenadas como tombstones. Una reimportación idéntica preservará overrides y tombstones y no duplicará eventos. No se intentará inferir actualizaciones mediante proximidad temporal.

Una importación completamente fuera de la ventana se rechaza antes de confirmarla. Si contiene filas dentro y fuera, solo se aceptan las filas que se solapen y las demás se muestran como omitidas.

La previsualización se mantiene hasta que el usuario la acepta o vuelve a importar otro archivo. Al confirmar, se recalcula la ventana vigente.

##### Cambio de aerolínea

Requiere confirmación explícita. Elimina fuentes y eventos importados de la compañía anterior, incluidos overrides, tombstones, secretos, estados e incidencias relacionados. Conserva el perfil, las amistades, las preferencias y todos los eventos creados manualmente. Registrar la operación sin conservar el contenido eliminado.

#### 12. Calendario y zonas de visualización
- Mes completo, semana desde lunes.
- El calendario superpone la programación propia y las programaciones compartidas de las amistades que el observador haya decidido mostrar. Las coincidencias semánticas quedan fuera de esta versión y se reservan para una funcionalidad futura.
- Todos los eventos autorizados y visibles aparecen en el calendario interactivo. No existe límite funcional de slabs por celda. No usar `+3`, agrupaciones sustitutivas, truncamiento, recorte ni overflow interno que oculte eventos.
- Cada fila semanal aumenta su altura tanto como sea necesario para acomodar todos los slabs. En móvil se prioriza texto legible y desplazamiento razonable de la vista completa, sin scroll interno que oculte eventos dentro de una celda.
- Slab con sigla o ruta IATA-IATA; detalle accesible bajo interacción.
- Orden determinista por inicio, duración y clave estable.
- No depender solo del color para diferenciar usuarios o estados.
- La exportación PDF se rige por `download_pdf.md` y puede paginar, pero nunca recortar eventos silenciosamente.

La base es válida cuando existe en `airports.csv`. Cambiar la base actualiza la zona del perfil a la zona IANA asociada y recalcula la ventana; no obliga a usar esa zona como zona de visualización.
Cada usuario puede elegir la zona de visualización de los eventos temporizados:
- UTC;
- zona de su propia base;
- zona de cualquier lugar seleccionado mediante IATA y resuelta con `airports.csv`, usando el mismo estilo de buscador por IATA y ciudad que la selección de base.

La elección es una preferencia de visualización y no modifica la base, la zona del perfil, la ventana ni los datos almacenados. Los eventos de día completo conservan su fecha civil y no se desplazan al cambiar la zona de visualización.

Un evento temporizado puede solaparse visualmente con un evento de día completo al convertirlo a otra zona. Ambos se muestran: el evento de día completo ocupa la banda de día completo de su fecha civil y el temporizado se coloca según los instantes convertidos. El solapamiento no modifica, recorta ni fusiona ninguno de los dos.

#### 13. Amistades y visibilidad

Amistad recíproca, solicitud por username normalizado, sin límite de producto inicial. Borrar rompe la relación en ambos sentidos. Cada usuario puede asignar color y ocultar temporalmente la programación de una amistad sin afectarla.

El propietario puede dejar de compartir:
- toda su programación, globalmente o para amistades concretas;
- solo sus actividades manuales, globalmente o para amistades concretas.

Las actividades manuales pueden además excluir a amistades concretas. La regla más restrictiva prevalece.

Ocultar la programación de una amistad es una preferencia local del observador: no cambia los permisos concedidos por el propietario ni afecta a la otra persona.

Las amistades autorizadas pueden ver todos los campos funcionales compartidos, pero nunca pueden editarlos. Nunca ven fuentes, enlaces Webcal, archivos CSV, correo, configuración privada, registros de importación, identificadores internos ni datos técnicos. En Swiftair nunca ven el campo bruto `DESCRIPTION` del Webcal; solo la descripción funcional resultante del catálogo o clasificador.

#### 14. Actividades manuales

Aplicar `new_event.md`. Solo el propietario puede crear, editar y eliminar. Validar intervalo, zona horaria y solapamientos según sus reglas.

No se permite crear ni editar una actividad manual para que quede completamente antes del inicio del mes natural anterior. Sí se permiten actividades del mes anterior, del mes actual y de cualquier fecha futura, sin límite funcional. Los eventos que se solapen con el inicio de la ventana se conservan completos.

#### 15. PDF

Aplicar `download_pdf.md`. Versiones color y blanco y negro; título, usuario, mes, horarios y leyenda. Si no cabe, paginar o continuar en hoja adicional. No omitir eventos sin indicarlo. Debe respetar todas las reglas de compartición y preferencias de visualización aplicables a la vista exportada.

#### 16. Idiomas y accesibilidad
- Español de España por defecto e inglés seleccionable.
- Códigos, rutas y descripciones de catálogos no se traducen.
- Contraste WCAG AA para texto normal cuando sea viable, foco visible, etiquetas, navegación por teclado, zoom, objetivos táctiles adecuados y `prefers-reduced-motion`.
- El significado no dependerá exclusivamente del color.

#### 17. Observabilidad y errores

Estados de importación visibles: pendiente, procesando, correcta, parcial y fallida. Mostrar mensajes accionables sin datos sensibles. Los registros técnicos vinculados a una cuenta se eliminan al eliminarla.

#### 18. Definición de terminado
- Migraciones reproducibles y RLS probada.
- Lint, tipos, pruebas unitarias, integración y E2E superadas.
- Importaciones idempotentes y reconciliación probada, incluida sustitución de Webcal por una fuente válida vacía y bloqueo de trabajos obsoletos por generación.
- Ventanas de almacenamiento, cambio de mes, cambio de base, eventos solapados y horas DST ambiguas probados.
- Calendario propio y de amistades probado sin límite de slabs, sin truncamiento ni overflow oculto.
- PWA instalable, responsive y bilingüe.
- Revisión de seguridad y privacidad.
- Lista manual de comprobaciones completada.
- README y ADR actualizados.


### 19. Decisiones transversales consolidadas
- **Eliminación:** barrera irreversible `active -> deleting`, incremento de `account_generation`, invalidación de fuentes, jobs y leases. Toda escritura exige perfil activo y generación coincidente en la misma transacción. Después se borran idempotentemente datos e identidad Auth.
- **Cambio de aerolínea:** borrar integración anterior y volver al onboarding; conservar perfil, amistades, preferencias y actividades manuales.
- **Modelo temporal:** todos los eventos cumplen el modelo temporal estrictamente discriminado definido en §7: temporizados mediante instantes UTC y días completos mediante fechas civiles con final exclusivo. Las dos variantes son mutuamente excluyentes.
- **Versionado:** conservar versiones lógicas y hashes; congelar clasificación, identidad, slab, descripción funcional, zona interpretativa e instantes. Nombres descriptivos pueden actualizarse sin alterar semántica.
- **Cambio de base:** mostrar «¡Atención! Al cambiar de base puede que se borren eventos. Si solo quieres cambiar el horario, ve a la opción Zona horaria» / «Warning! Changing base location can delete some events. If you would only like to change timezone go to Time Zones».
- **Evolución de airports.csv:** cambio de zona requiere aviso persistente y confirmación; mientras tanto se mantiene todo y continúan adquisiciones. IATA desaparecido conserva datos y consulta, pero bloquea nuevas importaciones/sincronizaciones hasta seleccionar base válida.
- **Estado parcial:** exige al menos un evento almacenado y un error individual; ventana, duplicados y avisos no bloqueantes no bastan.
- **Trazabilidad:** mantener `traceability_matrix.md`; ninguna regla normativa se considera terminada sin criterio y prueba automatizada.
