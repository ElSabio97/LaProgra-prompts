# Decisiones técnicas mediante ADR

Estas decisiones concretan las reglas normativas sin introducir reglas de negocio nuevas. Cada ADR queda cerrado para iniciar la implementación del componente afectado; cualquier cambio de semántica visible exige modificar primero los documentos normativos.

## ADR-001 — Modelo temporal discriminado
- El tipo de dominio es una unión sellada `Timed | AllDay`; no existe un estado híbrido.
- Persistencia y API imponen nulabilidad cruzada y finales posteriores mediante validación de dominio y restricciones equivalentes.
- Conversión para visualización nunca cambia la representación canónica.

## ADR-002 — Admisibilidad Webcal
- La adquisición produce uno de tres resultados: `Body(bytes, clean_eof)`, `Empty204` o `AcquisitionFailure`.
- Vacío exacto y `Empty204` son admisibles. Para no vacío se aplica en streaming la frontera normativa de `swiftair_parser.md` §3.1.
- El reconocedor no ejecuta HTML, XML ni sniffing MIME y no busca tokens en posiciones arbitrarias.
- Tras reconocer el contenedor, el parser emite únicamente `VEVENT` cerrados; los demás errores son incidencias individuales o de contenedor y no revierten autoridad.

## ADR-003 — Adquisición HTTP resistente a SSRF y caché condicional
- Solo se permiten `https` y la normalización inicial de `webcal` a `https`; se rechazan userinfo, hosts vacíos, puertos no permitidos y URL no normalizable.
- Para cada salto se resuelve DNS, se normalizan IPv4, IPv6 e IPv4 mapeada en IPv6 y se rechazan loopback, privados, link-local, multicast, benchmark, documentación, CGNAT, metadata cloud y rangos no enrutable definidos por la política versionada.
- Se fija la conexión a una dirección validada mientras `Host` y SNI conservan el hostname original. No se reutiliza una resolución no validada y se revalida cada redirección.
- Las credenciales, cabeceras sensibles y validadores solo se reenvían al mismo origen normalizado. Un cambio de origen elimina credenciales; una redirección insegura falla.
- Se aplican límites en streaming a cabeceras, cuerpo comprimido, cuerpo descomprimido, duración total, inactividad y redirecciones. Superarlos produce `AcquisitionFailure`, nunca un cuerpo truncado autoritativo.
- El estado HTTP no decide autoridad. `204` produce `Empty204`; cualquier otro estado puede producir `Body` si la transferencia termina limpiamente.
- Petición condicional solo cuando la ventana efectiva coincide. Se prefiere `If-None-Match`; `If-Modified-Since` se añade si existe. `304` reutiliza hash y resultado procesado solo si fuente, generación y ventana siguen coincidiendo.
- Si llega `304` con ventana distinta o cambia la ventana durante la petición, se revalidan generaciones y se permite un único GET incondicional. Otro `304`, carrera o cambio invalida el trabajo sin commit.

## ADR-004 — Custodia de secretos
- La URL Webcal se cifra con clave gestionada fuera de la base de datos y envoltura por secreto; solo el servicio de adquisición tiene permiso de descifrado.
- API, vistas, RPC, logs y analítica usan proyecciones que excluyen el secreto por construcción.
- Rotación de clave vuelve a envolver sin exponer el valor a clientes.

## ADR-005 — Generaciones, lease y ventana efectiva
- Cada trabajo captura un token inmutable con `account_generation`, `source_generation`, compañía, perfil, base, zona, versión de catálogo, versión de adaptador/configuración, lease y ventana efectiva.
- La ventana efectiva se serializa canónicamente como versión de algoritmo, zona IANA, inicio inclusivo, fin exclusivo y generaciones interpretativas anteriores. El hash del token identifica igualdad; el cuerpo Webcal no se conserva.
- Antes del commit, una única transacción bloquea o compara las filas de control, comprueba cuenta activa, lease vigente y coincidencia exacta del token. Cualquier diferencia descarta todo el resultado.
- El hash del cuerpo solo evita reproceso si coincide también el token completo de ventana efectiva.

## ADR-006 — Recurrencias, excepciones y cancelaciones
- La expansión ocurre después de validar la serie y antes de ventana/reconciliación, acotada por ventana efectiva más el margen mínimo necesario para incluir ocurrencias que empiezan antes y solapan dentro.
- La identidad de ocurrencia es `(source-series-id, recurrence-key)`. `source-series-id` usa UID o huella estable; `recurrence-key` es el valor original normalizado de la ocurrencia en la clase y zona de `DTSTART`, no su hora modificada por una excepción.
- Se forma el conjunto base con `DTSTART`, `RRULE` y `RDATE`; se eliminan claves de `EXDATE`; después se aplican excepciones por `RECURRENCE-ID`. `STATUS:CANCELLED` elimina solo la clave correspondiente; una serie maestra cancelada elimina todas sus ocurrencias reconciliables.
- Los duplicados de maestro o excepción se resuelven antes de expandir mediante SEQUENCE, LAST-MODIFIED, DTSTAMP y última aparición.
- Una excepción hereda propiedades del maestro y sustituye las presentes. Su `DTSTART` puede mover la ocurrencia, pero la identidad conserva el `RECURRENCE-ID` original. Duración propia sustituye la heredada.
- Protección, cancelación y reconciliación se aplican por ocurrencia usando el corte único. Una ocurrencia protegida no se duplica ni muta; las futuras siguen reconciliables.

## ADR-007 — RLS y matriz de autorización
- Todas las entidades con datos de usuario activan RLS sin excepciones para roles cliente. Las tablas base no se exponen directamente a amistades.
- Propietario activo: CRUD de perfil permitido por operación; CRUD de actividades manuales; lectura de importados; mutación de importados solo mediante servicio/adaptador autorizado. Estado `deleting` deniega toda escritura ordinaria.
- Amistad aceptada: solo lectura de una vista/RPC de eventos funcionales cuyo predicado reproduzca la regla más restrictiva. Nunca lee secretos, origen bruto, correo, configuración, logs, importaciones, overrides internos ni exclusiones ajenas.
- Pendiente, rechazada, cancelada, eliminada, oculta sin amistad o amistad sin compartición: sin lectura de eventos ajenos. Ocultar localmente no concede ni revoca autorización.
- Operaciones de amistad verifican actor, estado y transición atómicamente; búsqueda devuelve proyección mínima y aplica rate limiting.
- Funciones `SECURITY DEFINER` fijan `search_path`, validan identidad y generaciones, conceden solo EXECUTE explícito y no aceptan `owner_id` autoritativo del cliente.
- El rol de adquisición puede descifrar una fuente concreta mediante RPC estrecha y escribir solo resultados con token vigente. El rol administrativo no se usa en rutas de usuario y queda auditado.
- Las pruebas negativas recorren entidad, operación, rol, estado de amistad, revocación, exclusión, cuenta deleting y acceso mediante tabla, vista, RPC y función.

## ADR-008 — Eliminación de cuenta
- Saga irreversible: transición transaccional `active -> deleting`, incremento global, revocación, invalidación de fuentes/jobs/leases, borrado idempotente y, al final, identidad Auth.
- Triggers, callbacks y tareas administrativas exigen cuenta activa y generación coincidente.
- El cliente purga su espacio local por cuenta; el servidor no depende de esa purga para completar la saga.

## ADR-009 — Datos y cachés locales
- Toda clave local incluye ID de cuenta y versión de esquema/interpretación. Las entradas tienen TTL y no se comparten entre sesiones.
- Cierre de sesión o deleting instala primero una barrera local que impide lectura; después purga IndexedDB, local/session storage, Cache Storage, blobs, URLs de objeto, previews y PDF.
- El Service Worker recibe un mensaje de purga y además rechaza respuestas cacheadas cuya cuenta/versión no coincida. La purga es idempotente y se reintenta al arrancar.

## ADR-010 — Observabilidad segura
- La redacción ocurre antes de serializar. Esquemas allow-list excluyen URL Webcal, CSV, DESCRIPTION bruto, tokens, PII descartada y cuerpos HTTP.
- Errores externos usan códigos y metadatos mínimos; los valores sensibles se sustituyen antes de logs, trazas y analítica.
- Pruebas con canarios verifican todos los sinks.

## ADR-011 — Contrato de adaptadores
- El núcleo consume un descriptor versionado de capacidades, sin condicionales por aerolínea.
- Adquisición declara `manual-upload | polled-secret | api`, preview local/remota, secreto requerido y soporte condicional.
- Autoridad declara `full-snapshot`, `incremental` o `append-only`. Solo `full-snapshot` permite cancelación por ausencia; `incremental` exige cursor y cancelaciones explícitas; `append-only` nunca elimina por ausencia.
- El adaptador entrega elementos normalizados, incidencias, token de adquisición, versión de clasificación e identidad estable. Expone estrategias de recurrencia, cancelación, edición por campo y catálogo.
- Reconciliación recibe capacidades declaradas y rechaza combinaciones incoherentes, como ausencia-as-cancelación en incremental o edición de campos no declarados.
- Catálogo/configuración se versionan y forman parte del token interpretativo. Cambiar versión no reclasifica automáticamente eventos históricos salvo operación normativa expresa.
- Cada adaptador supera una suite contractual común para autoridad, idempotencia, duplicados, carreras, cancelaciones, ventana, preview y secretos.

## ADR-012 — LayoutPlan PDF
- Plan inmutable desde snapshot autorizada; medición con fuente final; geometría compartida por temas; verificación de cobertura y autorización antes de entrega.
- Temporales aislados por exportación se destruyen en éxito, fallo, cancelación y timeout.

## ADR-013 — Presupuestos de recursos
- Límites técnicos centralizados y versionados. Rechazo por límite es explícito y nunca se convierte en truncamiento autoritativo ni omisión silenciosa.

## ADR-014 — DST Iberia
- Enumerar candidatos UTC; usar Asunto solo para DST; aplicar la selección normativa y marcar ambigua si no existe intervalo coherente.

## ADR-015 — Retenciones administradas por proveedores
- Mantener inventario versionado de proveedor, servicio, región, categoría de datos, finalidad, cifrado, roles con acceso, plazo/criterio de expiración y mecanismo de supresión.
- Ninguna retención se monta en rutas de lectura, restauración o soporte ordinario. Restaurar una cuenta borrada desde esas copias está prohibido.
- Cambios de proveedor o plazo requieren revisión de privacidad y actualización documental antes del uso productivo.
