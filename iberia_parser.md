Aquí defino el parser de los contenidos provenientes del CSV de iberia:

## Introducción:

Iberia sube los datos en archivos CSV mensuales y las actividades que hay en un mes no aparecen en otro mes.

El usuario descarga de la intranet de Iberia el CSV y lo sube a LaProgra.

## Requisitos previos:

- El CSV debe anonimizarse en el navegador así que de todas las columnas del CSV SOLO se utilizan: Asunto, Fecha de comienzo, Comienzo, Fecha de finalización, Finalización.

- En principio no debería aparecer problemas de caracteres corrompidos (tildes, ñ etc). Pero existe la posibilidad y hay que tenerlo contemplado.

- Los valores que aparecen en las columnas Fecha de comienzo, Comienzo,Fecha de finalización, Finalización son en LOCAL de MADRID, España.

- El código debe leer lo que dice la columna Asunto del CSV y compararlo con la columna ID de `iberiaCodes.txt`. Si existe un match entonces el slab será el valor correspondiente a esa fila de la columna slab de `iberiaCodes.txt` y la descripción el valor correspondiente a esa fila de la columna description de `iberiaCodes.txt`.

- Si no existe match puede ser por al menos dos motivos:
    - una ruta de vuelo
    - la palabra "firma" aparece en la los valores de la columna "Asunto"

La ruta de vuelo aparecerá con el formato: MAD1200 - BCN1300. Y puede que tenga texto antes y después.

- De ahí hay que extraer el origen y destino IATA, compararlo con airports.csv para obtener el origen y destino en valores de la columna City y así poder mostrarlo en la descripción del evento como "Madrid - Barcelona"

Si el valor de la fila en la columna asunto contiene firma quiere decir que es el evento en el que se inicia una sucesión de vuelos que puede durar uno o varios días. En inglés se llamará "Report for duty". 

- Si aparece firma entonces debe aparecer su valor de "Comienzo" detrás del horario en la descripción del evento:
    - Por ejemplo: 12:00 - 13:00 (firma 11:00)
    - En inglés: 12:00 - 13:00 (report 11:00)

- En principio si no existe match de primeras con `iberiaCodes.txt` solo debe ser por esos dos motivos. Si no hubiera match por otro motivo no contemplado aquí quiero que el valor de la columna "Asunto" se ponga tal cual en la descripción y en el slab aparezca "N/D" o "N/A" en inglés.

- El archivo original nunca debe salir del dispositivo.

- Debe descartarse inmediatamente después del análisis.