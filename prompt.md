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
- El núcleo no contendrá condicionales de negocio específicos de Iberia o Swiftair.
- Migraciones Supabase versionadas, legibles y reproducibles.
- Entornos local, preview y producción con secretos separados.
- Documentación de instalación, configuración, pruebas, mantenimiento, recuperación y despliegue.

#### 7. Modelo mínimo de datos

Entidades mínimas: `profiles`, `events`, `friendships`, `friend_preferences`, `sharing_preferences`, `event_visibility_exclusions` e `import_sources`.

Cada evento importado tendrá `owner_id`, integración o aerolínea, `starts_at`, `ends_at`, zona horaria original y datos normalizados. Los eventos de día completo conservarán además fechas civiles de inicio y fin exclusivo y no dependerán únicamente de instantes UTC para su presentación.

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

La limpieza será automática e idempotente y eliminará también overrides, tombstones, exclusiones y otros datos dependientes que hayan quedado sin evento asociado.

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

##### 8.1 Eliminación de cuenta

Antes de eliminar la cuenta se requiere confirmación explícita mediante el texto correspondiente al idioma de la interfaz:
- Español: **Por favor, confirma. Esta acción borrará permanentemente tu cuenta y los datos almacenados en LaProgra.**
- Inglés: **Please confirm. This will permanently delete your account and all data stored in LaProgra.**

Tras la confirmación se eliminan la cuenta y absolutamente todos los datos asociados controlados por LaProgra, incluidos perfil, eventos importados, actividades manuales, overrides, tombstones, fuentes, secretos Webcal, amistades, solicitudes, preferencias, exclusiones, estados de importación, incidencias, trabajos pendientes, leases y registros técnicos vinculados. La relación de amistad se elimina sin borrar datos propios de la otra persona. La pérdida de acceso para las amistades es inmediata. La operación es irreversible e idempotente ante reintentos.

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

El proceso será reanudable e idempotente. Un fallo no debe crear datos parciales irreparables. Una primera sincronización Swiftair válida pero sin eventos no completa el onboarding.

#### 11. Importaciones

##### Swiftair

Webcal privado. Primera importación y posteriores sincronizaciones en servidor. Los cambios deben aparecer normalmente en menos de cinco minutos. La planificación debe usar colas o lotes, control de concurrencia, timeout, backoff, jitter, caché condicional y registro de estado. No lanzar una petición externa independiente por usuario cada minuto.

Se intentará revisar cada fuente vencida con una cadencia objetivo de un minuto, distribuyendo las fuentes mediante lotes. Se usarán `ETag` y `Last-Modified` cuando existan. Cuando se obtenga contenido, se calculará el hash del contenido completo:
- si el hash no cambia, no se modifican eventos;
- si cambia, se aplica la reconciliación definida en `swiftair_parser.md`.

Una nueva respuesta obtenida es la fuente de verdad para los eventos futuros interpretables. Un cuerpo vacío o un iCalendar válido sin eventos representa un conjunto futuro vacío. Si el contenido contiene `VEVENT` completos interpretables, se usa el conjunto de esos eventos aunque el documento esté parcial o incorrectamente terminado. Si se obtiene contenido pero no contiene ningún `VEVENT` completo interpretable, representa un conjunto futuro vacío. Un fallo de conexión o de obtención que no produzca cuerpo nuevo conserva temporalmente el estado anterior.

Los eventos con `starts_at <= now` quedan protegidos: no se eliminan ni actualizan durante reconciliación. Los eventos importados no se editan ni eliminan manualmente, salvo correcciones administrativas documentadas.

##### Iberia

CSV procesado y anonimizado en navegador. Usar `source_uid` o una huella determinista para garantizar idempotencia. Dos filas con huellas idénticas se consideran duplicadas y generan un único evento.

Iberia no proporciona un nuevo CSV cuando cambia horarios o cancela eventos ya publicados. El propietario editará manualmente esos cambios, almacenados como overrides por campo, y eliminará manualmente las cancelaciones, almacenadas como tombstones. Una reimportación idéntica preservará overrides y tombstones y no duplicará eventos. No se intentará inferir actualizaciones mediante proximidad temporal.

Una importación completamente fuera de la ventana se rechaza antes de confirmarla. Si contiene filas dentro y fuera, solo se aceptan las filas que se solapen y las demás se muestran como omitidas.

La previsualización se mantiene hasta que el usuario la acepta o vuelve a importar otro archivo. Al confirmar, se recalcula la ventana vigente.

##### Cambio de aerolínea

Requiere confirmación explícita. Elimina fuentes y eventos importados de la compañía anterior, incluidos overrides, tombstones, secretos, estados e incidencias relacionados. Conserva el perfil, las amistades, las preferencias y todos los eventos creados manualmente. Registrar la operación sin conservar el contenido eliminado.

#### 12. Calendario y zonas de visualización
- Mes completo, semana desde lunes.
- Todos los eventos visibles en el calendario interactivo. No usar `+3` ni ocultar slabs.
- Las filas pueden crecer; en móvil se prioriza texto legible y desplazamiento razonable.
- Slab con sigla o ruta IATA-IATA; detalle accesible bajo interacción.
- Orden determinista por inicio, duración y clave estable.
- No depender solo del color para diferenciar usuarios o estados.
- La exportación PDF se rige por `download_pdf.md` y puede paginar, pero nunca recortar eventos silenciosamente.

Cada usuario puede elegir la zona de visualización de los eventos temporizados:
- UTC;
- zona de su propia base;
- zona de otra base seleccionada mediante IATA y resuelta con `airports.csv`.

La elección es una preferencia de visualización y no modifica los datos almacenados. Los eventos de día completo conservan su fecha civil y no se desplazan al cambiar la zona de visualización.

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
- Importaciones idempotentes y reconciliación probada.
- Ventanas de almacenamiento y eventos solapados probados.
- PWA instalable, responsive y bilingüe.
- Revisión de seguridad y privacidad.
- Lista manual de comprobaciones completada.
- README y ADR actualizados.
