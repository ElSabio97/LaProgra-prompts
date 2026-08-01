# LaProgra: especificación maestra del producto

## 1. Objetivo
Construye **LaProgra**, una PWA para que pilotos y TCP importen, visualicen y comparen sus programaciones laborales con las de sus amistades.

## 2. Instrucciones para el agente
1. Lee este documento y todos los documentos especializados antes de implementar.
2. No inicies una funcionalidad si contiene un requisito `[PENDIENTE BLOQUEANTE]`.
3. Si hay contradicción, este documento prevalece en producto y reglas transversales; el documento especializado prevalece en su funcionalidad. Si persiste la contradicción, registra la decisión antes de programar.
4. No inventes reglas de negocio. Documenta las decisiones técnicas relevantes mediante ADR.
5. Implementa pruebas automatizadas y completa los criterios de aceptación.

## 3. Documentación complementaria
- `swiftair_parser.md`: importación Webcal de Swiftair.
- `iberia_parser.md`: importación y anonimización del CSV de Iberia.
- `upload_instructions.md`: instrucciones para usuarios.
- `iberia_codes.csv` y `swiftair_codes.csv`: catálogos literales, no traducibles.
- `download_pdf.md`: exportación de calendarios.
- `new_event.md`: actividades manuales.
- `friendships.md`: amistades, visibilidad y colores.
- `airports.csv`: `IATA,City,Timezone`.
- `airlines.csv`: `Name,ICAO,IATA`.

Los CSV se guardarán en UTF-8, con cabecera, coma como delimitador y campos entre comillas cuando sea necesario. `Timezone` será una zona IANA válida.

## 4. Glosario
- **Programación o progra:** eventos laborales mensuales.
- **Compañía:** aerolínea empleadora.
- **Tripulante:** piloto o TCP.
- **TCP:** tripulante de cabina de pasajeros.
- **Base:** aeropuerto de sede operativa, identificado por IATA.
- **Evento:** vuelo, simulador, curso, día libre u otra actividad.
- **Slab:** bloque visual de un evento.
- **Actividad manual:** evento creado por el usuario.
- **Línea o pairing:** sucesión relacionada de vuelos.
- **DH:** vuelo posicional en el que el tripulante viaja como pasajero, no como piloto.

## 5. Alcance de la primera versión
Incluye autenticación, onboarding, Iberia, Swiftair, calendario mensual, actividades manuales, amistades, español e inglés, configuración y PDF. Quedan fuera pagos, mensajería, bloqueo de usuarios y otras aerolíneas.

## 6. Arquitectura
- Producción: `laprogra.app`, Vercel y Supabase.
- PWA responsive con prioridad móvil.
- Stack recomendado: Next.js estable, TypeScript estricto, Supabase y un sistema de i18n.
- Las integraciones de aerolínea implementarán una interfaz común y estarán desacopladas de UI, calendario y persistencia.
- Migraciones Supabase versionadas, legibles y reproducibles.
- Entornos local, preview y producción con secretos separados.
- Documentación de instalación, configuración, pruebas, mantenimiento, recuperación y despliegue.

## 7. Modelo mínimo de datos
Entidades: `profiles`, `events`, `friendships`, `friend_preferences`, `import_sources`.

Cada evento importado tendrá `owner_id`, `airline`, `starts_at`, `ends_at`, zona horaria original, datos normalizados.

Las ediciones de eventos importados sustituirán al evento guardado. Las eliminaciones borrarán el evento en cuestión. Una reimportación no debe reactivar un evento eliminado ni sobrescribir un campo modificado sin aplicar la política de reconciliación.

## 8. Seguridad y privacidad
- Supabase Auth para correo y contraseña. También permitir acceso con Google.
- RLS obligatoria en todas las tablas con datos de usuario.
- Ningún secreto ni Webcal se entregará al navegador después de guardarse.
- El CSV original de Iberia se procesa en el navegador y no se sube.
- Los logs no contendrán Webcals, CSV originales, tokens ni datos personales.
- Validar formato de archivo CSV antes de subirlo.
- Aplicar rate limiting que consideres a búsqueda, importaciones y sincronizaciones.

## 9. Autenticación e identidad
- Una persona no debe obtener dos perfiles funcionales con el mismo correo.
- La vinculación de identidades seguirá el comportamiento seguro documentado por Supabase. No se confiará solo en mensajes de interfaz.
- Nombre de usuario: 3 a 20 caracteres visibles; letras Unicode, números, punto, guion y guion bajo; sin espacios ni controles.
- Normalizar con NFKC y minúsculas a `username_normalized`, con índice único.
- Reservar nombres del sistema como `admin`, `support`, `laprogra` y equivalentes.

## 10. Onboarding
Modal obligatorio hasta completar:
1. Nombre de usuario válido.
2. Compañía (de momento solo Iberia o Swiftair).
3. Base IATA válida, mostrando ciudad como crosscheck de que se ha elegido una base que existe.
4. Importación inicial correcta.

El proceso será reanudable e idempotente. Un fallo no debe crear datos parciales irreparables.

## 11. Importaciones
### Swiftair
Webcal privado. Primera importación y posteriores sincronizaciones en servidor. Los cambios deben aparecer normalmente en menos de cinco minutos. La planificación debe usar colas o lotes, control de concurrencia, timeout, backoff, jitter, caché condicional y registro de estado. No lanzar una petición externa independiente por usuario cada minuto.

Los eventos pasados se conservan. Los importados no se editan ni eliminan manualmente, salvo correcciones administrativas documentadas.

### Iberia
CSV procesado y anonimizado en navegador. Usar `source_uid` o una huella determinista. La reconciliación preservará overrides y tombstones. Los eventos importados podrán ajustarse mediante overrides sin mutar el registro de origen.

### Cambio de aerolínea
Requiere confirmación explícita. Elimina fuentes y eventos de la compañía anterior, incluidos overrides relacionados, pero conserva perfil, amistades y preferencias. Registrar la operación.

## 12. Calendario
- Mes completo, semana desde lunes y zona horaria del usuario.
- Todos los eventos visibles en el calendario interactivo. No usar `+3` ni ocultar slabs.
- Las filas pueden crecer; en móvil se prioriza texto legible y desplazamiento razonable.
- Slab con sigla o ruta `IATA-IATA`; detalle accesible bajo interacción.
- Orden determinista por inicio, duración y clave estable.
- No depender solo del color para diferenciar usuarios o estados.

La exportación PDF es una excepción: se rige por `download_pdf.md` y puede paginar, pero nunca recortar eventos silenciosamente.

## 13. Amistades
Amistad recíproca, solicitud por username normalizado, sin límite de producto inicial. Borrar rompe la relación en ambos sentidos. Cada usuario puede asignar color y ocultar temporalmente a una amistad sin afectarla.

Los amigos solo ven los datos de calendario expresamente compartidos, nunca fuentes, enlaces, importaciones, correo ni datos internos.

## 14. Actividades manuales
Aplicar `new_event.md`. Solo propietario puede crear, editar y eliminar. Validar intervalo, zona horaria y solapamientos según reglas del documento especializado.

## 15. PDF
Aplicar `download_pdf.md`. Versiones color y blanco y negro; título, usuario, mes, horarios y leyenda. Si no cabe, paginar o continuar en hoja adicional. No omitir eventos sin indicarlo.

## 16. Idiomas y accesibilidad
- Español de España por defecto e inglés seleccionable.
- Códigos, rutas y descripciones de catálogos no se traducen.
- Contraste WCAG AA para texto normal cuando sea viable, foco visible, etiquetas, navegación por teclado, zoom, objetivos táctiles adecuados y `prefers-reduced-motion`.
- El significado no dependerá exclusivamente del color.

## 17. Observabilidad y errores
Estados de importación visibles: pendiente, procesando, correcta, parcial y fallida. Mostrar mensajes accionables sin datos sensibles. Conservar métricas, trazas y errores con retención definida.

## 18. Definición de terminado
- Migraciones reproducibles y RLS probada.
- Lint, tipos, pruebas unitarias, integración y E2E superadas.
- Importaciones idempotentes y reconciliación probada.
- PWA instalable, responsive y bilingüe.
- Revisión de seguridad y privacidad.
- Lista manual de comprobaciones completada.
- README y ADR actualizados.
