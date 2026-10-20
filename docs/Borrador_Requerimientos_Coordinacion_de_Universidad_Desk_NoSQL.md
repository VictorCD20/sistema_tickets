# Coordinación de Universidad Desk NoSQL

Borrador de requisitos, roles y planificación técnica alineado con la implementación actual del repositorio utm-desk-nosql-cloud. El documento separa las funciones ya implementadas de las propuestas para fases posteriores.

## 1. Alcance y decisión arquitectónica

La versión actual es una aplicación web para registrar, consultar, atender, resolver y analizar incidencias universitarias. MongoDB Atlas es la fuente principal del sistema: almacena usuarios, incidencias y contadores. Supabase se utiliza únicamente como almacenamiento privado de evidencias. La autenticación actual es propia, con contraseñas cifradas mediante bcrypt y sesiones JWT.

| Capa | Tecnología actual | Responsabilidad |
| --- | --- | --- |
| Frontend | React, Vite y Tailwind CSS | Login, formularios por categoría, bandeja, detalle, analítica y carga de evidencias. |
| Backend | Node.js 20, Express y JavaScript ESM | API, validación, autorización, transiciones y reglas de negocio. |
| Datos operativos | MongoDB Atlas, driver nativo | Colecciones usuarios, incidencias_campus y contadores; índices, TTL y agregaciones. |
| Archivos | Supabase Storage | Bucket privado evidencias; URLs firmadas temporales para imágenes y PDF. |
| Despliegue | Vercel y variables de entorno | Publicación del frontend/API y gestión de configuración. |
| Colaboración | Git, GitHub y Postman/Newman | Trabajo por ramas, revisión, documentación y pruebas de API. |

No forman parte de la implementación actual: Next.js, TypeScript, Supabase Auth/PostgreSQL, Redis/Upstash, registro público, bot de Groq y un rol separado de Soporte TI. Pueden evaluarse como ampliaciones, pero no deben presentarse como funciones terminadas.

## 2. Roles y responsabilidades

| Rol | Qué hará | Restricciones |
| --- | --- | --- |
| Reportante | Iniciar sesión, crear incidencias, consultar sus propios reportes y adjuntar evidencias autorizadas. | No cambia estados finales, no administra usuarios y no modifica roles. |
| Técnico | Consultar incidencias, atender casos, actualizar estado, registrar comentarios, diagnóstico, horas, etiquetas y evidencias; marcar una incidencia como resuelta. | No cierra definitivamente, no elimina incidencias y no administra roles. |
| Coordinador | Supervisar operación, editar datos generales, validar y cerrar incidencias, eliminar registros cuando corresponda, revisar etiquetas y consultar analítica. | Sus acciones deben quedar autenticadas; no debe existir autoconcesión de privilegios. |
| Directivo | Consultar incidencias y paneles de analítica para seguimiento institucional. | Rol de consulta; no crea, actualiza ni elimina incidencias. |

Soporte TI puede existir como área o especialización del técnico, pero no como quinto rol en la versión actual. El bot asistente tampoco es un usuario con permisos: se propone para una fase futura y comenzaría sólo con consultas de lectura.

## 3. Autenticación, usuarios y permisos

- El usuario se autentica en POST /api/auth/login; el backend consulta la colección usuarios.
- La contraseña se verifica con bcrypt y el backend entrega un JWT. GET /api/auth/me valida la sesión activa.
- Las rutas protegidas requieren JWT, usuario activo y rol permitido.
- El repositorio actual no tiene endpoint de registro abierto. Si se decide habilitarlo, deberá crear usuarios con rol inicial reportante y aplicar validación de correo, estado activo y límites contra abuso.
- Los roles no se autoconceden. Un eventual cambio de rol debe ser una operación explícita, auditada y reservada al coordinador autorizado o a una futura cuenta administrativa.
{ "email": "usuario@universidad.mx", "rol": "reportante", "activo": true, "password_hash": "<bcrypt>", "fecha_creacion": "<Date>" }

## 4. Modelo de datos

Inventario formal: la implementación actual utiliza exactamente tres colecciones de MongoDB: usuarios, incidencias_campus y contadores. No existen colecciones independientes de roles, ubicaciones, técnicos, asignaciones o evidencias; esas piezas se guardan como campos simples, snapshots o subdocumentos embebidos.

| Colección | Contenido |
| --- | --- |
| usuarios | Identidad operativa, correo, hash de contraseña, rol y estado activo. |
| incidencias_campus | Expediente completo: código, reportante, ubicación, categoría, detalle específico, estado, historial, tags, evidencias y fechas. |
| contadores | Consecutivo diario generado de forma atómica para los códigos de incidencia. |

Las evidencias físicas no se guardan dentro de MongoDB. Mongo conserva sus metadatos, mientras Supabase Storage mantiene los objetos en el bucket privado evidencias y entrega URLs firmadas con duración limitada.

La relación entre usuarios e incidencias se resuelve mediante referencias ObjectId y snapshots embebidos: reportante.usuario_id identifica al usuario, mientras nombre, tipo y correo conservan los datos de la incidencia al momento de registrarla. El historial conserva su propio snapshot del autor.

{ "codigo": "INC-20261019-0001", "categoria": "computo_redes", "estado": "en proceso", "reportante": { "usuario_id": "...", "nombre": "...", "correo": "..." }, "detalle_especifico": { "tipo_falla": "switch", "direccion_ip": "192.168.1.10", "vlan": 20, "equipos_afectados": 12 }, "historial_seguimiento": [], "evidencias_adjuntas": [] }

## 5. Categorías y datos específicos

Todas las incidencias comparten título, descripción, prioridad, ubicación, reportante, fechas y estado. La categoría determina detalle_especifico y el formulario valida sólo los campos correspondientes.

| Categoría | Campos específicos actuales | Ejemplo |
| --- | --- | --- |
| Computo y redes | tipo_falla: switch, access_point, servidor o equipo_laboratorio; direccion_ip válida; vlan 1–4094; puerto_uplink; equipos_afectados. | Falla de switch en laboratorio, VLAN 20, 12 equipos afectados. |
| Infraestructura | tipo_dano: filtracion, luminaria, climatizacion, cerradura u otro; riesgo_seguridad; requiere_contratista; insumos_requeridos y contratista_externo cuando aplica. | Filtración con riesgo medio que requiere contratista. |
| Académico | plataforma: lms, portal_calificaciones o correo_institucional; periodo_escolar YYYY-1/2/3; estudiantes_impactados; codigo_curso salvo correo. | LMS no disponible en periodo 2026-1 para 35 estudiantes. |

Las reglas de categoría se validan en Zod antes de guardar y también en el validador $jsonSchema de MongoDB. Esto evita que el documento termine con campos mezclados entre categorías.

### 5.1 Ejemplos JSON por categoría

{ "categoria": "computo_redes", "detalle_especifico": { "tipo_falla": "switch", "direccion_ip": "192.168.1.10", "vlan": 20, "puerto_uplink": "SW-A-02-14", "equipos_afectados": 12 } }

{ "categoria": "infraestructura", "detalle_especifico": { "tipo_dano": "filtracion", "riesgo_seguridad": "medio", "requiere_contratista": true, "insumos_requeridos": ["sellador"], "contratista_externo": "Servicios Generales" } }

{ "categoria": "academico", "detalle_especifico": { "plataforma": "lms", "periodo_escolar": "2026-1", "estudiantes_impactados": 35, "codigo_curso": "BDN-2026" } }

Estos ejemplos representan únicamente detalle_especifico. En el documento final deben integrarse con los campos comunes de la incidencia y con valores BSON Date para fecha_creacion, fecha_actualizacion y fecha_resolucion cuando corresponda.

## 6. Flujo operativo de una incidencia

1. El usuario autorizado inicia sesión y recibe un JWT.
1. El reportante captura los datos comunes, selecciona una categoría y completa su detalle específico.
1. La API valida el payload, genera el código mediante contadores y guarda la incidencia en MongoDB con estado abierta.
1. El técnico consulta la bandeja, atiende el caso y registra avances, horas, etiquetas, comentarios y evidencias.
1. El técnico puede pasar el caso a en proceso, en espera o resuelta; no puede cerrarlo definitivamente.
1. El coordinador revisa la solución y cambia resuelta a cerrada. Una incidencia cerrada no se reabre en el flujo actual.
1. Directivo consulta el estado y la analítica sin intervenir en la operación.
Estados permitidos: abierta, en proceso, en espera, resuelta y cerrada. La transición a cerrada está restringida al coordinador.

## 7. Requerimientos funcionales

| ID | Función | Criterio de aceptación | Estado |
| --- | --- | --- | --- |
| RF-01 | Inicio de sesión | Autenticar usuarios activos con bcrypt y JWT. | Implementado |
| RF-02 | Sesión | Consultar el usuario actual mediante /api/auth/me y proteger rutas. | Implementado |
| RF-03 | Crear incidencia | Reportante, técnico y coordinador pueden registrar una incidencia válida. | Implementado |
| RF-04 | Validación por categoría | Aplicar detalle específico y reglas propias de computo_redes, infraestructura y academico. | Implementado |
| RF-05 | Consecutivo | Generar códigos diarios por categoría y anuales con actualización atómica en contadores. | Implementado |
| RF-06 | Consulta | Listar, filtrar, paginar, ver detalle y buscar por texto. | Implementado |
| RF-07 | Estados | Actualizar estados respetando transiciones y permisos. | Implementado |
| RF-08 | Seguimiento | Registrar historial, horas, comentarios, diagnóstico y etiquetas. | Implementado |
| RF-09 | Evidencias | Subir imágenes/PDF de hasta 4 MB a Storage y guardar metadatos en Mongo. | Implementado |
| RF-10 | Analítica | Consultar incidencias por edificio y ranking de etiquetas para roles autorizados. | Implementado |
| RF-11 | Edición y borrado | Permitir al coordinador editar o eliminar una incidencia según las reglas del backend. | Implementado |
| RF-12 | Retención | Aplicar TTL a incidencias resueltas durante 60 días. | Implementado |
| RF-13 | Pruebas | Mantener colección Postman/Newman para respuestas exitosas y errores 400, 401 y 403. | Por verificar en entrega |
| RF-14 | Registro abierto | Crear cuentas nuevas con rol inicial reportante. | Pendiente |
| RF-15 | Bot de consulta | Responder consultas autorizadas sin mutaciones. | Futuro |

## 8. Requerimientos no funcionales

| ID | Atributo | Criterio verificable | Estado |
| --- | --- | --- | --- |
| RNF-01 | Seguridad | JWT, bcrypt, helmet, CORS, rate limiting, secretos en variables de entorno y bucket privado. | Implementado |
| RNF-02 | Integridad | Validación Zod, $jsonSchema, índices únicos y operaciones atómicas. | Implementado |
| RNF-03 | Rendimiento | Índices para código, reportante, texto, categoría/estado/edificio y paginación. | Implementado |
| RNF-04 | Disponibilidad | Despliegue en Vercel, MongoDB Atlas y Supabase Storage. | Por verificar en nube |
| RNF-05 | Mantenibilidad | Arquitectura por dominio, aplicación, infraestructura e interfaz HTTP; JavaScript ESM. | Implementado |
| RNF-06 | Usabilidad | Interfaz responsive y formularios que muestren los campos de la categoría elegida. | Por validar manualmente |
| RNF-07 | Trazabilidad | Historial de seguimiento con usuario, fecha y descripción del cambio. | Implementado |
| RNF-08 | Privacidad | URLs firmadas temporales y acceso a incidencias según rol. | Implementado |
| RNF-09 | Compatibilidad | Navegadores modernos en escritorio y móvil. | Por validar |
| RNF-10 | Colaboración | Git/GitHub con cambios revisables y documentación de API. | Implementado |

### 8.1 Matriz de correspondencia con los PDFs

| Tema del enunciado | Implementación en la repo | Conclusión |
| --- | --- | --- |
| MongoDB Atlas | Colecciones incidencias_campus, usuarios y contadores; driver nativo sin Mongoose. | Cumple el núcleo NoSQL. |
| Redis para consecutivo | Contadores atómicos en MongoDB mediante $inc y upsert. | Cambio de implementación; documentar que se descartó Redis por decisión del proyecto. |
| Cloud Storage | Supabase Storage, bucket privado evidencias y URLs firmadas. | Cumple la estrategia de archivos. |
| Tres categorías | computo_redes, infraestructura y academico con detalle_especifico polimórfico. | Cumple; los nombres son versiones compactas del enunciado. |
| 20 a 30 incidencias y evidencias | Requisito del seed y del entregable académico. | Debe verificarse con db:seed y db:export. |
| Pipelines de analítica | Analítica por edificio y ranking de tags. | Cumple en código; falta evidenciar resultados en el reporte final. |
| Postman | Colección y pruebas de API en el repositorio. | Debe anexarse con códigos, cabeceras y payloads. |

## 9. API y entregables actuales

| Método | Endpoint | Acceso |
| --- | --- | --- |
| POST | /api/auth/login | Público |
| GET | /api/auth/me | Autenticado |
| POST | /api/incidencias | Reportante, técnico, coordinador |
| GET | /api/incidencias y /:codigo | Autenticado |
| GET | /api/incidencias/buscar | Autenticado |
| PATCH | /api/incidencias/:codigo/estado | Técnico y coordinador |
| PATCH/DELETE | /api/incidencias/:codigo | Coordinador |
| POST | /api/incidencias/:codigo/evidencias | Usuario autorizado |
| POST/DELETE | /api/incidencias/:codigo/tags | Usuario autorizado |
| GET | /api/analitica/edificios y /tags | Coordinador y directivo |
| GET | /api/health | Público |

El repositorio ya contiene frontend, backend, documentación, especificaciones, colección de Postman y entregables. La planificación debe partir de estas piezas y evitar rediseñar la arquitectura hacia Next.js, TypeScript o Supabase Auth sin una decisión nueva del equipo.

## 10. Pendientes y fases posteriores

| Tema | Estado |
| --- | --- |
| Registro abierto | Pendiente: no existe endpoint; si se aprueba, inicia siempre como reportante. |
| Bot Groq | Pendiente: comenzar sólo como consulta de lectura, sin acceso directo a bases ni mutaciones. |
| Soporte TI | Pendiente: modelarlo como área/especialidad del técnico antes de convertirlo en rol. |
| Cambio de roles | Pendiente: definir quién será coordinador inicial y agregar auditoría específica si se habilita. |
| Supabase Auth/PostgreSQL | No necesario para la versión actual; evaluar sólo si se requiere proveedor externo de identidad. |
| Redis/Upstash | Descartado en la versión actual; el consecutivo usa contadores atómicos de MongoDB. |

## 11. Referencia

Documento académico base: Proyecto Unidad II UTM Desk NoSQL. Este borrador distingue los requisitos del proyecto de las decisiones de implementación verificadas en el repositorio utm-desk-nosql-cloud.

