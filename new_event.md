# Actividades manuales

## Alcance
El propietario puede crear, consultar, editar y eliminar actividades manuales. No se utilizará esta función para modificar directamente eventos importados.

## Campos
- Título: obligatorio, 1 a 80 caracteres visibles.
- Inicio y fin: obligatorios; fin posterior al inicio.
- Todo el día: opcional.
- Zona horaria: IANA, por defecto la del perfil.
- Lugar, notas y color: opcionales y con límites definidos.
- Visibilidad para amistades: por defecto la misma política del calendario del usuario.

## Comportamiento
- Permitir eventos que crucen medianoche y de varios días.
- Advertir de solapamientos, pero no bloquearlos salvo regla futura explícita.
- Guardar instantes UTC y zona horaria original.
- Confirmar eliminaciones y permitir recuperación breve si la interfaz lo soporta.
- RLS limita todas las mutaciones al propietario.

## Criterios de aceptación
- Validación equivalente en cliente y servidor.
- Visualización correcta alrededor de DST.
- Los amigos no pueden modificar la actividad.
- Crear, editar y eliminar actualiza calendario y PDF.
