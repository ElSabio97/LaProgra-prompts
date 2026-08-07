## Amistades, preferencias y compartición

### 1. Reglas de amistad
- Las solicitudes se envían por nombre de usuario normalizado.
- El nombre de usuario es único sin distinguir mayúsculas y minúsculas.
- Una amistad aceptada es recíproca.
- No hay bloqueo en la primera versión.
- Eliminar una amistad rompe la relación para ambos.
- No existe límite funcional inicial, aunque se aplicará rate limiting contra abuso.
- El color de una amistad es una preferencia privada y local de cada usuario.

### 2. Estados

`pending`, `accepted`, `declined` y `cancelled`.

Solo el destinatario acepta o rechaza; solo el remitente cancela una solicitud pendiente; cualquiera elimina una amistad aceptada.

### 3. Preferencias locales del observador
- Un usuario puede ocultar temporalmente la programación de una amistad sin eliminarla.
- Esta preferencia solo afecta a quien la configura.
- Ocultar una amistad no cambia los permisos concedidos por la otra persona.
- La amistad permanece aceptada y puede volver a mostrarse en cualquier momento.
- Tener una amistad agregada no obliga a mostrar su programación.
- El calendario conjunto superpone la programación propia y las programaciones de las amistades visibles, sin límite funcional de slabs por celda y sin ocultarlos mediante overflow.

### 4. Compartición controlada por el propietario

Por defecto, una amistad aceptada puede ver los eventos que el propietario no haya dejado de compartir.

El propietario dispone de dos niveles de control:

#### 4.1 Toda la programación
- Español: **Dejar de compartir progra**.
- Inglés: **Unshare roster**.
- Puede aplicarse globalmente a todas las amistades.
- Puede aplicarse individualmente a amistades concretas.
- Oculta tanto eventos importados como actividades manuales.

#### 4.2 Solo actividades manuales
- Español: **Dejar de compartir eventos privados**.
- Inglés: **Unshare private events**.
- En esta especificación, “eventos privados” se refiere exclusivamente a actividades manuales; no se relaciona con columnas descartadas de archivos de importación.
- Puede aplicarse globalmente a todas las amistades.
- Puede aplicarse individualmente a amistades concretas.
- No oculta eventos importados cuando la programación general sigue compartida.

#### 4.3 Exclusiones por actividad manual
- Una actividad manual puede excluir además a amistades concretas.
- La exclusión solo afecta al evento correspondiente.

#### 4.4 Precedencia

La regla más restrictiva prevalece:
1. Sin amistad aceptada no existe acceso.
2. Si el propietario deja de compartir toda la progra globalmente, ninguna amistad accede a sus eventos.
3. Si deja de compartir toda la progra con una amistad, esa amistad no accede a sus eventos.
4. Para una actividad manual, si deja de compartir eventos manuales globalmente, ninguna amistad accede a ella.
5. Para una actividad manual, si deja de compartir eventos manuales con una amistad, esa amistad no accede a ella.
6. Una exclusión del evento manual impide el acceso de la amistad excluida.
7. La preferencia local del observador de ocultar una amistad no revoca permisos; solo evita mostrar sus eventos en su propia UI y exportaciones personales.

### 5. Privacidad
- La búsqueda devuelve solo datos mínimos y no permite enumerar usuarios masivamente.
- Una amistad accede únicamente a eventos compartidos.
- Nunca accede a correo, Webcal, CSV, configuración privada, registros de importación, identificadores internos ni datos técnicos.
- Para Swiftair, nunca accede al `DESCRIPTION` bruto del Webcal; solo a la descripción funcional resultante del catálogo o clasificador.
- Una amistad autorizada puede ver todos los campos funcionales compartidos, pero nunca editar ni eliminar eventos ajenos.

### 6. Integridad y autorización
- Evitar solicitudes a uno mismo, duplicadas o cruzadas.
- Una solicitud inversa pendiente puede convertirse transaccionalmente en amistad aceptada.
- Usar restricciones únicas y operaciones atómicas.
- Las reglas de compartición se aplican en servidor y mediante RLS, no solo ocultando elementos en la interfaz.
- Las preferencias locales de visualización no deben utilizarse como sustituto de autorización.
- El borrado de una amistad elimina el acceso inmediatamente en ambos sentidos.

### 7. Criterios de aceptación
- Dos usuarios ven el mismo estado de amistad.
- El borrado elimina el acceso inmediatamente en ambos sentidos.
- Ocultar o cambiar color solo afecta al usuario que cambia la preferencia.
- Ocultar una amistad no modifica lo que esa amistad comparte ni rompe la relación.
- Dejar de compartir toda la progra globalmente impide el acceso de todas las amistades.
- Dejar de compartir toda la progra con una amistad solo impide el acceso de esa amistad.
- Dejar de compartir actividades manuales no oculta eventos importados.
- Las reglas globales, particulares y exclusiones por evento aplican la opción más restrictiva.
- Una amistad excluida no puede recuperar el evento mediante UI ni API.
- RLS impide consultar relaciones o eventos ajenos no compartidos.
- PDF y calendario conjunto respetan la misma visibilidad que la vista interactiva.
- La superposición de cualquier número de amistades no recorta, agrupa ni oculta slabs; las filas crecen según sea necesario.
