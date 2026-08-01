# LaProgra — Especificación del proyecto

## 1. Objetivo y rol

Actúa como programador experto en desarrollo de aplicaciones web.

Construye **LaProgra**, una web app para ayudar a pilotos y TCP a comparar sus programaciones laborales y organizar mejor su vida personal.

Antes de comenzar:

1. Lee este documento completo.
2. Lee todos los archivos adjuntos.
3. No comiences el desarrollo si falta un archivo necesario o existe un requisito marcado como **[PENDIENTE]** que impida continuar.
4. Si existen contradicciones:
   - Prevalece este documento para los requisitos generales.
   - Prevalece el archivo especializado para los detalles de su funcionalidad.
   - Si la contradicción no puede resolverse con esta regla, pregunta antes de continuar.

---

## 2. Documentación complementaria

- `swiftair_parser.md`: importación y procesamiento del Webcal de Swiftair.
- `iberia_parser.md`: importación y anonimización del CSV de Iberia.
- `upload_instructions.md`: instrucciones de importación.
- `iberia_codes.csv`: descripciones de códigos de Iberia.
- `swiftair_codes.csv`: descripciones de códigos de Swiftair.
- `download_pdf.md`: generación de PDF.
- `new_event.md`: creación manual de actividades.
- `friendships.md`: gestión de amistades.
- `airports.csv`: catálogo de aeropuertos.
- `airlines.csv`: catálogo de aerolíneas.

### `airports.csv`

Contiene estas columnas:

- `IATA`
- `City`
- `Timezone`

`Timezone` es una zona horaria IANA válida, como `Europe/Madrid`.

### `airlines.csv`

Contiene estas columnas:

- `Airline Name`
- `ICAO`
- `IATA`



- Origen de `airports.csv`: de momento raiz del proyecto.
- Origen de `airlines.csv`: de momento raiz del proyecto.
- Se incluirán dentro del proyecto

---

## 3. Glosario

- **Programación:** listado mensual de eventos asignados a un empleado.
- **Progra:** abreviatura de **Programación:**
- **Compañía:** aerolínea.
- **Tripulante:** piloto o TCP empleado por una aerolínea.
- **TCP:** tripulante de cabina de pasajeros.
- **Base:** aeropuerto, identificado mediante su código IATA, donde el usuario tiene su sede operativa.
- **Evento:** tarea asignada, como un vuelo, simulador, curso o día libre.
- **Slab:** bloque de color que representa un evento dentro del calendario.
- **Actividad manual:** evento creado directamente por el usuario y no importado desde su aerolínea.
- **Línea:** sucesión de varios vuelos. Se llama **pairing** en inglés.
---

## 4. Infraestructura y arquitectura

- Nombre de la aplicación: **LaProgra**.
- Dominio de producción: `laprogra.app`. Lo compré en `cloudflare.com`
- Alojamiento de la aplicación: **Vercel**.
- Base de datos y autenticación: **Supabase**.
- Elige el stack más adecuado, priorizando:
  - Compatibilidad con Vercel y Supabase.
  - Seguridad.
  - Rendimiento.
  - Mantenimiento.
  - Escalabilidad.
  - Facilidad de comprensión por otros desarrolladores y agentes de IA.
- La aplicación debe ser instalable como **PWA**.
- Debe funcionar en las versiones actuales de los navegadores principales.
- Debe existir un entorno local de desarrollo.
- La producción se desplegará en Vercel cuando la aplicación esté preparada.
- La arquitectura debe permitir incorporar nuevas aerolíneas y métodos de importación sin rehacer la aplicación.
- Las reglas específicas de cada aerolínea deben estar desacopladas del calendario y de la interfaz.
- Debe existir una carpeta para las migraciones de Supabase.
- Las migraciones tendrán nombres legibles, ordenados y compatibles con Supabase.
- El código debe estar bien estructurado.
- Debe comentarse cuando el comentario aporte contexto o explique una decisión que no resulte evidente.
- Debe incluirse documentación para instalar, configurar, mantener y desplegar el proyecto.
- Incluir lista de comprobaciones manuales.

---

## 5. Aerolíneas e importación

La primera versión será compatible con **Swiftair** e **Iberia**.

La arquitectura no debe asumir que las futuras aerolíneas utilizarán Webcal o CSV.

### 5.1. Swiftair

- El usuario proporciona un enlace Webcal.
- Los eventos se procesan según `swiftair_parser.md`.
- El enlace Webcal se almacena de forma privada en Supabase.
- Ningún usuario, incluidos los amigos, debe poder visualizar literalmente el enlace Webcal de otra persona.
- El enlace no debe exponerse mediante consultas desde la consola del navegador.
- Las políticas RLS de Supabase deben impedir el acceso no autorizado.
- La sincronización debe actualizar los eventos futuros sin borrar eventos pasados.
- Los eventos importados de Swiftair no pueden editarse ni eliminarse manualmente.
- Flujo:
  - Primera importación de eventos cargados desde el enlace webcal al ser pegado en el navegador y actualizaciones posteriores mediante una Edge Function.
- Frecuencia de sincronización automática mediante Supabase Cron y Edge Functions: cada minuto.
- Definición de «eventos futuros»:
  - Desde el instante actual.
- Cifrado adicional del enlace Webcal antes de almacenarlo en Supabase: negativo.

### 5.2. Iberia

- El usuario proporciona un archivo CSV de la compañía.
- El archivo debe procesarse y anonimizarse en el navegador.
- El CSV original no debe subirse a Supabase.
- Solo se subirán los datos necesarios para el funcionamiento de LaProgra.
- Deben seguirse las reglas de `iberia_parser.md`.
- Solo el propietario puede editar o eliminar sus eventos importados de Iberia.
- Los cambios manuales deben persistir después de importar un nuevo CSV.
- Estrategia exacta para preservar los cambios manuales después de una nueva importación: marcar con un flag los eventos creados manualmente para diferenciarlos de los creados por la importación.
- Iberia proporciona eventos mes a mes por lo tanto es imposible que se solapen.

### 5.3. Cambio de aerolínea

- El usuario podrá cambiar de aerolínea desde Configuración.
- Al cambiar de aerolínea los eventos almacenados hasta el momento se eliminan pero el resto de los aspectos de la cuenta permanecen intactos. Se consulatará lo siguiente:

### Español

> ¿Estás seguro? Esto eliminará permanentemente los eventos que tenías cargados y se solicitará una nueva importación.

### Inglés

> Are you sure? This will permanently delete the events you loaded and a new upload will be required.



---

## 6. Diseño, accesibilidad e idiomas

- Diseño moderno, elegante y parecido a las aplicaciones nativas de Apple.
- Interfaz tipo dashboard.
- Diseño responsive con prioridad para móviles.
- En pantallas estrechas debe priorizarse la legibilidad del texto de los slabs, aunque sea necesario reducir mucho los márgenes.
- El idioma inicial será español de España.
- El usuario podrá cambiar el idioma a inglés.
- La preferencia de idioma se guardará en su perfil.
- Los textos de la interfaz deben tener la ortografía correcta.
- Las descripciones procedentes de los archivos de códigos (como swiftair_codes.csv y iberia_codes.csv) no deben traducirse.
- Los códigos, rutas y datos originales de los eventos no deben traducirse.
- La interfaz debe aplicar un único criterio razonable de accesibilidad:
  - Buen contraste favoreciendo la estética más que la accesibilidad.
  - La población de usuarios es bastante narrow en el sentido de salud y apreciación de colores.

---

## 7. Autenticación

La autenticación se realizará mediante Supabase Auth.

Debe permitirse:

- Registro mediante correo electrónico y contraseña.
- Inicio de sesión mediante correo electrónico y contraseña.
- Inicio de sesión con Google.
- Mostrar u ocultar la contraseña mientras se escribe.
- Recuperar la contraseña si el usuario la ha olvidado.
- Cerrar sesión.
- Verificar el correo utilizando el flujo estándar de Supabase.
- Aplicar los requisitos mínimos de contraseña configurados en Supabase.
- Solo puede existir una cuenta por dirección de correo electrónico.

No debe permitirse 
- Crear manualmente una segunda cuenta con el mismo correo utilizado previamente mediante Google.
- Indicar "email ya usado con Google" / "email previously used with Google"
- Permitir o impedir la vinculación automática de identidades de Supabase con el mismo correo: quiero decir con esto que no se puede crear una cuenta de Google y una cuenta sin google con el mismo correo.

---

## 8. Primera sesión

Durante el primer inicio de sesión se mostrará un modal obligatorio.

El usuario no podrá acceder al resto de la aplicación hasta completar correctamente el proceso.

El modal solicitará:

- Nombre de usuario.
- Aerolínea.
- Base operativa.
- CSV de Iberia o enlace Webcal de Swiftair, según la aerolínea.

El modal debe mostrar las instrucciones definidas en `upload_instructions.md`.

La importación inicial es obligatoria y debe finalizar correctamente antes de permitir el acceso a la aplicación.

### 8.1. Nombre de usuario

- El nombre de usuario debe ser único. No utilizar un place holder con el nombre de usuario de la cuenta de google o el prefijo del mail.
- La búsqueda de amigos utilizará este nombre.
- La comparación para determinar su unicidad no debe distinguir entre mayúsculas y minúsculas.
- Por ejemplo, `Pedro`, `pedro` y `PEDRO` deben considerarse el mismo nombre.
- Longitud mínima y máxima: longitud máxima 10 caracteres y sin longitud mínima.
- Caracteres permitidos: todos
- Permitir espacios en el nombre: no

### 8.2. Aerolínea

- Inicialmente solo podrán seleccionarse Iberia y Swiftair.
- La selección determinará el método de importación.
- El usuario podrá cambiar de aerolínea desde Configuración.

### 8.3. Base operativa

- La base se introducirá mediante un campo de texto.
- El valor debe validarse contra la columna `IATA` de `airports.csv`.
- La validación no debe distinguir entre mayúsculas y minúsculas.
- Tras escribir un código válido:
  - Se mostrará el nombre de la ciudad correspondiente.
  - El campo mostrará visualmente que el valor es válido.
- No se permitirá continuar con una base que no exista en `airports.csv`.

---

## 9. Calendario

El calendario es el elemento principal de la aplicación.

- Mostrará un mes completo.
- La semana comenzará en lunes.
- El grid tendrá las columnas y filas necesarias para representar todos los días del mes.
- Los eventos aparecerán como slabs monocolor.
- Cada slab mostrará o bien una sigla o sino una ruta como `IATA1-IATA2`.
- La altura de la fila de celdas podrá aumentar para mostrar todos sus slabs.
- No deben ocultarse eventos detrás de un indicador del tipo `+3`, incluso si la pantalla es estrecha.
- Los colores de usuarios y amigos deben mantener un contraste suficiente con el texto.

### 9.1. Detalle de eventos

Al pulsar un slab se mostrará:

- Nombre del usuario propietario.
- Texto del slab.
- Ciudades de origen y destino, cuando corresponda, obtenidas mediante `airports.csv`.
- Horario.
- Descripción obtenida de `iberia_codes.csv` o `swiftair_codes.csv`.
- Número de vuelo, cuando exista.
- Si el vuelo es como pasajero pondrá: `Vuelo en situación en Ryanair FR2430` o `Deadhead flight on Ryanair FR2430`
- Acciones permitidas para ese evento.

Los amigos pueden abrir los eventos y consultar toda su información, pero nunca pueden editarlos ni eliminarlos.

### 9.2. Edición y eliminación

- Solo el propietario puede editar o eliminar sus eventos.
- Los eventos importados de Iberia pueden editarse y eliminarse.
- Los eventos importados de Swiftair no pueden editarse ni eliminarse.
- Las actividades insertadas manualmente pueden editarse y eliminarse independientemente de la aerolínea del usuario.
- Los botones de edición y eliminación solo deben mostrarse cuando la operación esté autorizada.

### 9.3. Horarios y zonas horarias

El usuario podrá elegir en Configuración entre:

- Hora local de su base.
- Hora local de origen y destino.
- UTC.

Las conversiones utilizarán la columna `Timezone` de `airports.csv`.

Los eventos que crucen medianoche se mostrarán como un evento normal. La hora de llegada incluirá un superíndice con el número de días transcurridos.

Ejemplo: `20:50 – 00:10⁺¹`

- Zona horaria aplicable a cursos, días libres, simuladores y otros eventos sin aeropuerto: esos eventos tienen que tener el uso horario de la base operativa del usuario.
- Confirmar que el formato horario será siempre de 24 horas: confirmado 24 horas.
- Comportamiento cuando falte la zona horaria de un aeropuerto: no va a faltar.

---

## 10. Amigos

La sección **Amigos** permitirá consultar la programación de otros usuarios dentro del mismo calendario.

Debe permitir:

- Buscar usuarios únicamente por nombre de usuario.
- Enviar solicitudes de amistad.
- Aceptar o rechazar solicitudes recibidas.
- Cancelar solicitudes enviadas.
- Eliminar amigos.
- Elegir individualmente el color utilizado para los slabs de cada amigo.
- Mostrar u ocultar los eventos de cada amigo.
- No habrá un límite establecido de amigos.

Cada amigo se representará mediante un control visual o «galleta» que contendrá:

- Selector de color.
- Nombre de usuario.
- Emoji de avión si está volando en ese momento.
- Botón de menú para eliminarlo.
- Un control para mostrar u ocultar sus eventos.

Cuando un amigo esté oculto, la galleta debe mostrar un estado visual diferente.

Los amigos podrán consultar todos los detalles de los eventos compartidos, pero no podrán acceder a:

- Correo electrónico.
- CSV original.
- Enlace Webcal.
- Configuración.
- Preferencias privadas.
- Datos de autenticación.
- Información privada no necesaria para mostrar su programación.

### 10.1. Vuelo en curso

Se considerará que un usuario está volando cuando:

- Tenga un evento de vuelo.
- La hora actual se encuentre entre la salida y la llegada del evento.
- Se hayan aplicado correctamente las zonas horarias.

Si el vuelo tiene un número de vuelo, al pulsar el emoji del avión se abrirá en una pestaña nueva:

`https://www.flightradar24.com/{flightNumber}`

El enlace debe utilizar `target="_blank"` y `rel="noopener noreferrer"`.

Si el evento no contiene un número de vuelo válido, el emoji no se mostrará.

- Definir exactamente qué parte de la galleta muestra u oculta al amigo: toda la galleta.
- Definir si el emoji se muestra solo durante el tiempo en el aire o desde la hora de salida hasta la hora de llegada programadas: desde la hora de salida a la de llegada programada.
- Confirmar que los formatos importados incluyen el número de vuelo necesario para Flightradar24: no, de hecho eso se explica en los archivos `iberia_parser.md` y `swiftair_parser.md`

---

## 11. Configuración

Un icono de engranaje abrirá el modal de Configuración.

Permitirá:

- Cambiar el nombre de usuario.
- Cambiar el color de los eventos propios mediante un selector libre.
- Cambiar la base operativa.
- Cambiar de aerolínea.
- Subir un nuevo CSV o cambiar el enlace Webcal, según aplique.
- Seleccionar el modo de zona horaria.
- Cambiar el idioma.
- Generar las dos versiones de PDF descritas en `download_pdf.md`.
- Crear actividades según `new_event.md`.
- Cerrar sesión.
- Eliminar la cuenta.

---

## 12. Eliminación de cuenta

La eliminación será inmediata y permanente.

Antes de eliminar la cuenta se mostrará:

### Español

> ¿Estás seguro? Esto eliminará permanentemente tu cuenta y todos los datos asociados a ella en LaProgra.

### Inglés

> Are you sure? This will permanently delete your account and all data associated with it in LaProgra.

La eliminación debe incluir:

- Perfil.
- Eventos importados.
- Actividades manuales.
- Preferencias.
- Solicitudes de amistad enviadas y recibidas.
- Relaciones de amistad.
- Enlace Webcal.
- Datos asociados a importaciones.
- Cuenta de Supabase Auth.
- Cualquier otro dato personal relacionado con el usuario.

La operación debe ejecutarse de forma segura en el servidor y no depender únicamente del navegador.

Mecanismo de confirmación:

- Botones «Cancelar» y «Eliminar».

---

## 13. Seguridad y privacidad

- Activar Row Level Security en todas las tablas que contengan información de usuarios.
- Un usuario solo podrá modificar sus propios datos.
- Los amigos solo podrán consultar los eventos permitidos.
- Los usuarios que no sean amigos no podrán consultar las programaciones.
- El enlace Webcal no podrá consultarse desde el navegador por otros usuarios.
- El CSV original de Iberia nunca se almacenará en Supabase.
- No deben incluirse secretos en el código del cliente.
- Las operaciones administrativas, las sincronizaciones y la eliminación completa de cuentas deben realizarse mediante funciones seguras del servidor.
- Deben validarse todos los archivos, URLs y datos introducidos por el usuario.
- La aplicación debe evitar duplicados y accesos no autorizados.
- Deben aplicarse las reglas adicionales definidas en los archivos especializados.

- Tamaño máximo permitido para CSV: sin limite
- Límite de solicitudes de amistad para evitar abuso: 5 por minuto
- Tiempo máximo de espera y número de reintentos para sincronizar Webcal: 30 segundos, 5 intentos por minuto.

---

## 14. Entrega

Construye la aplicación completa siguiendo este documento y los archivos adjuntos.

- No muestres el código fuente directamente en el chat.
- Genera todos los archivos necesarios.
- Incluye `.env.example` sin secretos reales.
- Incluye instrucciones de:
  - Instalación.
  - Desarrollo local.
  - Configuración de Supabase.
  - Aplicación de migraciones.
  - Configuración de Google OAuth.
  - Configuración de las Edge Functions.
  - Configuración de Cron.
  - Despliegue en Vercel.
  - Configuración del dominio.
- Incluye un archivo `tree.csv` en la raíz.
- `tree.csv` debe mostrar el árbol completo del proyecto.
- No incluyas datos de demostración.
- Incluye las pruebas mínimas necesarias en una lista de comprobaciones manuales.
- Entrega los archivos individualmente en formato `.csv`.
- Indica para cada archivo:
  - Nombre del archivo entregado.
  - Ruta de destino.
  - Nombre y extensión finales.

Ejemplo:

`src_app_page.tsx.csv` → `src/app/page.tsx`

Los archivos de texto deben poder copiarse directamente a su ubicación dentro del proyecto.

### PENDIENTE

- Método de entrega de archivos binarios, como iconos de la PWA, favicon o imágenes: indica donde los quieres en la raiz del proyecto y yo los pondré.
- Forma de dividir la entrega si la cantidad de archivos supera el límite disponible: dame un aviso para poder pedirtelos después.
- Confirmar si se mantiene la entrega mediante archivos individuales o se permite también un ZIP: se permite un zip si con eso puedes darme los archivos en carpetas y subcarpetas con la extensión apropiada, sino damelos individualmente.

---

## 15. Condición para comenzar el desarrollo

Antes de construir la aplicación:

1. Comprueba que todos los archivos especializados estén disponibles.
2. Revisa todos los apartados marcados como **[PENDIENTE]**.
3. Enumera únicamente las decisiones pendientes que sean realmente bloqueantes.
4. No escribas código ni construyas el proyecto hasta que se hayan resuelto todas las dudas que tengas.
