# Exportación PDF

## Versiones
### Color
Usa los colores del calendario, manteniendo contraste suficiente.

### Blanco y negro
Optimizada para impresión. No depende de fondos de color; utiliza texto, bordes, patrones o marcadores distinguibles.

## Contenido común
- Formato A4 horizontal por defecto.
- Nombre de usuario y mes en cabecera.
- Semana desde lunes y zona horaria indicada.
- Cada evento muestra sigla o ruta y horario `HHmm-HHmm`.
- Leyenda inferior `IATA - Ciudad` solo para códigos presentes.
- Tipografía incrustada y caracteres Unicode correctos.

## Desbordamiento
No utilizar `+3` ni recortar eventos silenciosamente. Aumentar altura cuando sea viable; si no cabe, continuar el día o el mes en una página adicional con referencia clara. La exportación debe indicar si hubo errores de renderizado.

## Privacidad
El PDF respeta los filtros de amistades visibles. No incluye correos, URLs Webcal, identificadores internos ni notas privadas no seleccionadas.

## Criterios de aceptación
- Ambas versiones contienen el mismo conjunto de eventos.
- No hay texto cortado ni eventos omitidos.
- La impresión en escala de grises sigue siendo interpretable.
- Meses de cuatro, cinco y seis semanas se renderizan correctamente.
