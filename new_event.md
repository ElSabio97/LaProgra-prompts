## Actividades manuales

### 1. Alcance

El propietario puede crear, consultar, editar y eliminar actividades manuales. Esta función no modifica directamente eventos importados de Iberia o Swiftair.

### 2. Campos

#### Título
- Obligatorio.
- Es el texto que aparece en el slab.
- No puede quedar vacío después de eliminar espacios exteriores.

#### Descripción
- Opcional.
- Se muestra literalmente.
- Se trata como texto seguro, no como HTML ejecutable.

#### Inicio y fin
- Obligatorios para eventos temporizados.
- El final debe ser posterior al inicio.
- Se permiten eventos que crucen medianoche y de varios días.

#### Todo el día
- Opcional.
- Cuando está activo, se representa con semántica de fecha civil completa y final exclusivo.
- Mantiene `start_date`, `end_date_exclusive` y zona original.
- No cambia de fecha al modificar la zona de visualización.

#### Zona horaria
- Debe ser una zona IANA válida.
- Por defecto se utiliza la zona del perfil.
- En eventos temporizados se guardan instantes UTC y zona original.
- En eventos de día completo se conservan las fechas civiles y zona original; los instantes derivados no gobiernan su presentación.

#### Lugar de inicio y lugar de fin
- Opcionales e independientes.
- Pueden coincidir.

#### Visibilidad
- Por defecto sigue las reglas generales de compartición del propietario.
- El propietario puede dejar de compartir globalmente todas las actividades manuales.
- Puede dejar de compartirlas para amistades concretas.
- Puede excluir amistades concretas de una actividad manual individual.
- La regla más restrictiva prevalece.
- Una amistad autorizada ve todos los campos funcionales, pero nunca puede editar ni eliminar.

### 3. Comportamiento
- Advertir de solapamientos, sin bloquearlos salvo regla futura explícita.
- Mostrar correctamente eventos alrededor de cambios DST.
- Confirmar eliminaciones.
- Si la interfaz ofrece recuperación breve, restaurar el evento completo y su visibilidad.
- Las exportaciones PDF posteriores reflejan el estado actualizado.
- RLS limita mutaciones al propietario.
- La consulta por amistades se autoriza en servidor y mediante RLS.
- Un evento temporizado puede caer dentro de la fecha civil ocupada por un evento de día completo al visualizar otra zona. Ambos se muestran de forma independiente; no se fusionan, recortan ni desplazan.

### 4. Validación
- Validación equivalente en cliente y servidor.
- Título no vacío.
- Inicio, fin y zona obligatorios en eventos temporizados.
- Final posterior al inicio en eventos temporizados.
- Fechas de inicio y fin exclusivo válidas en eventos de día completo.
- Zona IANA válida.
- Descripción y lugares almacenados como texto seguro.
- No se imponen límites arbitrarios de producto. Los límites técnicos contra abuso serán centralizados, documentados e idénticos en cliente y servidor.

### 5. Ventana
- No se permite crear ni editar una actividad para que quede completamente antes del inicio del mes natural anterior.
- Se permiten actividades del mes anterior, mes actual y cualquier fecha futura.
- Un evento que se solape parcialmente con el inicio de la ventana se conserva completo.

### 6. Criterios de aceptación
- El propietario puede crear, consultar, editar y eliminar una actividad manual.
- El título aparece en el slab y la descripción en el detalle.
- Los eventos temporizados guardan inicio, fin, zona original e instantes UTC.
- Los eventos de día completo guardan fechas civiles, final exclusivo y zona original.
- Un evento de día completo no cambia de fecha al cambiar la zona de visualización.
- Se admiten medianoche, varios días y DST.
- Los solapamientos generan advertencia sin bloqueo.
- Se pueden indicar lugares de inicio y fin diferentes.
- El propietario puede dejar de compartir actividades manuales globalmente o por amistad.
- Puede excluir una amistad de una actividad concreta.
- Una amistad autorizada ve todos los campos funcionales, pero no puede modificar.
- Una amistad excluida no puede consultar mediante UI ni API.
- Las modificaciones aparecen en calendario y PDF posteriores.
- Un evento temporizado y uno de día completo pueden coincidir visualmente sin alterar sus datos.
