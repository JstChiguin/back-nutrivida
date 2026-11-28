# NutriVida · Backend

Servicios REST de **Clínica NutriVida**, una plataforma de atención nutricional. El backend está formado por tres microservicios Spring Boot que comparten un módulo común de seguridad, validación y manejo de errores, y que persisten sus datos en MongoDB.

El cliente web correspondiente está en **[front-nutrivida](https://github.com/JstChiguin/front-nutrivida)**.

## Contenido

- [Funcionalidades](#funcionalidades)
- [Arquitectura](#arquitectura)
- [Tecnologías](#tecnologías)
- [Requisitos](#requisitos)
- [Configuración segura](#configuración-segura)
- [Ejecución con Docker Compose](#ejecución-con-docker-compose)
- [Compilación y ejecución sin contenedores](#compilación-y-ejecución-sin-contenedores)
- [Pruebas](#pruebas)
- [Swagger y endpoints](#swagger-y-endpoints)
- [Integración con el frontend](#integración-con-el-frontend)
- [Seguridad](#seguridad)
- [Roles y permisos](#roles-y-permisos)
- [Base de datos](#base-de-datos)
- [Despliegue en AWS](#despliegue-en-aws)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Limitaciones conocidas](#limitaciones-conocidas)

## Funcionalidades

- Registro público de pacientes y administración de cuentas con roles.
- Inicio y cierre de sesión con JWT; los tokens cerrados quedan revocados hasta su expiración.
- Catálogo público de servicios nutricionales y de profesionales, con filtros y paginación.
- Perfiles de pacientes y nutricionistas, y autorización explícita de cada profesional sobre sus pacientes.
- Fichas clínicas con evoluciones que no se sobrescriben: una corrección crea un nuevo registro y marca el original.
- Mediciones con cálculo de IMC y serie de evolución lista para graficar.
- Planes alimentarios versionados.
- Agenda semanal, bloqueos y ausencias, disponibilidad por servicio y reglas para evitar dobles reservas, fechas pasadas y bloqueos que afecten citas activas.
- Ciclo de vida de las citas: solicitud, confirmación, rechazo, cancelación con anticipación mínima, reprogramación, atención realizada e inasistencia.
- Pagos simulados con comprobante ficticio, sin procesamiento de dinero real.
- Formulario público de contacto con detección de duplicados del mismo día.
- Reportes agregados y bitácora de auditoría para administradores.

## Arquitectura

```
                     Frontend (React)
                           │  HTTPS / JSON + Bearer JWT
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
servicio-autenticacion  servicio-clinico     servicio-agenda
      :8081                :8082                :8083
      │   ▲                  ▲                    │
      │   └── perfil de ─────┘◄── autorización ───┘
      │       paciente (API interna, rol SERVICIO_INTERNO)
      └───────────────┬─────────────────────────────┘
                      ▼
                   MongoDB
```

| Servicio | Puerto | Responsabilidad |
|---|---|---|
| `servicio-autenticacion` | 8081 | Registro, inicio y cierre de sesión, emisión de JWT, usuarios, roles y revocación de tokens. |
| `servicio-clinico` | 8082 | Servicios, nutricionistas, pacientes, fichas clínicas, mediciones, planes alimentarios, solicitudes de contacto, auditoría y reportes clínicos. |
| `servicio-agenda` | 8083 | Agendas, bloqueos, disponibilidad, citas, pagos simulados y reportes de actividad. |
| `comun` | — | Biblioteca compartida: seguridad JWT, errores, validaciones (RUT), paginación, auditoría, OpenAPI y esquemas de MongoDB. |

Cada microservicio sigue una arquitectura en capas:

- `controlador`: recibe la solicitud HTTP, valida la entrada y devuelve el código HTTP adecuado; no contiene reglas de negocio.
- `servicio`: reglas de negocio y coordinación de repositorios.
- `repositorio`: acceso a datos con Spring Data MongoDB.
- `modelo`: documentos persistidos.
- `dto`: contratos de entrada y salida de la API, para no exponer los documentos completos ni campos sensibles.

Cada servicio gestiona sus propias colecciones dentro de la base `nutrivida`. Cuando necesitan datos de otro dominio se comunican por HTTP con un token de servicio interno, y las rutas `/api/v1/internos/**` se bloquean en el proxy público del frontend.

## Tecnologías

- Java 21 y Spring Boot 3.5 (Spring Web, Validation, Security, Data MongoDB).
- JWT con la biblioteca JJWT (firma HMAC).
- BCrypt para el almacenamiento de contraseñas.
- springdoc-openapi (Swagger UI y OpenAPI 3).
- MongoDB 7.
- Maven multimódulo.
- JUnit 5, Spring Boot Test, MockMvc y Spring Security Test.
- Docker y Docker Compose.

## Requisitos

- Docker y Docker Compose, para la ejecución con contenedores.
- JDK 21 y Maven 3.9 o superior, para compilar y ejecutar sin contenedores.
- Un servidor MongoDB 7 accesible (el Compose incluido lo levanta localmente).
- Node.js 20.6 o superior, solo para el script opcional de datos ficticios.

## Configuración segura

Ningún secreto se incluye en el repositorio. Copie la plantilla y complete los valores localmente:

```bash
cp .env.example .env
```

| Variable | Obligatoria | Descripción |
|---|---|---|
| `NUTRIVIDA_JWT_SECRETO` | Sí | Clave de firma de los JWT. Mínimo 32 caracteres aleatorios; el servicio no inicia con una clave más corta. |
| `NUTRIVIDA_ADMIN_CORREO` | No | Correo de la cuenta administradora inicial. |
| `NUTRIVIDA_ADMIN_CONTRASENA` | No | Contraseña del administrador inicial. Si queda vacía se usa `NUTRIVIDA_CONTRASENA_DEMO`. Solo se aplica al crear la cuenta; no modifica una cuenta existente. |
| `NUTRIVIDA_CONTRASENA_DEMO` | No | Contraseña común de las cuentas ficticias que crea el script de datos de demostración. |
| `NUTRIVIDA_ORIGENES_PERMITIDOS` | No | Orígenes CORS permitidos, separados por comas. Por defecto `http://localhost:5173`. |
| `NUTRIVIDA_JWT_EXPIRACION_MINUTOS` | No | Vigencia del token de acceso. Por defecto 15. |
| `NUTRIVIDA_SESION_MAXIMA_HORAS` | No | Límite absoluto de la sesión de renovación. Por defecto 8. |
| `NUTRIVIDA_COOKIE_SEGURA` | No | Atributo `Secure` de la cookie de refresco. Por defecto `true`; el compose local lo deja en `false` para HTTP. |
| `NUTRIVIDA_COOKIE_SAME_SITE` | No | Atributo `SameSite` de la cookie de refresco. Por defecto `Lax`. |
| `NUTRIVIDA_HORAS_RECORDATORIO` | No | Ventana de recordatorio de citas confirmadas. Por defecto 24. |
| `NUTRIVIDA_MONGO_URI` | No | URI de MongoDB. Por defecto `mongodb://localhost:27017/nutrivida`; el Compose fija la base local del contenedor. |
| `PUERTO_MONGO` | No | Puerto local publicado por el contenedor de MongoDB. Por defecto 27018. |

Para generar una clave aleatoria:

```bash
openssl rand -base64 48
```

Con MongoDB Atlas u otro servidor administrado, guarde la cadena de conexión únicamente en el `.env` privado del servidor y restrinja el acceso de red a las direcciones que lo necesiten. Los archivos `.env` están excluidos por `.gitignore`.

## Ejecución con Docker Compose

```bash
git clone https://github.com/JstChiguin/back-nutrivida.git
cd back-nutrivida
cp .env.example .env
# Edite .env: defina NUTRIVIDA_JWT_SECRETO y, si lo desea, las contraseñas del administrador y de demostración
docker compose up --build -d
docker compose ps
```

El Compose levanta MongoDB (publicado solo en `127.0.0.1:27018`) y los tres servicios en los puertos 8081, 8082 y 8083. Para detener el entorno:

```bash
docker compose down        # conserva los datos
docker compose down -v     # elimina también el volumen de MongoDB
```

Para cargar datos exclusivamente ficticios, con los servicios en ejecución y `NUTRIVIDA_CONTRASENA_DEMO` definida:

```bash
node --env-file=.env scripts/sembrar-datos-demo.mjs
```

## Compilación y ejecución sin contenedores

```bash
mvn clean package
```

Cada servicio genera un JAR ejecutable (`servicio-*/target/nutrivida-servicio-*-1.0.0.jar`). Con MongoDB disponible y las variables de entorno definidas:

```bash
java -jar servicio-autenticacion/target/nutrivida-servicio-autenticacion-1.0.0.jar
java -jar servicio-clinico/target/nutrivida-servicio-clinico-1.0.0.jar
java -jar servicio-agenda/target/nutrivida-servicio-agenda-1.0.0.jar
```

Los puertos pueden cambiarse con `PUERTO_AUTENTICACION`, `PUERTO_CLINICO` y `PUERTO_AGENDA`. `NUTRIVIDA_URL_CLINICO` indica a los servicios de autenticación y agenda dónde está el servicio clínico (por defecto `http://localhost:8082`).

## Pruebas

Las pruebas de integración usan un servidor MongoDB real y trabajan sobre bases `nutrivida_pruebas_*`. Utilice un servidor desechable, distinto de cualquier base con datos propios, e indíquelo con `NUTRIVIDA_MONGO_SERVIDOR_PRUEBAS`:

```bash
docker run -d --name mongo-pruebas -p 127.0.0.1:27019:27017 mongo:7
export NUTRIVIDA_MONGO_SERVIDOR_PRUEBAS=mongodb://localhost:27019
mvn clean verify
```

Para ejecutar la suite completa con un MongoDB temporal aislado en Docker, sin tocar ninguna base de uso habitual, use `powershell -ExecutionPolicy Bypass -File scripts\verificar-backend.ps1`. El script falla si Maven termina con error, si hay pruebas omitidas o fallidas, o si se ejecutan menos pruebas que los métodos `@Test` declarados, y muestra la cobertura JaCoCo por módulo. Sin MongoDB accesible las pruebas de integración se omiten y Maven informa igualmente `BUILD SUCCESS`.

Resultado verificado de la versión actual: **161 métodos `@Test`, 161 ejecutados, 0 fallos y 0 omitidos** (11 del módulo común, 46 de autenticación, 53 del servicio clínico y 51 de agenda). Cobertura de líneas: autenticación 86,0 %, clínico 92,8 %, agenda 91,5 % y común 20,5 % (este módulo se ejercita sobre todo desde las pruebas de integración de los otros). Cubren registro, inicio y cierre de sesión, revocación de tokens, 401 y 403, reglas de negocio de citas y agenda, aislamiento entre pacientes, versionado de la API y validaciones.

## Swagger y endpoints

Cada servicio expone su documentación OpenAPI y Swagger UI. Con el Compose local:

| Servicio | Swagger UI | Definición OpenAPI |
|---|---|---|
| Autenticación | http://localhost:8081/swagger-ui/index.html | http://localhost:8081/v3/api-docs |
| Clínico | http://localhost:8082/swagger-ui/index.html | http://localhost:8082/v3/api-docs |
| Agenda | http://localhost:8083/swagger-ui/index.html | http://localhost:8083/v3/api-docs |

Para ejecutar endpoints protegidos desde Swagger UI: inicie sesión con `POST /api/v1/autenticacion/iniciar-sesion`, copie el valor de `token` de la respuesta, pulse **Authorize** y péguelo (Swagger añade el prefijo `Bearer`).

Todos los endpoints usan el prefijo versionado `/api/v1`. Resumen por recurso:

| Servicio | Recurso | Operaciones principales |
|---|---|---|
| Autenticación | `/api/v1/autenticacion` | `POST registro`, `POST iniciar-sesion`, `POST cerrar-sesion`, `GET yo` |
| Autenticación | `/api/v1/usuarios` | CRUD de usuarios, `PATCH {id}/rol`, `PATCH {id}/estado`, `PATCH yo/contrasena` |
| Clínico | `/api/v1/servicios` | `GET` público, `POST`, `PUT`, `PATCH {id}/estado`, `DELETE` (desactivación lógica) |
| Clínico | `/api/v1/nutricionistas` | `GET` público, `POST`, `PUT`, `GET/PUT yo`, `PATCH {id}/estado`, `DELETE` |
| Clínico | `/api/v1/pacientes` | búsqueda, `GET/PUT yo`, `GET/PUT {id}`, autorización de nutricionistas, `DELETE` |
| Clínico | `/api/v1/pacientes/{id}/ficha-clinica` | apertura, consulta, corrección, evoluciones y correcciones de evoluciones |
| Clínico | `/api/v1/pacientes/{id}/mediciones` | registro, listado, evolución y anulación |
| Clínico | `/api/v1/pacientes/{id}/planes-alimentarios`, `/api/v1/planes-alimentarios/{id}` | creación, historial, plan vigente, nueva versión y finalización |
| Clínico | `/api/v1/solicitudes-contacto` | `POST` público; listado y cambio de estado para administradores |
| Clínico | `/api/v1/auditorias`, `/api/v1/reportes/pacientes` | consulta para administradores |
| Agenda | `/api/v1/agendas/{nutricionistaId}` | consulta, horarios, bloqueos y disponibilidad |
| Agenda | `/api/v1/citas` | solicitud, listado, consulta, confirmación, rechazo, cancelación, reprogramación, atención realizada, inasistencia y pagos |
| Agenda | `/api/v1/pagos`, `/api/v1/reportes/*` | consulta y anulación de pagos simulados; reportes de actividad |

El catálogo completo, con roles permitidos y códigos de respuesta, se obtiene de la definición OpenAPI de cada servicio. Las rutas `/api/v1/internos/**` del servicio clínico son de uso exclusivo entre microservicios.

### Códigos de respuesta

| Código | Uso |
|---|---|
| 200, 201, 204 | Consulta o modificación correcta, recurso creado, operación sin contenido. |
| 400 | Datos inválidos o formato incorrecto. |
| 401 | Falta el token, es inválido, expiró o fue revocado. |
| 403 | El token es válido, pero el rol o la propiedad del recurso no permiten la operación. |
| 404 | El recurso o la ruta no existe. |
| 409 | Conflicto de negocio: correo o RUT duplicado, horario ocupado u operación no permitida en el estado actual. |
| 503 | Un servicio del que depende la operación no está disponible. |

Los errores se devuelven con un cuerpo uniforme: `fecha`, `estado`, `codigo`, `mensaje`, `ruta` y `detalles`.

## Integración con el frontend

- CORS permite únicamente los orígenes de `NUTRIVIDA_ORIGENES_PERMITIDOS`, los métodos `GET`, `POST`, `PUT`, `PATCH`, `DELETE` y `OPTIONS` y los encabezados `Authorization` y `Content-Type`.
- En producción, el frontend publica los tres servicios bajo un mismo origen (`/autenticacion/`, `/clinico/` y `/agenda/`) mediante un proxy inverso.
- Las respuestas listadas están paginadas con `contenido`, `pagina`, `tamano`, `totalElementos` y `totalPaginas`.

## Seguridad

- **Contraseñas:** se almacenan con BCrypt y nunca se devuelven en las respuestas. El registro exige entre 8 y 72 caracteres con letras y números.
- **JWT:** se firma con la clave `NUTRIVIDA_JWT_SECRETO` e incluye identificador del token, emisor, usuario, correo, rol, fecha de emisión y de expiración. La validación comprueba firma, emisor, vigencia y que el token no esté revocado.
- **Sin estado:** la sesión de Spring Security es `STATELESS` y cada solicitud se autentica con el encabezado `Authorization: Bearer`. Por ser una API sin cookies de sesión, CSRF está deshabilitado.
- **Revocación:** `POST cerrar-sesion` guarda el identificador del token en la colección `tokens_revocados` hasta su expiración y revoca la sesión de renovación.
- **Renovación:** `POST /api/v1/autenticacion/refrescar` exige la cookie HttpOnly `nutrivida_refresco` y la cabecera `X-Nutrivida-Csrf: 1`. El refresh token es opaco, se guarda solo como hash SHA-256 en `tokens_refresco`, rota en cada uso (consumo atómico), revoca toda la familia si se reutiliza uno rotado y no extiende el límite absoluto de la sesión. Desactivar a un usuario o cambiar su rol revoca sus sesiones de renovación; el token de acceso ya emitido sigue válido hasta expirar.
- **Recordatorios:** `GET /api/v1/citas/recordatorios` (paciente) lista las citas confirmadas propias que comienzan dentro de la ventana configurada. Se muestran en la aplicación; no se envían correos ni mensajes.
- **Autorización:** cada endpoint sensible declara el rol requerido con `@PreAuthorize`, y los servicios verifican además la propiedad del recurso (por ejemplo, un paciente solo accede a su propia ficha y un nutricionista solo a pacientes que lo autorizaron).
- **Rutas públicas:** solo las configuradas explícitamente en `application.yml` (catálogo, profesionales, registro, inicio de sesión y contacto).
- **Datos y validación:** los DTO validan la entrada con Jakarta Validation, incluyendo el RUT chileno, y MongoDB aplica validadores de esquema e índices únicos.
- **Auditoría:** las operaciones relevantes quedan registradas en la colección `auditorias`.
- **Contenedores:** las imágenes ejecutan la aplicación con un usuario sin privilegios.

## Roles y permisos

| Rol | Alcance |
|---|---|
| `PACIENTE` | Su propio perfil, citas, pagos simulados, ficha clínica, mediciones y planes. |
| `NUTRICIONISTA` | Su perfil y agenda, citas que le corresponden y pacientes que lo autorizaron: ficha, evoluciones, mediciones y planes. |
| `ADMINISTRADOR` | Usuarios, roles, nutricionistas, servicios, autorizaciones, pagos, solicitudes de contacto, auditoría y reportes. |
| `SERVICIO_INTERNO` | Rol técnico para las llamadas entre microservicios; no se asigna a personas. |

## Base de datos

MongoDB almacena los datos en la base `nutrivida`. Los servicios crean los índices y aplican los validadores de esquema al iniciar (`colecciones-con-esquema`). Colecciones principales: `usuarios`, `tokens_revocados`, `pacientes`, `nutricionistas`, `servicios`, `fichas_clinicas`, `mediciones`, `planes_alimentarios`, `solicitudes_contacto`, `agendas`, `ocupaciones_agenda`, `citas`, `pagos` y `auditorias`. Las relaciones entre documentos se hacen mediante referencias por identificador.

La carpeta `mongodb/` contiene scripts de `mongosh` para crear colecciones, validadores e índices, cargar datos ficticios y ejecutar consultas de ejemplo. `scripts/generar-script-validadores.mjs` regenera el primero a partir de los esquemas JSON de `comun/src/main/resources/esquemas`.

## Despliegue en AWS

`despliegue/aws/backend/` contiene un `docker-compose.yml` para ejecutar los tres servicios en una instancia EC2 usando MongoDB Atlas, junto con su `.env.example`. Define límites de memoria, comprobaciones de salud, reinicio automático y publica los puertos solo en la dirección indicada por `IP_ESCUCHA`. La variable `REGISTRO_IMAGENES` debe apuntar al repositorio de imágenes de su cuenta de Docker Hub.

## Estructura del proyecto

```
back-nutrivida/
├── comun/                   Módulo compartido (seguridad, errores, validación, auditoría, OpenAPI)
├── servicio-autenticacion/  Microservicio de autenticación
├── servicio-clinico/        Microservicio clínico
├── servicio-agenda/         Microservicio de agenda
├── mongodb/                 Scripts de mongosh
├── scripts/                 Datos ficticios y generador de validadores
├── despliegue/aws/backend/  Compose y plantilla de variables para AWS
├── docker-compose.yml       Entorno local completo con MongoDB
├── Dockerfile               Imagen parametrizada por servicio (ARG SERVICIO)
└── pom.xml                  Proyecto Maven multimódulo
```

Para construir la imagen de un servicio:

```bash
docker build --build-arg SERVICIO=servicio-clinico -t nutrivida-clinico .
```

## Limitaciones conocidas

- No existe un endpoint de renovación de tokens: al vencer el JWT hay que iniciar sesión de nuevo.
- Las rutas de documentación Swagger son públicas en los servicios; los endpoints protegidos exigen token al ejecutarse.
- Los recordatorios automáticos de citas no están implementados y los pagos son simulados.
- No hay integración continua configurada ni pruebas de carga.
- El proyecto no debe usarse con datos clínicos reales sin una revisión formal de seguridad y privacidad.
