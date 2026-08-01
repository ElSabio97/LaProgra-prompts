# Amistades

## Reglas
- Las solicitudes se envían por nombre de usuario normalizado.
- El nombre de usuario es único sin distinguir mayúsculas y minúsculas.
- Una amistad aceptada es recíproca.
- No hay bloqueo en la primera versión.
- Eliminar una amistad rompe la relación para ambos.
- No existe límite funcional inicial, aunque se aplicará rate limiting contra abuso.
- El color de una amistad es una preferencia privada y local de cada usuario.
- Una amistad puede ocultarse temporalmente sin eliminarla.

## Estados
`pending`, `accepted`, `declined` y `cancelled`. Solo el destinatario acepta o rechaza; solo el remitente cancela una solicitud pendiente; cualquiera elimina una amistad aceptada.

## Privacidad
La búsqueda devuelve solo datos mínimos. No permite enumerar usuarios masivamente. Una amistad puede ver únicamente eventos compartidos, nunca correo, Webcal, CSV, configuración privada ni registros de importación.

## Integridad
Evitar solicitudes a uno mismo, duplicadas o cruzadas. Una solicitud inversa pendiente puede convertirse de forma transaccional en amistad aceptada. Usar restricciones únicas y operaciones atómicas.

## Criterios de aceptación
- Dos usuarios ven el mismo estado de amistad.
- El borrado elimina el acceso inmediatamente en ambos sentidos.
- Ocultar o cambiar color solo afecta al usuario que cambia la preferencia.
- RLS impide consultar relaciones ajenas.
