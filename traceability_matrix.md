# Matriz de trazabilidad de LaProgra

Convenciones: U unitaria; I integración; E2E extremo a extremo; SEC seguridad; PROP propiedades. Estado inicial: **por implementar**. Los fixtures solo aportan casos de prueba; ninguna fila los usa como fuente normativa.

## Transversales y temporalidad
- **TR-001 — Precedencia documental.** Documento: `prompt.md` §§2-3. Criterio: resolución determinista y contradicción explícita si la precedencia no basta. Pruebas: I.
- **TIME-001 — Unión temporal discriminada.** Documento: `prompt.md` §§7 y 19. Criterio: exactamente una variante válida. Pruebas: U,I,PROP.
- **TIME-002 — Final posterior y exclusivo.** Documento: `prompt.md` §7. Criterio: `ends_at > starts_at` o `end_date_exclusive > start_date`. Pruebas: U,PROP.
- **TIME-003 — Día completo sin instantes canónicos.** Documento: `prompt.md` §§7 y 19. Criterio: conversión de vista no muta identidad, ventana ni fechas. Pruebas: U,I,PROP.
- **WIN-001 — Solapamiento con ventana.** Documento: `prompt.md` §7.1. Criterio: conservar completo si solapa; omitir solo si queda completamente fuera. Pruebas: U,PROP.
- **WIN-002 — Token de ventana efectiva.** Documento: `prompt.md` §§7.1 y 19; ADR-005. Criterio: cambios de mes, perfil, base, zona, compañía, fuente, catálogo o configuración invalidan el token. Pruebas: I,PROP.
- **WIN-003 — Commit con generaciones vigentes.** Documento: `prompt.md` §§7.1, 8.1 y 19. Criterio: trabajo obsoleto no escribe. Pruebas: I,SEC,PROP.

## Iberia
- **IB-001 — Proceso local y cinco columnas.** Documento: `iberia_parser.md` §§1-2. Criterio: archivo y columnas descartadas nunca salen del dispositivo. Pruebas: E2E,SEC.
- **IB-002 — Codificación y CSV conforme.** Documento: `iberia_parser.md` §3. Criterio: UTF-8/Windows-1252, delimitadores permitidos, comillas, escapes y multilínea. Pruebas: U,PROP.
- **IB-003 — Cabeceras y proyección temprana.** Documento: `iberia_parser.md` §§2-5. Criterio: cinco cabeceras exactas; descarte antes de preview, error o red. Pruebas: U,I,SEC.
- **IB-004 — DST repetido.** Documento: `iberia_parser.md` §4. Criterio: candidatos y desempate determinista. Pruebas: U,PROP.
- **IB-005 — DST inexistente.** Documento: `iberia_parser.md` §4. Criterio: Asunto solo crea una pareja única y coherente; si no, ambigua. Pruebas: U,PROP.
- **IB-006 — Clasificación y firma contigua.** Documento: `iberia_parser.md` §6. Criterio: orden único y solo fila física siguiente. Pruebas: U,PROP.
- **IB-007 — Identidad, duplicados y reimportación.** Documento: `iberia_parser.md` §§7-8. Criterio: huella estable, sin duplicar, sobrescribir overrides ni reactivar tombstones. Pruebas: U,I,PROP.
- **IB-008 — Preview local versionada.** Documento: `iberia_parser.md` §5. Criterio: cambio interpretativo exige reselección y carrera impide commit. Pruebas: I,E2E,PROP.
- **IB-009 — Limpieza local y offline.** Documento: `iberia_parser.md` §§2,5,11; `prompt.md` §8.0.1. Criterio: cierre, cambio o borrado purga y no reabre datos offline. Pruebas: E2E,SEC.
- **IB-010 — Limpieza estricta del mes saliente.** Documento: `iberia_parser.md` §§10-12. Criterio: elimina evento, override y tombstone fuera de ventana. Pruebas: I,PROP.

## Swiftair: adquisición y autoridad
- **SW-001 — Secreto Webcal.** Documento: `swiftair_parser.md` §2; `prompt.md` §8. Criterio: secreto no vuelve a cliente ni observabilidad. Pruebas: I,SEC.
- **SW-002 — SSRF y redirecciones.** Documento: `swiftair_parser.md` §2; ADR-003. Criterio: DNS, IPv4/IPv6/mapeadas, Host/SNI, credenciales y cada salto validados. Pruebas: I,SEC,PROP.
- **SW-003 — Vacío exacto y 204.** Documento: `swiftair_parser.md` §§3-3.1. Criterio: cero octetos o 204 es autoritativo; espacios solos no. Pruebas: U,I,PROP.
- **SW-004 — Frontera iCalendar reconocible.** Documento: `swiftair_parser.md` §3.1. Criterio: UTF-8, primera línea, VERSION y prohibiciones exactas. Pruebas: U,SEC,PROP.
- **SW-005 — Truncamiento remoto frente a fallo local.** Documento: `swiftair_parser.md` §§3-3.1. Criterio: EOF limpio puede ser autoritativo; timeout, corte, descompresión o límite no. Pruebas: I,SEC,PROP.
- **SW-006 — Estado HTTP independiente de autoridad.** Documento: `swiftair_parser.md` §3. Criterio: cuerpo admisible de cualquier estado puede sustituir; estado conserva función diagnóstica. Pruebas: U,I,PROP.
- **SW-007 — VEVENT completo e individualmente válido.** Documento: `swiftair_parser.md` §§3-4. Criterio: solo cerrados y válidos forman el conjunto; cero válidos vacía futuro. Pruebas: U,I,PROP.
- **SW-008 — Hash y ventana.** Documento: `swiftair_parser.md` §§3 y 10.2. Criterio: solo ambos iguales omiten reproceso. Pruebas: I,PROP.
- **SW-009 — ETag, Last-Modified y 304.** Documento: `swiftair_parser.md` §§3 y 10.2; ADR-003. Criterio: ventana nueva exige cuerpo; una sola recuperación incondicional tras carrera. Pruebas: I,PROP.

## Swiftair: temporalidad, recurrencia y clasificación
- **SW-010 — DATE, DATE-TIME, zonas y flotantes.** Documento: `swiftair_parser.md` §§4 y 10.1. Pruebas: U,PROP.
- **SW-011 — DURATION y exclusión con DTEND.** Documento: `swiftair_parser.md` §4.1. Criterio: positiva, final posterior, DATE solo días/semanas, DST determinista. Pruebas: U,PROP.
- **SW-012 — Duración cero y Full day.** Documento: `swiftair_parser.md` §4.2. Criterio: señal necesaria; Z usa fecha UTC; DESCRIPTION no visible. Pruebas: U,PROP.
- **SW-013 — Identidad UID/huella y duplicados.** Documento: `swiftair_parser.md` §§4 y 10.1. Pruebas: U,PROP.
- **SW-014 — Conjunto recurrente.** Documento: `swiftair_parser.md` §4; ADR-006. Criterio: DTSTART/RRULE/RDATE menos EXDATE y excepciones. Pruebas: U,I,PROP.
- **SW-015 — Identidad y protección por ocurrencia.** Documento: `swiftair_parser.md` §§4 y 8. Criterio: cambio de UID no duplica protegidas y no congela futuras. Pruebas: I,PROP.
- **SW-016 — Cancelaciones recurrentes.** Documento: `swiftair_parser.md` §§4 y 8. Criterio: cancelación individual o maestra determinista. Pruebas: U,I,PROP.
- **SW-017 — Orden de clasificación.** Documento: `swiftair_parser.md` §6. Criterio: descartado exacto, catálogo exacto, GRD, vuelo, prefijo largo, desconocido. Pruebas: U,PROP.
- **SW-018 — Firma Swiftair.** Documento: `swiftair_parser.md` §5. Criterio: zona explícita, solo vuelos y candidato anterior determinista. Pruebas: U,PROP.

## Swiftair: reconciliación y onboarding
- **SW-019 — Corte único inmutable.** Documento: `swiftair_parser.md` §§8 y 10.2. Criterio: ninguna decisión vuelve a consultar el reloj. Pruebas: I,PROP.
- **SW-020 — Protección completa.** Documento: `swiftair_parser.md` §8. Criterio: no incorporar, actualizar ni eliminar comenzados; futuro en corte sigue reconciliable. Pruebas: I,PROP.
- **SW-021 — Limpieza separada.** Documento: `swiftair_parser.md` §§8 y 10.2. Criterio: puede retirar comenzados completamente fuera. Pruebas: I,PROP.
- **SW-022 — Primera importación vigente.** Documento: `swiftair_parser.md` §3. Criterio: excluye terminados, incluye en curso y exige solapamiento. Pruebas: U,I,E2E.
- **SW-023 — Finalización de onboarding.** Documento: `prompt.md` §10; `swiftair_parser.md` §§3 y 9. Criterio: al menos un evento efectivo, visible y confirmado. Pruebas: I,E2E,PROP.
- **SW-024 — Sustitución y generaciones.** Documento: `swiftair_parser.md` §7. Criterio: vacío válido aceptado y trabajos antiguos no escriben. Pruebas: I,E2E,PROP.
- **SW-025 — Estado independiente de autoridad.** Documento: `swiftair_parser.md` §10.3. Criterio: fallida autoritativa puede dejar futuro vacío. Pruebas: I,E2E,PROP.

## Seguridad, amistad y eliminación
- **RLS-001 — Propietario y cuenta activa.** Documento: `prompt.md` §8; ADR-007. Pruebas: I,SEC.
- **RLS-002 — Amistad y regla más restrictiva.** Documento: `friendships.md` §§4-6; ADR-007. Pruebas: I,E2E,SEC.
- **RLS-003 — Vistas, RPC y funciones privilegiadas.** Documento: `prompt.md` §8; ADR-007. Criterio: ningún bypass por ruta alternativa. Pruebas: I,SEC.
- **RLS-004 — Revocación inmediata.** Documento: `friendships.md` §§6-7. Pruebas: I,E2E,SEC.
- **SEC-001 — Redacción de secretos y PII.** Documento: `prompt.md` §8; ADR-010. Pruebas: I,SEC.
- **DEL-001 — Barrera deleting y generación.** Documento: `prompt.md` §8.1; ADR-008. Pruebas: I,SEC,PROP.
- **DEL-002 — Jobs, leases, triggers y Auth.** Documento: `prompt.md` §8.1; ADR-008. Pruebas: I,SEC,PROP.
- **DEL-003 — Purga local y reapertura offline.** Documento: `prompt.md` §§8.0.1-8.1; ADR-009. Pruebas: E2E,SEC.
- **DEL-004 — Retenciones de proveedor.** Documento: `prompt.md` §§8.0.1-8.1; ADR-015. Criterio: inventariadas, sin lectura/restauración ni reactivación. Pruebas: SEC, revisión operativa.

## PDF, manuales, catálogos y adaptadores
- **PDF-001 — Snapshot autorizada y LayoutPlan.** Documento: `download_pdf.md`. Pruebas: U,I,PROP.
- **PDF-002 — Cobertura y desbordamiento.** Documento: `download_pdf.md`. Pruebas: I,PROP.
- **PDF-003 — Temas equivalentes, Unicode y leyenda.** Documento: `download_pdf.md`. Pruebas: I,PROP.
- **PDF-004 — Temporales.** Documento: `download_pdf.md`; `prompt.md` §8.0.1. Pruebas: I,SEC.
- **MN-001 — CRUD manual y temporalidad.** Documento: `new_event.md`. Pruebas: U,I,E2E.
- **MN-002 — Compartición manual.** Documento: `new_event.md`; `friendships.md`. Pruebas: I,E2E,SEC.
- **CAT-001 — Integridad de airports.** Documento: `prompt.md` §3. Criterio: claves y zonas válidas. Pruebas: U,PROP.
- **CAT-002 — Integridad Swiftair e Iberia.** Documento: parsers y catálogos. Criterio: unicidad exigida y precedencia documentada. Pruebas: U,PROP.
- **CAT-003 — Evolución de aeropuerto.** Documento: `prompt.md` §19. Criterio: cambio de zona requiere confirmación; desaparición bloquea importaciones. Pruebas: I,E2E,PROP.
- **ADP-001 — Capacidades declarativas.** Documento: `prompt.md` §6; ADR-011. Pruebas: I,PROP.
- **ADP-002 — Autoridad snapshot/incremental/append-only.** Documento: ADR-011. Pruebas: I,PROP.
- **ADP-003 — Suite contractual sin ramas por aerolínea.** Documento: `prompt.md` §6; ADR-011. Pruebas: I,PROP.

## Puerta de terminado
La CI falla si un identificador normativo o de trazabilidad se duplica, si una regla nueva carece de criterio verificable y niveles de prueba, si una referencia documental no existe o si una funcionalidad se declara terminada con filas relevantes sin pruebas. Un fixture puede enlazarse como caso, nunca como fuente de la regla.
