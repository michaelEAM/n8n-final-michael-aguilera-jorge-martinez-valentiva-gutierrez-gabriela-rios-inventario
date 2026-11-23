# Bitácora de la clase 11

## Revisión integral del proyecto y pendientes para la sustentación

**Curso:** Automatización inteligente con n8n  
**Tema:** Pruebas y confiabilidad de flujos con IA

Completen esta bitácora con la versión actual de su proyecto. Revisen el proceso completo, incluidos los workflows relacionados, las ramas y los destinos de sus operaciones. Amplíen las tablas según lo necesiten.

Hoy documentamos la revisión del diseño y los pendientes. Las pruebas formales y sus evidencias corresponden a la sustentación. Localizar una validación en un nodo no demuestra todavía que funciona en todos los casos.

## 1. Identificación

- **Fecha de la clase:** [2026-11-09]
- **Proyecto:** [Inventario: revisión de stock y alertas de reposición (n8n + Supabase + Gemini + Gmail)]
- **Integrantes presentes:** [Gabriela Rios, Valentina Gutierrez, Michael Aguilera, Jorge Martinez]
- **Versión revisada:** [Versión, fecha del JSON o referencia del repositorio]
- **Workflows relacionados:** [`Preparar_Producto.json`, `Manejador_Errores.json` y `Preparar_Supabase.sql`]
- **Diagrama revisado:** []

## 2. Recorrido completo del proyecto

Describan brevemente cómo ingresa una solicitud, qué información utiliza el proceso, dónde interviene la IA y cómo termina. Incluyan las rutas alternativas y las operaciones que consultan o modifican otros sistemas.

**Descripción del recorrido:**

Nuestro proyecto consiste en automatizar la revisión del inventario utilizando n8n. El proceso comienza cuando se recibe una solicitud mediante un Webhook o cuando ejecutamos el flujo manualmente para realizar pruebas.

Primero, el sistema consulta la base de datos de Supabase para buscar el producto solicitado y obtener información como su nombre, stock actual y stock mínimo. Después, verifica que el producto exista y que sus datos sean válidos.

Con esta información, el sistema calcula las unidades faltantes y clasifica la prioridad de reposición en alta, mediana o baja. Si el producto necesita reposición, se registra una alerta en Supabase y se utiliza inteligencia artificial con Google Gemini para redactar un correo de notificación, que posteriormente se envía mediante Gmail.

Si el producto tiene suficiente inventario, no se genera ninguna alerta. También existen rutas alternativas para productos no encontrados, datos incorrectos y problemas con la IA o el envío del correo.

Finalmente, el workflow reúne los resultados y devuelve una respuesta con el estado del producto y las acciones realizadas. La automatización permite identificar productos que necesitan reposición sin tener que revisar manualmente cada registro.

Registren las etapas que realmente tiene su proyecto. Agreguen filas cuando existan varias consultas, decisiones o acciones. Si una etapa no aplica, expliquen por qué. Si hace falta para cumplir el objetivo, regístrenla como pendiente.

| Etapa                                       | Workflow y nodo o tramo                                                           | Qué debe garantizar                                                                                                 | Destino o siguiente etapa                                                                           |
| ------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Recepción de la solicitud                   | `Webhook` / `When clicking 'Execute workflow'`                                    | Recibir el nombre del producto que se desea consultar, ya sea por solicitud HTTP o mediante pruebas manuales.       | HTTP Request                                                                                        |
| Consulta de fuentes                         | `HTTP Request` – Supabase                                                         | Consultar la tabla `stock_productos` y obtener la información del producto.                                         | ¿Encontró producto?                                                                                 |
| Validación y preparación de datos           | `¿Encontró producto?` → `Preparar producto (subworkflow)` → `¿Datos válidos?`     | Verificar que el producto exista y que los datos recibidos sean válidos para procesarlos.                           | `Transformar y calcular` o rutas de error                                                           |
| Cálculo de inventario                       | `Transformar y calcular` → `Asignar prioridad`                                    | Calcular las unidades faltantes y asignar una prioridad según el stock actual y el mínimo.                          | If                                                                                                  |
| Decisiones y reglas del negocio             | `If` → `Switch` → `Prioridad alta` / `Prioridad media` / `No requiere reposición` | Clasificar el producto según su inventario y determinar si necesita reposición.                                     | Registrar alerta o finalizar sin reposición                                                         |
| Operaciones en servicios externos           | Registrar alerta en Supabase                                                      | Registrar o actualizar la alerta en la tabla `alertas_reposicion`, evitando duplicados por identificador de alerta. | Redactar correo con IA                                                                              |
| Interpretación o respuesta de IA            | `Redactar correo con IA` + `Modelo Gemini` + `Contrato de respuesta`              | Generar un asunto y un mensaje de alerta utilizando únicamente los datos del inventario, sin inventar información.  | Armar correo                                                                                        |
| Preparación y envío de notificación         | `Armar correo` → `Enviar correo de alerta`                                        | Preparar el correo con los datos reales y enviarlo mediante Gmail a la persona encargada.                           | Restaurar alerta                                                                                    |
| Confirmación de operaciones                 | `Restaurar alerta` → `Confirmar registro`                                         | Recuperar la información de la alerta e indicar si se registró y si se envió el correo.                             | Unir resultados                                                                                     |
| Respuesta y efecto final                    | `Unir resultados` → `Respond to Webhook`                                          | Reunir los resultados de las distintas rutas y devolver una respuesta HTTP con el estado del proceso.               | Finalización                                                                                        |
| Tratamiento de fallos: producto inexistente | Producto no encontrado                                                            | Informar que el producto no existe y solicitar revisar su nombre.                                                   | Unir resultados                                                                                     |
| Tratamiento de fallos: datos incorrectos    | Datos no válidos                                                                  | Evitar procesar información incorrecta e indicar que se debe corregir el registro.                                  | Unir resultados                                                                                     |
| Tratamiento de fallos: IA o correo          | `Armar correo` / `Enviar correo de alerta`                                        | Utilizar una plantilla fija cuando la IA no devuelve una respuesta válida y registrar si el envío del correo falla. | `Restaurar alerta` → `Confirmar registro`                                                           |
| Tratamiento de fallos: registro externo     | Registrar alerta en Supabase                                                      | Reintentar el registro hasta tres veces cuando falla la solicitud.                                                  | Continuar si el registro se completa; el fallo definitivo no tiene una ruta específica configurada. |

## 3. Revisión de situaciones por etapa

Recorran todas las etapas anteriores y consideren las siguientes situaciones donde sean pertinentes:

- Operación normal con datos válidos.
- Resultado negativo válido, como ausencia de disponibilidad o rechazo por una regla.
- Datos incompletos, inválidos, ambiguos o contradictorios.
- Valores antes, en y después de los límites declarados.
- IA o servicio externo no disponible, o respuesta no utilizable.
- Interrupción después de completar solo una parte del proceso.
- Solicitud repetida, posible duplicado o cambio de contexto.
- Acción no permitida o uso de información que no corresponde a la solicitud.

No se establece un número fijo de casos. Una situación puede necesitar varias filas si afecta diferentes etapas o reglas. Marquen lo que no aplica y justifiquen la decisión.

### Registro de la revisión

En **Estado de revisión**, utilicen una de estas opciones:

- **Protección localizada:** indiquen dónde está y qué contempla. Su comportamiento aún requiere comprobación.
- **Brecha identificada:** falta una protección, está incompleta o su comportamiento no está claro.
- **No aplica justificado:** expliquen por qué la situación no corresponde al proceso.

| ID    | Etapa y situación concreta                                                          | Resultado esperado y efecto que debe evitarse                              | Protección actual y ubicación                                                                   | Estado de revisión y justificación                                                                                                      |
| ----- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| RC-01 | Recepción de una solicitud normal con un producto válido.                           | El sistema debe recibir el producto e iniciar su consulta.                 | Nodos `Webhook` y `HTTP Request`.                                                               | Protección localizada: existe una ruta de recepción y consulta. Falta comprobarla en n8n.                                               |
| RC-02 | Solicitud sin nombre de producto.                                                   | Debe solicitar un nombre válido y evitar consultar todo el inventario.     | `HTTP Request` reemplaza el nombre ausente por una cadena vacía.                                | Brecha identificada: no existe una validación inicial que rechace solicitudes vacías.                                                   |
| RC-03 | Solicitud ambigua que coincide con varios productos.                                | Debe identificar el producto correcto y evitar alertas de otros productos. | `HTTP Request` utiliza una búsqueda parcial con `ilike`.                                        | Brecha identificada: no se exige una coincidencia única antes de procesar los resultados.                                               |
| RC-04 | Producto que no existe en Supabase.                                                 | Informar que el producto no fue encontrado, sin generar alertas.           | `¿Encontró producto?` → `Producto no encontrado`.                                               | Protección localizada: existe una ruta para productos inexistentes.                                                                     |
| RC-06 | Producto existente con datos válidos.                                               | Continuar con los cálculos utilizando la información del inventario.       | `Preparar producto (subworkflow)` → `¿Datos válidos?`.                                          | Protección localizada: se comprueba el campo `datos_validos`, pero falta revisar el subworkflow.                                        |
| RC-07 | Producto con datos incompletos o inválidos.                                         | Detener el procesamiento y solicitar corregir el registro.                 | `¿Datos válidos?` → `Datos no válidos`.                                                         | Protección localizada: existe una ruta de rechazo; falta comprobar qué datos considera inválidos.                                       |
| RC-08 | Falla durante la ejecución del subworkflow.                                         | Evitar continuar con datos sin preparar y reportar el error.               | `Preparar producto (subworkflow)` espera su resultado, sin recuperación explícita.              | Brecha identificada: no hay una ruta específica para la indisponibilidad del subworkflow.                                               |
| RC-09 | Stock antes, en y después del mínimo: 9, 10 y 11 unidades frente a un mínimo de 10. | Clasificar correctamente las prioridades sin generar alertas innecesarias. | `Transformar y calcular`, `Asignar prioridad`, `If` y `Switch`.                                 | Protección localizada: las expresiones producen Alta, Mediana y Baja, respectivamente.                                                  |
| RC-10 | Producto agotado: stock 0, mínimo 10.                                               | Identificar el faltante de 10 unidades y generar una alerta urgente.       | `Transformar y calcular` → `Prioridad alta`.                                                    | Protección localizada: las reglas contemplan el stock agotado.                                                                          |
| RC-11 | Stock y mínimo iguales a cero.                                                      | Evitar alertas innecesarias, según las reglas que defina el negocio.       | `Asignar prioridad` clasifica la igualdad como Mediana.                                         | Brecha identificada: puede generar una alerta con cero unidades faltantes; falta definir el comportamiento deseado.                     |
| RC-12 | Stock superior al mínimo.                                                           | Finalizar sin registrar alertas ni enviar correos.                         | `If` → `No requiere reposición`.                                                                | Protección localizada: existe una ruta para el resultado negativo válido.                                                               |
| RC-13 | Registro normal de una alerta en Supabase.                                          | Guardar el producto, faltante, prioridad y acción correspondiente.         | `Registrar alerta en Supabase` mediante HTTP POST.                                              | Protección localizada: está configurado el registro de la alerta. Falta comprobar la escritura real.                                    |
| RC-14 | Fallo temporal al registrar una alerta.                                             | Reintentar la operación y evitar perder la alerta.                         | `Registrar alerta en Supabase` tiene tres intentos con espera de dos segundos.                  | Protección localizada: existen reintentos, pero falta comprobar qué sucede después del último fallo.                                    |
| RC-15 | Solicitud repetida del mismo producto durante el mismo día.                         | Evitar registros y notificaciones duplicados.                              | `Registrar alerta en Supabase` utiliza `alerta_id` y `on_conflict`.                             | Brecha identificada: se intenta reutilizar la alerta del día, pero no se evita el reenvío del correo.                                   |
| RC-16 | Cambio de stock después de consultar el producto.                                   | Evitar enviar una alerta basada en información desactualizada.             | No existe una segunda consulta antes de registrar la alerta.                                    | Brecha identificada: el stock puede cambiar durante la ejecución.                                                                       |
| RC-17 | Gemini genera correctamente el asunto y el cuerpo.                                  | Preparar una notificación clara basada en el inventario.                   | `Redactar correo con IA` → `Contrato de respuesta` → `Armar correo`.                            | Protección localizada: se comprueba la estructura de la respuesta generada.                                                             |
| RC-18 | Gemini no responde o devuelve información no utilizable.                            | Generar una notificación alternativa sin depender de la IA.                | `Redactar correo con IA` tiene salida de error y `Armar correo` utiliza una plantilla fija.     | Protección localizada: existe un mecanismo alternativo que debe comprobarse en ejecución.                                               |
| RC-19 | La IA inventa un proveedor, una fecha o una cantidad incorrecta.                    | Evitar comunicaciones con información que no provenga del inventario.      | Instrucciones de `Redactar correo con IA` y tabla de datos reales en `Armar correo`.            | Brecha identificada: se controla el formato, pero no se verifica automáticamente la veracidad del texto redactado.                      |
| RC-20 | La IA produce un asunto que excede los 80 caracteres solicitados.                   | Respetar el límite establecido para el asunto.                             | `Armar correo` permite hasta 120 caracteres.                                                    | Brecha identificada: la validación no coincide con el límite establecido en las instrucciones de la IA.                                 |
| RC-21 | Envío correcto de una notificación por Gmail.                                       | Enviar el aviso y conservar el resultado de la operación.                  | `Enviar correo de alerta` → `Restaurar alerta`.                                                 | Protección localizada: se utiliza el identificador del mensaje para determinar si se envió.                                             |
| RC-22 | Gmail no está disponible o falla el envío.                                          | Conservar la alerta y registrar que el correo no se envió.                 | `Enviar correo de alerta` continúa ante el error y `Restaurar alerta` calcula `correo_enviado`. | Protección localizada: permite continuar e indicar el fallo, pero no reintenta el envío.                                                |
| RC-23 | Interrupción después de guardar la alerta, pero antes de enviar el correo.          | Recuperar el envío pendiente sin duplicar operaciones.                     | No hay una cola de correos pendientes ni una recuperación específica.                           | Brecha identificada: la alerta podría quedar registrada sin notificación.                                                               |
| RC-24 | Una persona no autorizada consulta el Webhook.                                      | Evitar acceso no autorizado a la información del inventario.               | El nodo `Webhook` no presenta autenticación configurada en el JSON.                             | Brecha identificada: falta definir un mecanismo de autenticación para las solicitudes.                                                  |
| RC-25 | Respuesta final de las diferentes rutas del proceso.                                | Entregar el estado correspondiente sin perder resultados.                  | `Unir resultados` → `Respond to Webhook`.                                                       | Protección localizada: las rutas están conectadas a una respuesta común. Falta comprobar cada una.                                      |
| RC-26 | Solicitud de realizar compras o modificar directamente el stock.                    | No efectuar operaciones fuera del alcance del proyecto.                    | El workflow no incluye nodos de compra ni de actualización de existencias.                      | No aplica justificado: el objetivo actual es consultar inventario, registrar alertas y enviar notificaciones, no ejecutar reposiciones. |

**Etapas o situaciones que todavía faltan por revisar:**

- **Zona horaria de `alerta_id`:** usa `$now.toFormat('yyyy-MM-dd')`; si n8n y Supabase usan zonas distintas, la «fecha» de la alerta puede cambiar cerca de la medianoche.
- **Cambio del stock entre la consulta y el registro de la alerta** y solicitudes simultáneas del mismo producto.
- **Límites de cuota y tasa de Gemini y Gmail** frente a consultas con muchos productos.


## 4. Casos previstos para comprobar el comportamiento

Relacionen los casos con la revisión anterior. Usen datos ficticios concretos y condiciones que otra persona pueda reproducir. Definan el criterio antes de ejecutar.

Dupliquen la ficha para tantos casos como necesite el proyecto. Los identificadores son consecutivos y no limitan la cantidad.

### Caso CP-01: Consulta de un producto válido

- **Revisión asociada:** RC-01 y RC-06.
    
- **Workflow y etapas involucradas:** `Webhook` → `HTTP Request` → `¿Encontró producto?` → `Preparar producto (subworkflow)` → `¿Datos válidos?`.
    
- **Qué se busca comprobar:** Que el sistema reciba el nombre de un producto existente, consulte correctamente Supabase y valide su información.
    
- **Condiciones iniciales:** Existe en Supabase el producto `Puntada de piel`, con stock de 4 unidades y mínimo de 10. Todos sus campos obligatorios están completos.
    
- **Entrada exacta:** "producto": "Peluche Stitch"
    
- **Resultado esperado:** El sistema encuentra el producto, valida sus datos y continúa hacia el cálculo de inventario.
    
- **Criterio de aceptación:** Se recupera el producto correcto, sus datos son válidos.
    
- **Efecto que no debe ocurrir:** Procesar un producto distinto o continuar con información incorrecta.
    
- **Evidencia que se buscará:** Historial de ejecución de n8n, datos recuperados de Supabase y resultados de validación.
    
- **Cómo se preparará de forma controlada:** Crear o verificar el producto  y ejecutar una solicitud de prueba.
    
- **Resultado observado:** valores devueltos por Supabase y resultado de validación.
    
- **Conclusión de la prueba:** aprobada.
    
- **Evidencia de ejecución:**
- [Evidencia 1](imagenes/20261123095203.png)
- [Evidencia 2](imagenes/20261123095541.png)

    

### Caso CP-02: Solicitud sin nombre de producto

- **Revisión asociada:** RC-02.
    
- **Workflow y etapas involucradas:** `Webhook` → `HTTP Request` → validación de solicitud.
    
- **Qué se busca comprobar:** Que el sistema rechace solicitudes sin nombre de producto antes de realizar consultas.
    
- **Condiciones iniciales:** Workflow disponible y conexión con Supabase habilitada.
    
- **Entrada exacta:** `{"producto":""}`
    
- **Resultado esperado:** El sistema informa que debe proporcionarse un nombre válido y detiene la consulta.
    
- **Criterio de aceptación:** No se consulta todo el inventario ni se generan alertas.
    
- **Efecto que no debe ocurrir:** Recuperar múltiples productos por una búsqueda vacía o generar alertas incorrectas.
    
- **Evidencia que se buscará:** Entrada del Webhook, nodos ejecutados y respuesta del sistema.
    
- **Cómo se preparará de forma controlada:** Enviar una solicitud con el campo `producto` vacío desde el Webhook de prueba.
    
- **Resultado observado:** Fallida.
    
- **Conclusión de la prueba:** Datos invalidos.
    
- **Evidencia de ejecución:

- [Evidencia 3](imagenes/20261123095831.png)
- [Evidencia 4](imagenes/20261123100143.png)


### Caso CP-03: Consulta de producto inexistente

- **Revisión asociada:** RC-04.
    
- **Workflow y etapas involucradas:** `HTTP Request` → `¿Encontró producto?` → `Producto no encontrado` → `Unir resultados`.
    
- **Qué se busca comprobar:** Que el workflow gestione productos inexistentes sin generar alertas.
    
- **Condiciones iniciales:** El producto `Producto XYZ999` no existe en Supabase.
    
- **Entrada exacta:** `{"producto":"Producto XYZ999"}`
    
- **Resultado esperado:** El sistema informa que no encontró el producto solicitado.
    
- **Criterio de aceptación:** Se ejecuta la ruta `Producto no encontrado` y no se generan registros de alerta.
    
- **Efecto que no debe ocurrir:** Procesar otro producto o generar una alerta de reposición.
    
- **Evidencia que se buscará:** Respuesta de Supabase, ruta ejecutada y respuesta final.
    
- **Cómo se preparará de forma controlada:** Verificar que el producto no exista y ejecutar la consulta.
    
- **Resultado observado:** Producto no encontrado.
    
- **Conclusión de la prueba:** Consulto a la base de datos y como no lo encontró genero falso.
    
- **Evidencia de ejecución:** 

- [Evidencia 5](imagenes/20261123100315.png)
- [Evidencia 6](imagenes/20261123100426.png)
- [Evidencia 7](imagenes/20261123100457.png)

    

### Caso CP-04: Producto con nombre mal escrito

- **Revisión asociada:** RC-07.
    
- **Workflow y etapas involucradas:** `Preparar producto (subworkflow)` → `¿Datos válidos?` → `Datos no válidos`.
    
- **Qué se busca comprobar:** Que el sistema detecte información incompleta y evite cálculos incorrectos.
    
- **Condiciones iniciales:** Existe el producto ficticio `Producto incompleto`.
- 
- **Entrada exacta:** `{"producto":"Peluche Mario"}`
    
- **Resultado esperado:** El sistema identifica que falta un dato obligatorio y detiene el procesamiento.
    
- **Criterio de aceptación:** Se ejecuta la ruta de datos inválidos y no se generan alertas.
    
- **Efecto que no debe ocurrir:** Interpretar un dato nulo como cero o generar cálculos incorrectos.
    
- **Evidencia que se buscará:** Datos del producto, salida del subworkflow y decisión de validación.
    
- **Cómo se preparará de forma controlada:** Crear un registro ficticio incompleto o simular su respuesta en una copia del workflow.
    
- **Resultado observado:** Invalido.
    
- **Conclusión de la prueba:** Fallido.
    
- **Evidencia de ejecución:** 

- [Evidencia 8](imagenes/20261123102823.png)
- [Evidencia 9](imagenes/20261123102948.png)

    

### Caso CP-05: Producto agotado

- **Revisión asociada:** RC-10.
    
- **Workflow y etapas involucradas:** `Transformar y calcular` → `Asignar prioridad` → `If` → `Switch` → `Prioridad alta`.
    
- **Qué se busca comprobar:** Que el workflow identifique correctamente un producto agotado y determine la necesidad urgente de reposición.
    
- **Condiciones iniciales:** Existe `Producto agotado`, con stock actual de 0 unidades y mínimo de 10.
    
- **Entrada exacta:** `{"producto":"Camiseta Roja"}`
    
- **Resultado esperado:** Se calculan 10 unidades faltantes y se asigna prioridad Alta.
    
- **Criterio de aceptación:** El faltante es exactamente 10 y el producto continúa por la ruta de prioridad Alta.
    
- **Efecto que no debe ocurrir:** Clasificar el producto como disponible o asignar una prioridad inferior.
    
- **Evidencia que se buscará:** Cálculo del faltante, prioridad asignada y ruta seleccionada.
    
- **Cómo se preparará de forma controlada:** Crear un producto ficticio con stock 0 y mínimo 10 y ejecutar la consulta.
    
- **Resultado observado:** Geenero la acción de reponer inmediatamente 
    
- **Conclusión de la prueba:** Aproba.
    
- **Evidencia de ejecución:** 

- [Evidencia 10](imagenes/20261123104951.png)
- [Evidencia 11](imagenes/20261123104719.png)
- [Evidencia 12](imagenes/20261123105140.png)

### Caso CP-06: Stock superior al mínimo

- **Revisión asociada:** RC-12.
    
- **Workflow y etapas involucradas:** `Transformar y calcular` → `Asignar prioridad` → `If` → `No requiere reposición`.
    
- **Qué se busca comprobar:** Que el sistema reconozca cuando un producto tiene existencias suficientes y evite generar alertas.
    
- **Condiciones iniciales:** Existe `Producto disponible`, con stock actual de 50 unidades y mínimo de 10.
    
- **Entrada exacta:** `{"producto":"Camisa Roja"}`
    
- **Resultado esperado:** El sistema determina que no requiere reposición y finaliza sin registrar una alerta.
    
- **Criterio de aceptación:** Se calculan 0 unidades faltantes y no se crean registros ni se envían correos de reposición.
    
- **Efecto que no debe ocurrir:** Generar alertas o correos innecesarios.
    
- **Evidencia que se buscará:** Resultado del cálculo, ruta seleccionada y ausencia de nuevas alertas.
    
- **Cómo se preparará de forma controlada:** Crear un producto con stock superior al mínimo y comprobar los registros antes y después de ejecutar.
    
- **Resultado observado:** Aprobado.
    
- **Conclusión de la prueba:** Cumlple.
    
- **Evidencia de ejecución:** 

- [Evidencia 13](imagenes/20261123105344.png)
- [Evidencia 14](imagenes/20261123105736.png)


![[Pasted image 20261123105830.png]]
### Caso CP-07: Registro correcto de una alerta

- **Revisión asociada:** RC-13.
    
- **Workflow y etapas involucradas:** `Switch` → `Registrar alerta en Supabase` → `Redactar correo con IA`.
    
- **Qué se busca comprobar:** Que el workflow registre correctamente una alerta de reposición en Supabase.
    
- **Condiciones iniciales:** Existe `Producto alerta prueba`, con stock 3 y mínimo 10. 
    
- **Entrada exacta:** "producto": "Peluche Stitch"
    
- **Resultado esperado:** El sistema registra una alerta con el producto,  prioridad Alta.
    
- **Criterio de aceptación:** Se crea un registro en `alertas_reposicion` con los datos correctos, sin duplicaciones, y el proceso continúa hacia la preparación del correo.
    
- **Efecto que no debe ocurrir:** Guardar cantidades incorrectas, duplicar registros o perder la información de la alerta.
    
- **Evidencia que se buscará:** Resultado del nodo HTTP POST y registro creado en Supabase.
    
- **Cómo se preparará de forma controlada:** Utilizar un producto ficticio, verificar el estado inicial de la tabla y comparar los registros después de ejecutar.
    
- **Resultado observado:** Notificación en el correo .
    
- **Conclusión de la prueba:** aprobada, informa mediante una alerta al correo .
    
- **Evidencia de ejecución:** 

- [Evidencia 15](imagenes/20261123102350.png)
- [Evidencia 16](imagenes/20261123101833.png)
- [Evidencia 17](imagenes/20261123102123.png)


### Caso CP-08: Solicitud repetida del mismo producto

- **Revisión asociada:** RC-15.
    
- **Workflow y etapas involucradas:** `Webhook` → `HTTP Request` → `Registrar alerta en Supabase` → `Enviar correo de alerta`.
    
- **Qué se busca comprobar:** Que dos solicitudes del mismo producto durante el mismo día no generen registros ni notificaciones duplicadas.
    
- **Condiciones iniciales:** Existe `Producto repetido`, con stock 4 y mínimo 10. La tabla de alertas no contiene registros previos para este producto en el día de prueba.
    
- **Entrada exacta:** Ejecutar dos veces `{"producto":"Peluche Stitch"}` durante el mismo día.
    
- **Resultado esperado:** El sistema conserva una sola alerta correspondiente al producto y evita enviar un segundo correo innecesario.
    
- **Criterio de aceptación:** Existe un único registro de alerta para el producto y la fecha, y se genera como máximo una notificación.
    
- **Efecto que no debe ocurrir:** Crear alertas duplicadas o enviar varios correos por la misma situación.
    
- **Evidencia que se buscará:** Historial de ambas ejecuciones, registros en Supabase y correos enviados.
    
- **Cómo se preparará de forma controlada:** Utilizar un producto ficticio, ejecutar dos solicitudes consecutivas y comparar registros y notificaciones. Usar un destinatario de prueba.
    
- **Resultado observado:** Aprobado.
    
- **Conclusión de la prueba:** No genera duplicado.
    
- **Evidencia de ejecución:** 
    

### Caso CP-09: Fallo de Gemini y generación alternativa del correo

- **Revisión asociada:** RC-18.
    
- **Workflow y etapas involucradas:** `Redactar correo con IA` → salida de error → `Armar correo` → `Enviar correo de alerta`.
    
- **Qué se busca comprobar:** Que el workflow pueda generar una notificación válida aunque Gemini no responda o devuelva información inutilizable.
    
- **Condiciones iniciales:** Se dispone de una copia del workflow en la que puede simularse un fallo de Gemini.
    
- **Entrada exacta:** `{"producto":"Peluche Stitch"}` con fallo controlado del nodo de IA.
    
- **Resultado esperado:** El sistema utiliza una plantilla fija para construir el correo con los datos reales,
    
- **Criterio de aceptación:** Se genera un correo válido sin depender de Gemini, con información correcta y sin campos vacíos.
    
- **Efecto que no debe ocurrir:** Perder la notificación, inventar información o enviar un mensaje incompleto.
    
- **Evidencia que se buscará:** Error simulado de Gemini, salida del nodo `Armar correo` y contenido final del mensaje.
    
- **Cómo se preparará de forma controlada:** Duplicar el workflow, simular una respuesta fallida de Gemini y verificar el uso de la plantilla alternativa, enviando únicamente a un correo de prueba.
    
- **Resultado observado:** Fallo .
    
- **Conclusión de la prueba:** Aviso generado con plantilla fija porque la IA no entregó una respuesta utilizable.
    
- **Evidencia de ejecución:** .

- [Evidencia 18](imagenes/20261123113733.png)

    



## 5. Pendientes para la sustentación

Incluyan cada brecha o ajuste necesario, su responsable y la manera de comprobarlo. Si faltan etapas por revisar, también deben aparecer aquí.

Justifiquen la prioridad por su consecuencia: una operación indebida, pérdida de información o confirmación falsa requiere atención distinta de un problema de redacción. La prioridad no elimina los demás pendientes.

| ID   | Revisión asociada                               | Pendiente y consecuencia de no resolverlo                                                                                 | Prioridad y motivo                                                      | Responsable             | Criterio de cierre y caso asociado                                                                  | Estado                   |
| ---- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------- | --------------------------------------------------------------------------------------------------- | ------------------------ |
| P-01 | RC-24                                           | Implementar autenticación en el Webhook. Sin ella, una persona no autorizada podría consultar información del inventario. | Alta: riesgo de acceso no autorizado.                                   | Desarrollo              | [Qué debe quedar resuelto y cómo se comprobará]                                                     | [Pendiente / En proceso] |
| ID   | Revisión asociada                               | Pendiente y consecuencia de no resolverlo                                                                                 | Prioridad y motivo                                                      | Responsable             | Criterio de cierre y caso asociado                                                                  | Estado                   |
| P-01 | RC-24                                           | Implementar autenticación en el Webhook. Sin ella, personas no autorizadas podrían consultar el inventario.               | Alta: acceso no autorizado a información.                               | Desarrollo              | Rechazar solicitudes sin credenciales y permitir solo las autorizadas. Realizar prueba de acceso.   | Pendiente                |
| P-02 | RC-02, RC-03                                    | Validar nombres vacíos y búsquedas ambiguas para evitar procesar productos equivocados.                                   | Alta: consultas y alertas indebidas.                                    | Desarrollo              | Rechazar campos vacíos y exigir coincidencia inequívoca. CP-02 y prueba de múltiples coincidencias. | Pendiente                |
| P-03 | RC-15                                           | Impedir el envío de correos duplicados por solicitudes repetidas del mismo producto.                                      | Alta: notificaciones duplicadas.                                        | Desarrollo              | Mantener una sola alerta por producto y día y evitar reenvíos innecesarios. CP-08.                  | Pendiente                |
| P-04 | RC-22, RC-23                                    | Implementar recuperación de correos pendientes cuando Gmail falla o se interrumpe el proceso.                             | Alta: pérdida de notificaciones.                                        | Desarrollo              | Conservar la alerta, registrar el fallo y permitir reintentos sin duplicados. CP-10.                | Pendiente                |
| P-05 | RC-07, RC-08                                    | Verificar la validación de datos y agregar tratamiento de errores al subworkflow.                                         | Alta: procesamiento de información inválida.                            | Desarrollo              | Detener el procesamiento cuando falten datos o falle el subworkflow. CP-04 y prueba de fallo.       | Pendiente                |
| P-06 | RC-11                                           | Definir el comportamiento cuando el stock y el mínimo son cero.                                                           | Media: alerta innecesaria con cero faltantes.                           | Producto / Desarrollo   | Documentar la regla de negocio y comprobar el resultado con stock 0 y mínimo 0.                     | Pendiente                |
| P-07 | RC-13, RC-14                                    | Comprobar la escritura en Supabase y los errores después de agotar los reintentos.                                        | Alta: pérdida de alertas o confirmaciones falsas.                       | Desarrollo / Pruebas    | Verificar registro correcto y tratamiento del fallo tras tres intentos. CP-07.                      | Pendiente                |
| P-08 | RC-18, RC-19, RC-20                             | Comprobar el contenido de Gemini, la plantilla alternativa y la longitud del asunto.                                      | Media: comunicaciones incorrectas.                                      | Desarrollo / Pruebas    | Validar datos reales, asunto de máximo 80 caracteres y alternativa sin IA. CP-09.                   | Pendiente                |
| P-09 | RC-16                                           | Evitar alertas basadas en inventario que cambió después de la consulta inicial.                                           | Alta: decisiones con datos desactualizados.                             | Desarrollo              | Comprobar nuevamente el stock antes del registro o gestionar cambios concurrentes.                  | Pendiente                |
| P-10 | RC-01, RC-06, RC-10, RC-12, RC-17, RC-21, RC-25 | Ejecutar los casos pendientes y documentar evidencias del recorrido completo.                                             | Media: no existe evidencia suficiente para sustentar el funcionamiento. | Pruebas / Documentación | Registrar resultados, capturas y conclusiones de CP-01, CP-05, CP-06, CP-07, CP-09 y CP-10.         | Pendiente                |
**Cambios realizados durante esta clase, si los hubo:**

Durante la revisión se documentaron las etapas del workflow de inventario y alertas de reposición, se identificaron controles existentes y posibles brechas, y se definieron diez casos de prueba con condiciones iniciales, entradas, resultados esperados y criterios de aceptación.

Hasta el momento no se han confirmado modificaciones en la configuración ni en los nodos del workflow. Las actividades realizadas corresponden al análisis y documentación del proceso. Las pruebas funcionales y sus evidencias permanecen pendientes de ejecución.

**Versión del workflow:** Pendiente de verificar antes de la sustentación.

## 6. Revisión entre integrantes del grupo

Revisen juntos la bitácora y comparen el recorrido descrito con el diagrama y los workflows. Cada integrante debe poder señalar omisiones, discutir criterios y comprender los pendientes asignados. No se requiere intercambiar el proyecto con otro grupo.

| Integrante          | Etapas o situaciones revisadas                                                                       | Observación o duda identificada                                                                                      | Acuerdo del grupo y pendiente asociado, si corresponde                                                                                         |
| ------------------- | ---------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Valentina Gutiérrez | Recepción de solicitudes, consulta de productos y validación de datos (RC-01 a RC-08).               | Se identificaron solicitudes vacías, búsquedas ambiguas y posibles errores en la preparación de datos.               | Proponer validaciones iniciales y comprobar las rutas de error mediante CP-01 a CP-04. Pendientes P-02 y P-05.                                 |
| Michael Aguilera    | Cálculo de inventario, clasificación de prioridad y decisiones de reposición (RC-09 a RC-12, RC-16). | Se identificó que el stock igual a cero requiere reglas claras y que los datos podrían cambiar durante la ejecución. | Verificar cálculos y prioridades, definir el caso de stock y mínimo iguales a cero y revisar la vigencia de los datos. Pendientes P-06 y P-09. |
| Jorge Martínez      | Registro de alertas en Supabase, reintentos y control de duplicados (RC-13 a RC-15, RC-23).          | Se detectó riesgo de correos duplicados y falta de recuperación cuando una ejecución se interrumpe.                  | Comprobar el registro de alertas, los reintentos y la prevención de duplicados. Pendientes P-03, P-04 y P-07.                                  |
| Gabriela Ríos       | Generación de correos con Gemini, envío mediante Gmail y respuesta final (RC-17 a RC-22, RC-25).     | Se identificaron posibles errores en el contenido generado por IA, límites del asunto y fallos de envío.             | Verificar la plantilla alternativa, la integridad de los mensajes y el manejo de errores de Gmail. Pendientes P-04, P-08 y P-10.               |
### Comprobación final del equipo

- [x] Incluimos todos los workflows relacionados y las ramas del proceso.
- [x] Revisamos todas las etapas y registramos las situaciones pertinentes.
- [x] Justificamos los casos que no aplican y reconocimos lo que falta revisar.
- [x] Localizamos las protecciones existentes y distinguimos las brechas.
- [x] Consideramos la respuesta al usuario y el efecto real en los destinos.
- [x] Los casos previstos contienen condiciones, entradas y criterios concretos.
- [x] Los pendientes tienen responsable y una forma de comprobar su cierre.
- [x] Registramos las observaciones y acuerdos entre integrantes.
- [x] Los resultados de las pruebas siguen pendientes de ejecución, sin resultados inventados.
- [x] Las capturas y los ejemplos no exponen contraseñas, tokens ni datos personales reales.

## 7. Conclusión de la clase

**¿Qué permite afirmar la revisión del diseño actual?**

El diseño contiene protecciones localizadas para las situaciones principales: producto inexistente, datos de stock no válidos, reglas de prioridad en los límites, alerta única por producto y día en la base de datos, y una alternativa cuando fallan la IA o el correo. La IA queda acotada a redactar texto: no decide stock, prioridad ni destinatario, no tiene herramientas ni memoria, y los datos del correo provienen de Supabase. 

**¿Cuáles son las brechas más importantes y por qué?**

Consulta vacía: procesa todo el inventario y puede generar muchas alertas, correos y llamadas a IA.

Correos duplicados por consultas repetidas : la deduplicación existe en la tabla.

Fallos sin rastro : no está comprobada la asignación del manejador de errores y el fallo del correo no se guarda.

Webhook sin autenticación : cualquier cliente que conozca la URL puede disparar escrituras y correos.

Texto de la IA sin verificación de cifras : el contrato valida el formato, no la fidelidad del contenido.

**¿Qué debe quedar listo y qué deberán demostrar en la sustentación?**

Deben quedar resueltos los casos de prueba.


