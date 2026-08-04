## Exportación PDF

### Versiones

#### Color
Usa los colores del calendario, manteniendo contraste suficiente.

#### Blanco y negro
Optimizada para impresión. No depende de fondos de color; utiliza texto, bordes, patrones o marcadores distinguibles.

### Contenido común
- Formato A4 horizontal por defecto.
- Nombre de usuario y mes en cabecera.
- Semana desde lunes.
- Zona de visualización indicada: UTC, base propia u otra base seleccionada.
- Cada evento temporizado muestra sigla o ruta y horario `HHmm-HHmm` en la zona elegida.
- Los eventos de día completo se muestran como tales y conservan su fecha civil; no se desplazan por la zona de visualización.
- Leyenda inferior `IATA - Ciudad` solo para códigos presentes.
- Tipografía incrustada y caracteres Unicode correctos.

### Visibilidad
- La exportación aplica las mismas reglas de compartición, exclusiones y preferencias locales que la vista utilizada para generarla.
- Si el usuario ha ocultado la programación de una amistad, esa programación no aparece en su PDF.
- Si el propietario ha dejado de compartir eventos con quien genera la vista, esos eventos no aparecen.

### Desbordamiento
- No usar `+3` ni recortar eventos silenciosamente.
- Aumentar altura cuando sea viable.
- Si no cabe, continuar el día o el mes en otra página con referencia clara.
- La exportación debe indicar si hubo errores de renderizado.
- Un evento temporizado y otro de día completo que coincidan en una fecha se representan por separado sin fusionarlos.

### Privacidad
- El PDF respeta los filtros de amistades visibles y las reglas de compartición.
- No incluye correos, URLs Webcal, identificadores internos, datos técnicos, campos brutos de importación ni notas privadas no seleccionadas.

### Criterios de aceptación
- Ambas versiones contienen el mismo conjunto de eventos.
- No hay texto cortado ni eventos omitidos.
- La impresión en escala de grises sigue siendo interpretable.
- Meses de cuatro, cinco y seis semanas se renderizan correctamente.
- Los eventos de día completo mantienen su fecha civil en todas las zonas de visualización.
- Los eventos temporizados muestran la hora correspondiente a la zona seleccionada.
- Las reglas de compartición y preferencias locales se respetan.
- Si un día desborda, se pagina sin pérdida de eventos.
