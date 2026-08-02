# Actividades manuales

## 1. Alcance

El propietario puede crear, consultar, editar y eliminar actividades manuales. Esta función no se utiliza para modificar directamente eventos importados de Iberia o Swiftair.

## 2. Campos

### Título

- Obligatorio.
- Es el texto que aparece en el slab del calendario.
- No puede quedar vacío después de eliminar espacios exteriores.

### Descripción

- Opcional.
- Su contenido se muestra literalmente en la descripción del evento.
- Se trata como texto seguro, no como HTML ejecutable.

### Inicio y fin

- Obligatorios.
- El final debe ser posterior al inicio.
- Se permiten eventos que crucen medianoche y eventos de varios días.

### Todo el día

- Opcional.
- Cuando esté activo, se representa con semántica de fecha completa en la zona horaria original.

### Zona horaria

- Debe ser una zona IANA válida.
- Por defecto se utiliza la zona horaria del perfil.
- Se guardan los instantes normalizados en UTC y la zona horaria original.

### Lugar de inicio y lugar de fin

- Opcionales e independientes.
- Permiten indicar dónde comienza y dónde termina la actividad.
- Pueden coincidir.

### Visibilidad

- Por defecto sigue la política general del calendario del propietario.
- El propietario puede ocultar la actividad a amistades concretas.
- Una amistad autorizada puede ver todos los campos de la actividad, pero nunca editarla ni eliminarla.

## 3. Comportamiento

- Advertir de solapamientos, pero no bloquearlos salvo regla futura explícita.
- Mostrar correctamente los eventos alrededor de cambios DST.
- Confirmar las eliminaciones.
- Si la interfaz ofrece recuperación breve, debe restaurar el evento completo y su configuración de visibilidad.
- Las exportaciones PDF posteriores a una creación, edición o eliminación deben reflejar el estado actualizado.
- RLS limita todas las mutaciones al propietario.
- La consulta por amistades se autoriza en servidor y mediante RLS, no solo ocultando elementos en la interfaz.

## 4. Validación

- Validación equivalente en cliente y servidor.
- Título no vacío.
- Inicio, fin y zona horaria obligatorios.
- Final posterior al inicio.
- Zona horaria IANA válida.
- Descripción y lugares almacenados como texto seguro.
- No se imponen límites arbitrarios de producto. Pueden aplicarse límites técnicos razonables contra abuso o entradas desproporcionadas, centralizados, documentados e idénticos en cliente y servidor.

## 5. Criterios de aceptación

- El propietario puede crear, consultar, editar y eliminar una actividad manual.
- El título aparece en el slab y la descripción en el detalle.
- Se guardan correctamente inicio, fin, zona original e instantes UTC.
- Se admiten medianoche, varios días y DST.
- Los solapamientos generan advertencia sin bloqueo.
- Se pueden indicar lugares de inicio y fin diferentes.
- El propietario puede ocultar una actividad a amistades concretas.
- Una amistad autorizada ve todos los campos, pero no puede modificar el evento.
- Una amistad excluida no puede consultarlo mediante UI ni API.
- Las modificaciones aparecen en el calendario y en PDF posteriores.
