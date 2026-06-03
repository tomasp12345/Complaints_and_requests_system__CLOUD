# Sistema PQRS — Transportes RG

Este repositorio presenta mi experiencia desarrollando un sistema de gestión de pqrs para una empresa logística colombiana.

“PQRS” significa Peticiones, Quejas, Reclamos y Sugerencias, el canal formal que muchas empresas en Colombia utilizan para registrar, 
hacer seguimiento y responder solicitudes o inconformidades de clientes.

El código fuente se encuentra en un repositorio privado, ya que es una herramienta interna para una empresa real funcionando en produccion. 
Por eso, este README funciona como una explicación del sistema que construí, las funcionalidades implementadas y las decisiones técnicas que tomé, 
más que como una documentación directa del código. Estoy dispuesto a explicar la arquitectura y el desarrollo del proyecto en una entrevista.

## Qué hace el sistema

* Cada empleado inicia sesión con su propia cuenta.
  un espacio de "olvide mi contraseña" que envia un correo para restaurar contraseña.
* Permite registrar una PQRS con datos del cliente, tipo de solicitud, canal de recepción y archivos adjuntos.
* Permite asignar cada caso a un responsable, área, prioridad y fecha límite de respuesta.
* Maneja estados del proceso, como: Nuevo → En revisión → En progreso → Cerrado / Vencido.
* Incluye historial de auditoría completo: cada cambio registra quién lo hizo, cuándo lo hizo y qué cambió.
* Incluye notificaciones dentro de la aplicación, por ejemplo, para nuevos casos, asignaciones, casos resueltos o cerrados.
* Genera reportes sobre casos del mes, casos abiertos, casos vencidos, tiempo promedio de respuesta y desgloses por tipo, estado y responsable.
* Permite filtrar por año, mes, tipo, estado y responsable.

## Stack tecnológico

* Frontend: JavaScript + Vite. No utilicé framework porque el proyecto era pequeño y quería entender mejor el funcionamiento del DOM.
* Base de datos, autenticación y almacenamiento:** Supabase con PostgreSQL. 
  Elegí Supabase porque quería una arquitectura en la nube sin tener que administrar un servidor propio. 
  Además, el sistema está pensado para menos de 10 usuarios, por lo que el plan gratuito era suficiente para este caso.
* Hosting: Vercel, con despliegue automático desde GitHub para mantener la aplicación disponible fácilmente y sin costo adicional.
* Backups automatizados:** GitHub Actions ejecutando un `pg_dump` programado para generar backups diarios de forma automatica.

## Seguridad
* Vercel se encarga de https para comunicacion con el sitio (comunicación cifrada y certificados), además de protección contra DDOS. 
* Supabase se encarga de https para comunicacion con base de datos, encriptar contraseñas y manejo de tokens. RLS. Proteccion contra
  inyccion sql (consultas parametrizadas).
* Yo me encargue de proteccion contra inyección xss en el código, escribimos reglas del RLS, historial a prueba de manipulación,
  manejo de cuentas de usuarios (invitar, eliminar, banear), respaldos automáticos.

## Decisiones técnicas de las que estoy satisfecho

* Historial de auditoría a nivel de base de datos. 
  El cliente me indicó que el historial de cambios era una de las funcionalidades más importantes del sistema.

* En lugar de registrar los cambios desde el frontend, lo cual podría omitirse o ser manipulado, 
  utilicé triggers en PostgreSQL. De esta forma, cualquier cambio realizado sobre una PQRS queda registrado sin importar desde dónde se haga.

* Row Level Security. Ningún dato puede leerse o modificarse sin una sesión válida (Row level security). 
  Además, las notificaciones están limitadas para que cada usuario solo vea las que le corresponden.

* Columna generada para cálculos de reclamos. 
  Para los reclamos, la “pérdida de la empresa” se calcula restando al valor del reclamo lo que cubre cada parte. 
  Implementé este cálculo como una columna generada en PostgreSQL, de modo que el valor se mantenga correcto 
  automáticamente y no pueda desincronizarse de los datos originales.

## Problemas que encontré y aprendizajes importantes

* Los backups fallaban por una incompatibilidad de versiones.**
  El backup programado fallaba con el error `pg_dump: server version mismatch`. Supabase había migrado a PostgreSQL 17, 
  pero el runner de GitHub Actions seguía usando `pg_dump` 16.

  Instalé el cliente de PostgreSQL 17, pero el sistema seguía usando la versión 16 porque el 
  `PATH` del runner apuntaba primero al binario anterior. La solución fue llamar `pg_dump` usando su ruta completa:
  `/usr/lib/postgresql/17/bin/pg_dump`
  Este problema tomó varios intentos, pero me ayudó a entender mejor cómo funcionan los entornos de ejecución en GitHub Actions.

* **La aplicación se rompía al cambiar de pestaña en el navegador.**
  Al volver a la pestaña, aparecía el error `invalid input syntax for type uuid: "undefined"`.
  El problema ocurría porque Supabase dispara eventos de autenticación cuando refresca el token de sesión. 
  Mi código estaba renderizando nuevamente toda la aplicación, pero en ese proceso se perdía el ID del registro que el usuario estaba viendo.
  Lo solucioné guardando la vista actual y sus parámetros en el estado de la aplicación. 
  Además, ajusté la lógica para que la aplicación solo se renderizara completamente cuando hubiera un inicio o cierre de sesión real, 
  no en cada actualización automática del token.

* **Una palabra demasiado larga dañaba el diseño.**
  Un comentario de prueba con varios cientos de caracteres sin espacios empujó un panel completo fuera de la pantalla.
  Aunque parecía un error bobo, fue difícil de detectar porque el comentario estaba al final de la página y al principio no 
  consideré que el texto pudiera estar causando el problema. Lo solucioné usando `min-width: 0` 
  en los hijos del grid y `overflow-wrap` para permitir que las palabras largas se partieran correctamente.

## Capturas del sistema desplegado

Las siguientes capturas muestran la aplicación ya desplegada. Solo se muestra información temporal utilizada para pruebas.

<img width="2552" height="1350" alt="image" src="https://github.com/user-attachments/assets/66f32831-3dfb-4774-a09b-bbecb78c6fde" />

<img width="2089" height="1107" alt="image" src="https://github.com/user-attachments/assets/b901099d-d449-444d-875f-3be77ed85d14" />

<img width="1341" height="1148" alt="image" src="https://github.com/user-attachments/assets/8ed0eb70-4891-498b-b1cf-2532ab0ef967" />

## Qué haría después

* Alertas por correo para casos próximos a vencer.
* Exportación de reportes a Excel.
* Configurar un dominio personalizado.

## Contexto

Este fue mi primer proyecto usando Supabase y desplegando una aplicación completa de inicio a fin.

Cuando me pidieron desarrollar este sistema, honestamente pensé que no iba a ser capaz de hacerlo. Sin embargo, después de aprender sobre Supabase, Vercel y despliegues en la nube, el proceso terminó siendo mucho más alcanzable de lo que imaginaba.

También ayudó que el sistema estuviera pensado para un equipo pequeño de usuarios y no para una aplicación de cientos de miles de personas. Aun así, el proyecto me permitió aprender mucho sobre autenticación, seguridad, base de datos, triggers, despliegue, backups automatizados y manejo de errores reales en producción.

## Actualización 1: correos y contraseñas

Inicialmente, los usuarios eran creados manualmente por mí. Sin embargo, actualicé el sistema para mejorar la privacidad y la experiencia de los empleados.
Ahora envío un enlace al correo de cada empleado para que pueda crear su propia contraseña sin que yo tenga que conocerla. 
También agregué la opción de “olvidé mi contraseña”, permitiendo que los usuarios puedan recuperar el acceso a su cuenta de forma independiente.

Para lograrlo, configuré un SMTP personalizado, ya que Supabase solo me permitía enviar una cantidad muy limitada de correos desde su configuración por defecto. Con esta integración, el sistema puede enviar correos de invitación y recuperación de contraseña correctamente.

Además, utilicé una rama adicional en GitHub para hacer pruebas. Como la rama `main` estaba conectada directamente a Vercel y desplegaba la versión de producción, usé la rama `master` para probar los cambios en la aplicación desplegada sin afectar el sistema que ya estaba en uso.

