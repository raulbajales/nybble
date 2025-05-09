2 - Adicionalmente al punto 1 el cliente requiere demostrar como aseguraría la alta disponibilidad de este sistema en 24x7

Lo mas importante en este servicio para poder asegurar alta disponibilidad es la disponibilidad con el CRM (sistema externo sobre el cual no tenemos control)
Si consideramos un approach de microservicios para nuestro BE/API/Services hay que tener en cuenta ciertas practicas:

- Queue que nos permita separar el requester del task processor, en este caso lo tenemos resuelto por el uso del AWS SQS (o Kafka)
- Clientes para comunicacion entre microservicios: Deben tener implementado un retry policy tipo exonential backoff
- Circuit breaker que permita marcar un servicio como caido de manera que los clientes de dicho servicio no sean afectados (en nuestro caso si el microservicio CRMIntegration es notificado de que el CRM externo esta caido entonces no es necario invocar el API o Webhook del CRM ya que nos daria Service Unavailable, en este caso podemos usar una DLQ o queue de errores para procesarlos en otro momento)
- Instrumentacino y monitoreo aropiado de todo el sistema incluyendo dashboards con metricas y alertas (usando por ejemplo NewRelic o DataDog)

3 - Un sistema de base de datos se encuentra reportando problema de performance y deadlocks en ciertas tablas y ocasiones de manera random en procesos de larga duración. Por favor explica cómo lo solucionaría y que herramientas utilizaría.

(disclaimer: no soy DBA)

- Intentaria primero con herramientas de monitoreo asociados a nuestra DB (DataDog / New Relic, insights de AWS RDS)
- Intentaria encontrar el query culpable (que mas tiempo este tomando o mas recursos este consumiendo) para revisar el explain y ver si es un problema de indices y/o del modelo en general (a veces un poco de desnormalizacion en el model permite resolver problemas de carga evitando algunos joins)

4 - Explique brevemente cómo convertiría un programa complejo a arquitectura microservicio/s, que componentes requiere en la nube y como lo implementaría.

1. Identificaria que parte del monolito quiero extraer a un nuevo microservicio (incluyendo data model, REST APIs que estmos exponiendo, controllers y services)
2. Me aseguraria de tener un buen conjunto de tests para validar dicha parte del dominio, correria el test contra el monolito y me aseguraria de que todos los tests pasen
3. Comenzaria a reescribir esta parte del monolito en un nuevo domain service
4. Correria la bateria de tests del paso 2 contra el nuevo domain service del paso 3 e iteraria el trabajo en el nuevo domain service hasta asegurarme de que los tests pasan OK
5. Con este nivel de confianza y asumiendo que el nuevo domain service esta corriendo en un entorno de stage, usaria un Feature Flag en el monnolito corriendo en stage para que alguna parte de la funcionalidad que fue migrada se active para un usuario de test
6. Haria pruebas manuales con este usuario de test en stage (y me contactaria con product managers para que revisen si el feature migrado funciona correctamente)
7. Orgaizaria un bug bash session para que parte del team ejercite el featurte migrado para asegurarnos de que corre correctamente
8. Con este nivel de confianza depoloyaria en produccion y activaria una organizacio o usuario cliente al cual monitorearemos durante algunos dias
9. Con este nivel de confianza activaria el feature flag para mas usuarios, progresivamente, durante un periodo de una semana o un mes, segun la carga y funcionalidad, hasta que el FF este rolleado al 100%
10. En este punto el nuevo microservicio esta migrado 100% y necesita ser monitoreado

Eventualmente, en el paso 8 y 9 podria tambien internamente invocar (para cada request de un usuario que implique la ejecucion de la funcionalidad migrada) ambas implementaciones: la vieja y la nueva, y compararia y loguearia resultados y diferencias, esta comparacion se podria hacer tambien bajo otro FF para mas control, esto permitiria encontrar casos en los que un cliente dice ver una diferencia del servicio para la feature migrada.

5 - Nombre las mejores prácticas de OOP y programación que aplica y sus patrones de diseño y cuando los aplicaría y porque. Explique porque utilizaría MVC o que patrones de calidad de código.

En general seguir los principios SOLID permiten modelar codigo con alta cohesin y bajo acoplamiento, facilmente testeable

- Single responsability: Para asegurar alta cohesion, de manera que un componente se encargue de realizar una sola tarea o un grupo de tareas relacionadas
- Open for extension / closed for modification: Refiere a componentes cuyas implemnentaciones se pueden extender sin modificar su comportamiento interno
- Liskov substitution: Relacionado a lo anterior, una clase que extiende de otra deberia poder reemplazarla sin alterar el comportmiento de la app
- Interface segregation / Dependency injection: los componentes mas arriba en la jerajquia deberian tener dependencias con abstracciones (interfaces)

Patrones que uso habitualmente:

- Patrones relacionados a DDD: Repository (abstrae el acceso a datos), Entity (maneja el ciclo de vida de una entidad del negocio), Service (interacciona co entidades y servicios internos y externos, resuelven necesidades del negocio)
- Factory y Builder: Para crear componentes complejos basados en interfaces comunes.
- DI: Resueltas en componentes como el applicatino context de Spring o Gogle Guice, para resolver dependencias de componentes con abstracciones.
- Strategy: Para resolver problemas comunes cuya solucion concretadepende de un cierto contexto.

Por que utilizaria MVC:

- Separacion de concerns: Model modela entidades y negocio, View solo se encarga de layer de presentation, Controller interacciona entre ambas
- Esta separacion de concerns permite modificar un concern independientemente del otro, siempre que no se rompan protocolos del controller

6 - Explique que tecnologías/frameworks/complementes de backend utilizaría para interactuar con base de datos y como resolvería la edición concurrente de registros sin pérdida y sobre escritura de información por múltiples usuarios en una aplicación financiera de más de 300 usuarios concurrentes en una misma tabla de clientes.

1- SpringBoot para la aplication que recibe requests de end users exponiendo una REST API
2- Kafka para manejar una cola de eventos en un topico
3- Modelaria la operacioon contra la tabla, y eso seria el mensaje que enviaria al topic Kafka luego de recibir el request del usuario en la API REST
4- Otra aplicacion SringBoot configurada como consumer Kafka que lea las operaciones del topico del paso 2, al consumer cada operacion ejecutarla, asumiendo que la operacion implique un impacto en la tabla de clientes

7 - Se reportan incidentes de producción en ciertos proceso de larga duración sin informar al usuario afectando la operación crítica del cliente y no se posee servidor de pruebas. Explique los pasos que realizaría para la detección rápida y temprana, corrección futura para mejorar y arreglo de los mismos.

1- Revisar runbooks, logs, metrics, dashboards (DataDog, NewRelic) que puedan evidenciar el problema y dar mas informacion acerca de que esta ocurriendo, y que microservicio pueda estar siendo afectado
2- Reunir a los engineers reacionados segun lo que se encuentre en el paso 1 para evaluar posible causa
3- Una vez que tuviera una o mas posibles causas, intentar reproducir el problema ya sea localmente o en produccion con alguna otra cuenta de prueba nuesra
4- Si podemos reproducir el problema consistentemente entonces pensar en un hotfix (que seria bueno deployar con un feature flag) o como desactivar el icroservicio que este fallando (con un circuit breaker)
5- Una vez que validemos que el hotfix corrige el problema (revisano nuevamente paso 1) entonces asegurarnos de activar el FF de correccion de fix para mas clientes y monitorear, hacer esto hasta que el rollout del fix este al 100%
6- Una vez resuelto el problema revisar consecuencias que pudieramos tener de tener el bug en produccion (perdida de datos, procesos que no fueron ejecutados a tiempo etc.), preparar scripts que resuelvan estor problemas si fuera posible, correrlos, y eventualmente avisar al customer como pudo haber sido afectado
7- Preparar un documento postmorten para entender al detalle que paso, como se fixeo, y levantar jira issues para empezar a trabajar en una solucion mas robusta incluyendo tests unitarios y de integracion que validen que el bug fue resuelto (asumiendo que los tests se corran en el pipeline de CI/CD cada vez que se mergea un PR de manera que podamos saber si el bug aparece de nuevo)

8 - La empresa XYX con 40 sucursales con 4000 artículos disponibles y desea optimizar el flujo de envíos semanales d stock a cada sucursal. Se provee un archivo con las ventas de cada sucursal de los últimos 3 periodos. Indiqué cómo optimizaría el flujo de stock y porque.

1. Escribo un algoritmo que pueda identificar los articulos con mayor demanda de cada sucursal (en forma de historial, demanda enm el tiempo de articulo / sucursal), basado en el historial de los ultimos 3 periodos
2. Buscaria una manera de calcular demanda futura de articulos por sucursal basado en el historial y los resultados del paso 1
3. Con el historial + demanda futura de articulo / sucursal buscaria optimizar el envio del stock, agrupando articulos / sucursal / demanda esperada por periodo, y organizaria los envios
