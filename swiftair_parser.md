Aquí defino el parser de los contenidos provenientes del enlace webcal de Swiftair:

## Introducción:

Swiftair utiliza la app eCrew para publicar las programaciones de sus tripulantes. eCrew ofrece la posibilidad de exportar los eventos a un calendario del móvil como Google Calendar o Calendar de iOS. A su vez, Calendar de iOS permite exportar un enlace webcal para poder ver los eventos del calendario en cualquier otra aplicación de calendarios. Quiero aprovechar esa funcionalidad para poder importar los eventos de eCrew en LaProgra. Swiftair es una empresa con muchos cambios internos constantes y las programaciones de sus empleados sufren cambios constantemente. Swiftair re-publica la programación de vez en cuando en eCrew y sus eventos se exportan al calendario que elija el usuario (Calendar de iOS de ahora en adelante). Muchas veces los eventos pasados se borran en esa actualización que publica Swiftair. 

## Requisitos previos:

- Se permite guardar el enlace Webcal en Supabase para sincronizaciones posteriores.
- Se impide que los usuarios puedan ver el enlace webcal de sus amigos en la consola para desarrolladores del navegador
- Los eventos de Swiftair deben sincronizarse automáticamente. Para ello hay que dar de alta un CRON JOB y una edge function de Supabase que actualice una vez por minuto los eventos futuros provenientes del enlace webcal.
- Cuando el usuario introduce el enlace webcal por primera vez en el onboarding deben parsearse todos los eventos provenientes del mes en curso y en adelante.

A continuación muestro un ejemplo del contenido del descargable del enlace webcal:

BEGIN:VEVENT
CREATED:20260303T225949Z
DESCRIPTION:Reporting time : 2100\n4646  - HAJ  (A2141) - LGG  (E2254)\n*
  All times in UTC\n\n--- Created by the AIMS eCrew app ---
DTEND;TZID=Europe/Brussels:20260303T235200
DTSTAMP:20260303T225951Z
DTSTART;TZID=Europe/Brussels:20260303T220000
LAST-MODIFIED:20260303T225949Z
LOCATION:(2145Z-2252Z) HAJ
SEQUENCE:0
SUMMARY:4646  HAJ-LGG
UID:012B6285-F07B-4810-9239-E0E09D9A1020
X-APPLE-CREATOR-IDENTITY:aero.aims.eCrew
X-APPLE-CREATOR-TEAM-IDENTITY:CRVQTJ2BFS
END:VEVENT


- Del campo DTSTART podemos extraer el uso horario (TZID) y la fecha y hora de comienzo del evento
- Del campo DTEND lo mismo pero para el final del evento.
- Del campo SUMMARY podemos extraer código de actividad o el número de vuelo y la ruta.
- Del campo LOCATION podemos extraer los horarios en UTC del vuelo. 
- Del campo DESCRIPTION podemos extraer la hora de "firma" gracias a que pone "Reporting time: 2100" y está en UTC.
- Siempre que aparezca ese "Reporting time" en el campo DESCRIPTION aparecerá la hora de firma en al descripción del evento.

El contenido del slab y de la descripción del evento provienen de buscar un match entre el campo SUMMARY y la columna ID de `swiftairCodes.txt`.

Aquí tienes un ejemplo de un evento webcal asociado a un ID de `swiftairCodes.txt`:

BEGIN:VEVENT
CREATED:20260725T103900Z
DESCRIPTION:Leave / Vacaciones                      - V   \nFull day\nLoc
 ation: LIS\n* All times in UTC\n\n--- Inserted by the AIMS eCrew app ---
 
DTEND;VALUE=DATE:20261018
DTSTAMP:20260725T103901Z
DTSTART;VALUE=DATE:20261017
LAST-MODIFIED:20260725T103900Z
LOCATION:LIS 
SEQUENCE:0
SUMMARY:V    Leave / Vacaciones                      
UID:1795C28E-DDA5-43E6-B292-672D4B677C03
X-APPLE-CREATOR-IDENTITY:aero.aims.eCrew
X-APPLE-CREATOR-TEAM-IDENTITY:CRVQTJ2BFS
END:VEVENT

### Flujo de detección de eventos
1. Comprobar el campo SUMMARY y distinguir entre vuelos y todo lo demás. 
    Para ello comprobar si las letras de la primera palabra contienen numeros. Eso es el indicativo de que el evento es un vuelo puesto que ningún ID de `swiftairCodes.txt` contiene números. Ej de vuelos. `SUMMARY:4646  HAJ-LGG` o `SUMMARY:GRD1319  DUS-CGN` o `SUMMARY:IB539  MAD-LIS`
    Tener en consideración que puede haber errores tipograficos como que aparezca un punto o un espacio entre las letras y los números.
2. Si no cumple la lógica de ser vuelo, buscar un match contra los ID de `swiftairCodes.txt`. 

3. Si no es un vuelo pero tampoco hay match entonces pondremos lo que diga el campo SUMMARY tal cual.
    En el slab pondremos lo que diga SUMMARY antes del primer espacio y en la descripción del evento pondremos lo que haya después del espacio.

### Nota: 
- Aquellos números de vuelo que solo tengan numeros o numeros y después una o varias letras tendrán que ser complementados por las letras `WT`. Por ejemplo. `SUMMARY:4646  HAJ-LGG` el número de vuelo será `WT4646` o `SUMMARY:721P MAD-BCN` el número de vuelo será `WT721P`
- Si el numero de vuelo detectado es de tipo ICAO (tres letras, ej. AEA1145) habrá que traducirlo a IATA buscando el prefijo ICAO y cambiandolo por su equivalente IATA. Para eso hay que usar las columnas ICAO e IATA del archivo `airlines.txt`
- El número de vuelo sirve para poder determinar si el vuelo es en situación (deadhead) o es operado por el tripulante. Por ello quiero que exista un array o similar de codigos IATA hardcodeado en el codigo para determinar aquellos operados por los tripulantes de Swiftair ya que operan vuelos para diferentes compañias. De momento serán `OPERATED_FLIGHTS=['WT','QY']` todos los demás IATA serán vuelos en situación (deadhead).
- Hay veces que los tripulantes de Swiftair son situados (deadhead) por carretera. Esos "números de vuelo" serán aquellos que contengan las letras `GRD`. Por ejemplo: `SUMMARY:GRD1319  DUS-CGN`. En este caso la descripción del evento será `En coche`/`Ground transport`. 