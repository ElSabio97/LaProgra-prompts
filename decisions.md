# Registro de decisiones de LaProgra

Este registro consolida las decisiones adoptadas durante la revisión. Los documentos especializados siguen siendo normativos dentro de su función.

## D-001
Swiftair: orden único: descartado exacto, catálogo exacto, GRD, vuelo, catálogo por prefijo más largo y desconocido.

## D-002
Iberia DST: usar horas UTC de Asunto; respaldo conservador para no llegar tarde.

## D-003
Cambio de aerolínea: borrar integración y volver a onboarding; conservar excepciones.

## D-004
Estado parcial: al menos un evento guardado y un error individual.

## D-005
Preview Iberia local, responsive y sin reproceso al rotar.

## D-006
Identidad Iberia versionada; sin reconciliación entre CSV.

## D-007
Identidad Swiftair sin UID, excluyendo DESCRIPTION.

## D-008
Sin backups propios con datos de usuario.

## D-009
Modelo temporal discriminado.

## D-010
Versiones lógicas y hashes; semántica congelada.

## D-011
Cuerpos Swiftair: vacío o iCalendar reconocible es autoritativo; solo sus VEVENT completos y válidos forman el futuro. HTML o contenido claramente ajeno no es autoritativo.

## D-012
Cambio de base con advertencia; evolución de airports.csv controlada.

## D-013
Protección Swiftair solo en reconciliación; limpieza general independiente.

## D-014
Limpieza Iberia estricta, incluidos overrides y tombstones.

## D-015
Hash y ventana juntos determinan si hay reproceso.

## D-016
No conservar cuerpo Webcal bruto.

## D-017
Primera Swiftair solo eventos todavía vigentes.

## D-018
Condicionales, 304 y un único GET completo de recuperación.

## D-019
Compatibilidad DATE/DATE-TIME, zonas, flotantes y día sin final.

## D-020
Solo VEVENT completos e individualmente válidos forman el futuro; si todos son inválidos o incompletos, el futuro reconciliable queda vacío.

## D-021
Corte único; eventos ya comenzados no se incorporan ni mutan.

## D-022
Sustitución por fuente válida vacía sin segunda confirmación.

## D-023
Duración cero solo con señal Full day.

## D-024
Barrera de eliminación con estado y generación global.

## D-025
PDF con LayoutPlan determinista y verificación previa.

## D-026
Matriz de trazabilidad obligatoria.

### D-027
Primera Swiftair incluye eventos todavía vigentes aunque empezaran en el mes anterior.

### D-028
Reconciliación Swiftair usa un corte único inmutable y protege recurrencias por ocurrencia.

### D-029
Omitir reproceso exige coincidencia de hash y ventana efectiva.

### D-030
Modelo temporal estrictamente discriminado y mutuamente excluyente.

### D-031
Base desaparecida bloquea importaciones y sincronizaciones, no actividades manuales con zona IANA válida; todo trabajo revalida perfil, catálogo y ventana al commit.

### D-032
Iberia DST enumera candidatos UTC y aplica selección conservadora determinista.

### D-033
Las copias gestionadas inevitablemente por proveedores quedan fuera de la prohibición de backups propios y se documentan.

### D-034
Swiftair: la autoridad depende de la admisibilidad del cuerpo y no del estado HTTP. Cualquier respuesta HTTP con cuerpo admisible puede ser autoritativa; `204 No Content` representa futuro autoritativamente vacío.

### D-035
Onboarding Swiftair: solo se completa cuando la transacción confirma y almacena al menos un evento efectivo y visible, individualmente válido, no descartado ni cancelado, vigente en `onboarding_cutoff` y solapado con la ventana.

### D-036
Swiftair: en duración cero con señal `Full day`, un `DTSTART` con `Z` usa UTC como zona civil canónica y su fecha UTC como `start_date`.

### D-037
Un cuerpo Swiftair no vacío es reconocible como iCalendar solo si la descarga termina limpiamente, es UTF-8 válido, empieza por `BEGIN:VCALENDAR` y contiene exactamente `VERSION:2.0` antes del primer componente; los fallos locales de adquisición no son truncamientos autoritativos.

### D-038
Swiftair admite `DURATION` positiva y mutuamente excluyente con `DTEND`; con DATE solo semanas o días completos, y con DATE-TIME aplica aritmética civil para semanas y días y exacta para unidades menores.

### D-039
La eliminación abarca copias locales y reapertura offline. Las retenciones administradas por proveedores se documentan, quedan fuera de lectura y restauración del producto y no permiten reactivar cuentas borradas.
