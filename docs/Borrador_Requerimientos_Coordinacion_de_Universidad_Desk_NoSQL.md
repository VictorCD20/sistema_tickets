# Coordinación de Universidad Desk NoSQL

Borrador de requisitos, roles y arquitectura del sistema

Este documento define una primera versión del funcionamiento de Coordinación de Universidad Desk: un sistema para registrar, atender, dar seguimiento y analizar incidencias del campus universitario. El borrador integra la arquitectura híbrida acordada, los roles de usuario, la distribución de responsabilidades entre PostgreSQL y MongoDB, y los requisitos funcionales y no funcionales del proyecto.

El documento se encuentra en etapa de definición. Las decisiones aquí descritas deben validarse con el equipo y con el docente antes de iniciar la implementación final.

## 1. Alcance del sistema

Coordinación de Universidad Desk permitirá que usuarios del campus registren incidencias relacionadas con tres áreas operativas: Cómputo y Telecomunicaciones, Infraestructura Física y Mantenimiento, y Sistemas y Servicios Académicos. El sistema almacenará cada incidencia como un expediente documental en MongoDB, conservará su trazabilidad y ofrecerá consultas operativas y analíticas.

- Registro abierto de usuarios con rol inicial de reportante.
- Autenticación mediante Supabase Auth.
- Gestión de perfiles y roles en PostgreSQL mediante Supabase.
- Gestión de incidencias y seguimiento en MongoDB Atlas.
- Almacenamiento de imágenes y documentos en Supabase Storage.
- API y aplicación web desarrolladas con Next.js, React y TypeScript.
- Despliegue en Vercel y trabajo colaborativo mediante Git.
- Bot de consulta con Groq API, limitado inicialmente a operaciones de lectura.
## 2. Arquitectura tecnológica

La solución utilizará persistencia híbrida. Supabase será responsable de identidad, autenticación, roles y objetos multimedia. MongoDB Atlas será el núcleo operativo de las incidencias. El backend funcionará como única capa autorizada para comunicarse con las bases de datos y con el almacenamiento.

| Tecnología | Responsabilidad | Alcance |
| --- | --- | --- |
| Next.js | Aplicación web y API | Rutas, vistas, endpoints y despliegue integrado |
| React | Interfaz de usuario | Formularios, paneles, tablas y vistas por rol |
| TypeScript | Lenguaje del proyecto | Frontend, backend, validaciones y tipos compartidos |
| Supabase Auth | Identidad y sesiones | Registro, login, JWT, recuperación y verificación |
| PostgreSQL en Supabase | Roles y perfiles | Perfiles, roles, relación usuario-rol y auditoría |
| MongoDB Atlas | Operación de incidencias | Tickets, historial, etiquetas, asignaciones y analítica |
| Supabase Storage | Archivos | Imágenes, PDF y documentos de diagnóstico |
| Redis o Upstash Redis | Consecutivos | Contador diario para códigos de incidencia |
| Git | Colaboración | Ramas, historial, revisiones e integración del equipo |
| Vercel | Despliegue | Publicación de la aplicación Next.js |
| Groq API | Asistente de consulta | Respuestas sobre datos autorizados y seguimiento |

## 3. Distribución entre SQL y NoSQL

### 3.1 PostgreSQL y Supabase

PostgreSQL se utilizará para información estructurada de identidad y autorización. No almacenará el expediente operativo de la incidencia.

| Elemento | Qué almacena | Regla |
| --- | --- | --- |
| auth.users | Identidad, correo, contraseña gestionada y estado de autenticación | Administrado por Supabase Auth |
| perfiles | Nombre, apellidos, correo institucional y estado | Relacionado con auth.users |
| roles | Catálogo de roles del sistema | No se modifica desde el registro público |
| perfiles_roles | Relación entre usuarios y roles | El coordinador autorizado modifica roles |
| auditoria_roles | Cambios de rol y responsable del cambio | Toda modificación debe quedar registrada |

### 3.2 MongoDB Atlas

MongoDB almacenará los datos que cambian durante el ciclo de vida de una incidencia. La información principal deberá permanecer en un documento autocontenido para cumplir el objetivo documental del proyecto.

```text
incidencias_campus | ubicaciones | tecnicos
```

- incidencias_campus: expediente completo de la incidencia.
- ubicaciones: catálogo de lugares; la incidencia conserva un snapshot de la ubicación.
- tecnicos: disponibilidad, categorías atendidas y referencia al perfil de Supabase.
- El campo detalle_especifico será polimórfico según la categoría.
- historial_seguimiento, tags y evidencias_adjuntas serán arreglos dentro del ticket.
### 3.3 Archivos y referencias

Los archivos físicos se guardarán en Supabase Storage. MongoDB conservará únicamente sus metadatos y la ruta del objeto. El backend generará URLs firmadas temporales para el acceso a archivos privados.

## 4. Roles y responsabilidades

El registro será abierto, pero todo usuario nuevo recibirá automáticamente el rol reportante. Los usuarios podrán editar sus datos personales, pero no podrán autoproclamarse técnicos, coordinadores o directivos. El coordinador autorizado será responsable de asignar o retirar roles. No se incorporará un administrador como rol operativo en esta versión.

| Rol | Qué hará | Restricciones |
| --- | --- | --- |
| Reportante | Registrarse, iniciar sesión, crear incidencias, adjuntar evidencias y consultar sus propios reportes. | No asigna técnicos, no cambia roles y no cierra incidencias. |
| Técnico operativo | Atender casos asignados, registrar diagnóstico, horas, comentarios, etiquetas y evidencias; marcar como resuelta. | No administra usuarios ni cierra definitivamente. |
| Coordinador de soporte | Revisar casos, asignar y reasignar técnicos, ajustar prioridades, escalar, validar soluciones, cerrar y reabrir. | Sus acciones deben quedar auditadas. |
| Coordinador de área | Supervisar una categoría específica, administrar etiquetas y asignaciones de su área. | No debe administrar categorías ajenas sin permiso. |
| Analista de soporte | Buscar incidencias, consultar historial, filtros, etiquetas y reportes operativos. | Acceso principalmente de lectura. |
| Coordinación de Universidad | Consultar todas las áreas, tiempos promedio, fallas recurrentes y métricas por edificio. | No modifica la operación diaria salvo autorización. |
| Directivo | Consultar indicadores generales, incidencias críticas y tendencias ejecutivas. | Acceso limitado a datos personales y archivos sensibles. |
| Bot asistente | Responder consultas autorizadas sobre estados, incidencias y métricas. | Sólo lectura; no crea, actualiza ni elimina datos. |

## 5. Autenticación, registro y autorización

### 5.1 Registro abierto

1. El usuario proporciona nombre, correo institucional y contraseña.
1. Supabase Auth crea la cuenta y gestiona la contraseña.
1. Se verifica el correo institucional.
1. El backend crea el perfil asociado.
1. El sistema asigna automáticamente el rol reportante.
1. El usuario puede iniciar sesión y registrar incidencias.
### 5.2 Cambio de roles

El coordinador autorizado podrá asignar o retirar roles. Aunque no habrá un flujo formal de solicitud, cada modificación deberá guardar el usuario que la realizó, el rol anterior, el rol nuevo, la fecha y el motivo. El usuario nunca podrá modificar sus propios permisos.

### 5.3 Validación de acceso

```text
Frontend -> JWT de Supabase -> Backend TypeScript -> Validación de rol -> MongoDB o Storage
```

El frontend no se conectará directamente a MongoDB. Toda operación protegida pasará por la API de Next.js, que validará el JWT y los permisos antes de consultar o modificar información.

## 6. Flujo de una incidencia

1. El reportante inicia sesión y crea una incidencia.
1. El sistema valida categoría, prioridad, ubicación y detalle específico.
1. Redis genera el consecutivo diario.
1. MongoDB crea el documento con estado abierta e historial inicial.
1. El coordinador revisa el reporte, ajusta la clasificación y asigna un técnico.
1. El técnico cambia el estado a en proceso y registra avances.
1. Si necesita información, materiales o un proveedor, cambia el estado a en espera.
1. El técnico registra la solución y cambia el estado a resuelta.
1. El coordinador valida la solución y cambia a cerrada o reabre el caso.
1. MongoDB utiliza la información cerrada para analítica y aplica TTL después del periodo definido.
```text
abierta -> en_proceso -> en_espera -> en_proceso -> resuelta -> cerrada
```

## 7. Categorías de incidencia

| Área | Ejemplos de información específica | Roles principales |
| --- | --- | --- |
| Cómputo y Telecomunicaciones | Dirección IP, VLAN, puerto uplink, equipos afectados, tipo de falla. | Reportante, técnico de redes, coordinador de soporte. |
| Infraestructura Física y Mantenimiento | Tipo de daño, riesgo de seguridad, insumos, contratista requerido. | Reportante, técnico de mantenimiento, coordinador de área. |
| Sistemas y Servicios Académicos | Plataforma, código de curso, periodo escolar, estudiantes afectados. | Reportante, técnico de sistemas, Coordinación de Universidad. |

## 8. Requisitos funcionales

| ID | Requisito | Descripción |
| --- | --- | --- |
| RF-01 | Registro abierto | El sistema permitirá crear cuentas y asignará automáticamente el rol reportante. |
| RF-02 | Inicio de sesión | El sistema permitirá autenticarse mediante Supabase Auth y devolver una sesión válida. |
| RF-03 | Perfiles | El usuario podrá consultar y actualizar sus datos personales permitidos. |
| RF-04 | Roles | El coordinador autorizado podrá asignar y retirar roles; toda modificación quedará auditada. |
| RF-05 | Crear incidencia | El reportante podrá crear una incidencia con título, descripción, categoría, prioridad y ubicación. |
| RF-06 | Validación polimórfica | El backend y MongoDB validarán el detalle específico correspondiente a la categoría. |
| RF-07 | Código de incidencia | El sistema generará un código único mediante un contador diario. |
| RF-08 | Asignación | El coordinador podrá asignar y reasignar técnicos y registrar el cambio en el historial. |
| RF-09 | Estados | El sistema manejará abierta, en proceso, en espera, resuelta y cerrada. |
| RF-10 | Seguimiento | Los técnicos y coordinadores podrán agregar comentarios, horas y cambios de estado. |
| RF-11 | Etiquetas | El sistema agregará etiquetas sin duplicarlas y permitirá retirarlas. |
| RF-12 | Evidencias | Los usuarios autorizados podrán subir imágenes y documentos a Storage; MongoDB guardará metadatos. |
| RF-13 | Búsqueda textual | El sistema buscará por título y descripción mediante un índice de texto de MongoDB. |
| RF-14 | Filtros | El sistema filtrará por categoría, estado, prioridad, edificio, técnico y fecha. |
| RF-15 | Analítica | El sistema calculará tiempos promedio, horas totales, incidencias por edificio y ranking de etiquetas. |
| RF-16 | Ciclo de vida | MongoDB eliminará automáticamente incidencias resueltas después del periodo TTL definido. |
| RF-17 | API | La aplicación expondrá endpoints protegidos para usuarios, incidencias, evidencias y analítica. |
| RF-18 | Pruebas Postman | Se comprobarán respuestas 201, 200, 400, 401 y 403 mediante una colección de Postman. |
| RF-19 | Bot de consulta | El bot responderá preguntas autorizadas sin ejecutar operaciones de creación, actualización o eliminación. |

## 9. Requisitos no funcionales

| ID | Atributo | Criterio |
| --- | --- | --- |
| RNF-01 | Seguridad | Las contraseñas serán gestionadas por Supabase Auth; las claves permanecerán en variables de entorno. |
| RNF-02 | Autorización | El backend validará JWT, usuario activo y permisos antes de cada operación protegida. |
| RNF-03 | Privacidad | El acceso a incidencias, perfiles y archivos dependerá del rol y del área asignada. |
| RNF-04 | Integridad | MongoDB usará $jsonSchema, índices, operaciones atómicas y tipos nativos. |
| RNF-05 | Trazabilidad | Cada cambio de estado, asignación, etiqueta y comentario conservará usuario y fecha. |
| RNF-06 | Rendimiento | Las consultas críticas utilizarán índices y los listados grandes tendrán paginación. |
| RNF-07 | Disponibilidad | La solución se desplegará con MongoDB Atlas, Supabase, Storage y Vercel. |
| RNF-08 | Escalabilidad | El modelo permitirá nuevas categorías, áreas, edificios y roles sin rediseñar el ticket. |
| RNF-09 | Usabilidad | La interfaz será clara, responsive y mostrará sólo los campos pertinentes a cada categoría. |
| RNF-10 | Mantenibilidad | El código utilizará TypeScript, módulos separados, validaciones y documentación de API. |
| RNF-11 | Archivos | Los objetos se almacenarán fuera de MongoDB y se accederán mediante URLs temporales. |
| RNF-12 | Bot | Groq recibirá únicamente datos necesarios y autorizados; no tendrá acceso directo a las bases. |
| RNF-13 | Colaboración | Git gestionará ramas, revisiones, integración y trazabilidad del desarrollo. |
| RNF-14 | Compatibilidad | La aplicación funcionará en navegadores modernos de escritorio y dispositivos móviles. |

## 10. Operaciones de MongoDB requeridas

| Operación | Uso en el sistema |
| --- | --- |
| $set | Actualizar estado, prioridad, fechas y datos de asignación. |
| $inc | Acumular horas invertidas. |
| $push | Agregar eventos al historial y metadatos de evidencias. |
| $addToSet | Agregar etiquetas sin duplicarlas. |
| $pull | Retirar etiquetas. |
| $text | Buscar coincidencias en título y descripción. |
| $unwind y $group | Construir ranking de etiquetas. |
| $avg y $sum | Calcular promedios y totales por edificio. |
| TTL | Eliminar documentos resueltos después del periodo establecido. |

## 11. Endpoints iniciales

| Método | Endpoint | Acceso |
| --- | --- | --- |
| POST | /api/incidencias | Reportante y coordinador |
| GET | /api/incidencias | Según rol y área |
| GET | /api/incidencias/:id | Según rol y pertenencia |
| PATCH | /api/incidencias/:id/estado | Técnico y coordinador |
| PATCH | /api/incidencias/:id/asignacion | Coordinador |
| POST | /api/incidencias/:id/evidencias | Usuario autorizado |
| GET | /api/analitica | Analista, Coordinación y Directivo |
| PATCH | /api/usuarios/:id/roles | Coordinador autorizado |
| GET | /api/bot/consulta | Usuario autenticado |

## 12. Decisiones actuales y pendientes

| Tema | Decisión actual |
| --- | --- |
| Registro | Abierto; rol inicial automático reportante. |
| Roles | Los usuarios no se autoconceden permisos; el coordinador autorizado modifica roles. |
| Administrador | No se incorpora como rol operativo en esta versión. |
| Resolución | El técnico marca resuelta; el coordinador valida y cierra o reabre. |
| Bot | Sólo consulta; no realiza POST, PATCH, PUT ni DELETE. |
| Archivos | Supabase Storage; MongoDB conserva metadatos. |
| Consecutivos | Redis o Upstash Redis si se cumple literalmente el caso CU-01. |
| Pendiente | Definir coordinador inicial y alcance exacto de sus cambios de rol. |
| Pendiente | Confirmar el periodo TTL con el docente antes de eliminar datos reales. |

## 13. Referencia base

Proyecto Unidad II: Coordinación de Universidad Desk NoSQL Cloud. Sistema Integral de Gestión, Resolución y Analítica de Incidencias en el Campus. Documento académico proporcionado para la definición del proyecto.

