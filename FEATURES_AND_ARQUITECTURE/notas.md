# ¿Qué es Snowflake?

Snowflake es una **plataforma de datos cloud-native ofrecida como SaaS**. A diferencia de un data warehouse tradicional, soporta múltiples workloads: data warehouse (SQL, ACID), data lake (semi-estructurados, JSON, Avro), data engineering (Snowpipe, Tasks, Streams), data science (Snowpark, integración ML), data sharing (marketplace) y application development (UDFs, stored procedures, conectores).

**Cloud-native significa que fue construido desde cero para la nube**, no es un port de tecnología legacy (NO es "cloud-wash" - importante para el examen). Corre en AWS, Azure o GCP, heredando alta disponibilidad, escalabilidad, elasticidad y durabilidad de la nube.

Como **SaaS puro**, no hay gestión de infraestructura ni patches manuales. Updates semanales transparentes sin downtime. Modelo pay-as-you-go. Acceso simple vía UI o conectores programáticos.

**Key exam points:** Cloud-native (no cloud-wash), data platform (no solo warehouse), SaaS puro, optimizaciones/mejoras automáticas.

# Multi-Cluster Shared Data Architecture

Snowflake usa una arquitectura nueva construida específicamente para la nube, diferente a las tradicionales shared-disk y shared-nothing.

**Arquitecturas tradicionales:** Shared-disk tiene storage centralizado (simple pero con punto único de fallo, latencia de red, limitaciones de escalabilidad). Shared-nothing tiene storage y compute en cada nodo (mejor rendimiento pero acoplado, difícil redistribuir datos, tendencia a over-provisioning, scaling limitado).

**Arquitectura de Snowflake:** Multi-cluster Shared Data con tres capas físicamente separadas pero lógicamente integradas:

1. **Storage Layer** : Almacenamiento centralizado en la nube con todos los datos de tablas, similar a shared-disk.
**Profundizando más en la storage layer**: 
La storage layer es el blob storage del cloud provider donde está deployada tu cuenta (S3 en AWS, Azure Blob en Azure, GCS en GCP). Heredamos la escalabilidad casi infinita, disponibilidad y durabilidad del proveedor. Por ejemplo, S3 replica datos en tres availability zones físicamente separadas.

Los datos se organizan en databases, schemas y tables. Snowflake soporta archivos estructurados (CSV, TSV) y semi-estructurados (JSON, Avro, Parquet). Al cargar datos, Snowflake los reorganiza transparentemente a su **formato columnar propietario**, que permite read optimization (saltar columnas no necesarias en queries OLAP). Los archivos se comprimen automáticamente (más eficiente en columnar) y se encriptan por defecto con AES-256.

Los datos se dividen en **micro-partitions** para que Snowflake ignore particiones innecesarias durante queries. Todo este proceso (encriptación, compresión, columnar storage, micro-partitioning) es automático y gestionado por Snowflake. Los datos solo son accesibles vía SQL, no directamente en blob storage correspondiente.

**Billing:** Tarifa plana por TB/mes de datos almacenados, calculado a fin de mes.

2. **Query Processing Layer**: Múltiples compute clusters llamados Virtual Warehouses, ejecutan queries, tienen acceso consistente al storage y cachean datos localmente, similar a shared-nothing.
**Profundizando más en la compute o query processing layer**:
Realiza las tareas de procesamiento sobre los datos del storage layer para responder queries. Consiste en **Virtual Warehouses** (VWs): clusters de compute gestionados por Snowflake que se crean vía SQL. Son abstracciones sobre instancias cloud (EC2 en AWS) que ejecutan el motor de ejecución de Snowflake. Como es SaaS, no tenemos acceso directo a los nodos, solo interactuamos con el objeto warehouse.

Los VWs funcionan similar a shared-nothing: hacen llamadas remotas al storage layer y cachean los datos en almacenamiento local de alta velocidad para queries subsecuentes. Son **ephemeral y flexibles**: se crean/dropean instantáneamente, pueden pausarse (sin cobro en pausa) y resumirse. Esto permite ajustar compute a la demanda real en lugar de provisionar para capacidad pico.

**Ventajas clave:** Múltiples VWs permiten scale-out para workloads concurrentes y **workload isolation** (un VW para data loading, otro para analytics, sin contención de recursos). Cada VW puede tener su propia configuración.

**Tamaños:** Extra Small a 6XL, indicando poder de compute (número y capacidad de instancias). Permite scale-up individual según requisitos del workload.

Todos los VWs tienen acceso a los mismos datos en storage layer simultáneamente. Snowflake usa **procesamiento ACID estricto** (no eventually consistent) mediante el Transaction Manager en Cloud Services Layer, que sincroniza accesos para que todos los updates/inserts estén inmediatamente disponibles para todos los VWs.

3. **Cloud Services Layer**: Coordina todo - autenticación, infraestructura, optimización de queries.
**Porfundizando más en la service layer**
Colección de servicios altamente disponibles y escalables que coordinan actividades across todas las cuentas Snowflake para procesar requests de usuarios. No se explora profundamente en la documentación porque es SaaS (no necesitamos entender los internals). Usa un modelo **global multi-tenancy** en lugar de crear servicios específicos por cuenta, logrando economías de escala y simplificando features como secure data sharing.

**Servicios principales:**
- **Authentication & Access Control:** Verifica identidad, credenciales y privilegios para acciones.
- **Infrastructure Management:** Crea y gestiona recursos cloud subyacentes (blob storage, compute instances).
- **Transaction Management:** Garantiza ACID compliance y acceso consistente de datos por todos los VWs.
- **Metadata Management:** Mantiene información y estadísticas sobre objetos y datos (metadata cache).
- **Query Parsing & Optimization:** Convierte SQL queries en planes ejecutables para VWs.
- **Security:** Maneja encriptación de datos y rotación de keys.

Corre en instancias cloud como los VWs, pero sin control ni visibilidad del usuario.

**Resumen de la Multi-Cluster Shared Data Architecture**

**Ventajas clave:** Storage y compute **desacoplados** (se escalan independientemente), sin límites duros de escalabilidad, múltiples clusters aislados (workload isolation - analytics y ETL no compiten por recursos). Arquitectura service-oriented donde cada capa es un servicio físico separado comunicándose vía REST.

**Exam focus:** Este desacoplamiento y la capacidad multi-cluster son los diferenciadores principales de Snowflake, resultado directo de ser cloud-native.

# Snowflake Editions y Object Model

## Snowflake Editions

Snowflake ofrece cuatro ediciones que afectan el pricing de compute y storage, así como las features disponibles:

**Standard:** Edición introductoria con funcionalidad core. Incluye SQL ANSI estándar (DDL, DML, multi-table insert, merge, windowing functions) y suite de seguridad Continuous Data Protection (Time Travel, network policies).

**Enterprise:** Incluye todo de Standard más features para empresas grandes como multi-cluster warehouses y database failover.

**Business Critical:** Ofrece mayor protección de datos y seguridad para datos muy sensibles o que deben cumplir regulaciones. Incluye private connectivity y customer-managed encryption keys. Tiene todo de Enterprise y Standard.

**VPS (Virtual Private Snowflake):** Máximo nivel de seguridad para organizaciones con requisitos estrictos (gobiernos, instituciones financieras). Incluye todo de Business Critical pero en un **ambiente completamente separado**, sin compartir la services layer con otras cuentas (metadata store tampoco compartido).

## Object Model

Todo en Snowflake es un **objeto** (algo con lo que puedes interactuar vía comandos como CREATE o DROP). Todos los objetos son **securables** (tienen privilegios que se otorgan a roles, roles que se asignan a usuarios, determinando qué pueden ver y hacer).

**Jerarquía de objetos:**
- **Organization:** Feature opcional (no por defecto) para gestionar una o más cuentas Snowflake, se configura vía organization object.
- **Account:** Colección de servicios Snowflake (UI, compute, storage) accesible vía URL, también es un account object para cambiar parámetros a nivel cuenta.
- **Account-level objects:** Objetos que no contienen datos pero configuran la cuenta (users, warehouses, roles, etc.).
- **Databases:** Primera forma de organizar datos, puede haber muchos por cuenta.
- **Schemas:** Subdivisión de databases, un schema pertenece a un database.
- **Schema-level objects:** Tablas (representación lógica de datos almacenados), views, stages, streams, pipes, tasks, y otros objetos con propósitos específicos.

## Jerarquía de Contenedores

### Organization y Account

**Organization:** Metadata container para gestionar una o más cuentas. Se genera automáticamente al crear una cuenta vía self-service. **Single account organization** (una cuenta para empresas pequeñas/medianas) o **multiple account organization** (muchas cuentas para corporaciones con diferentes departamentos/proyectos/locaciones).

Para gestionar múltiples cuentas, las standard accounts tienen el role **org_admin** que permite crear un **organization account** (tipo especial con URL diferente) con el role **global_org_admin**. Desde ahí se usa `CREATE ACCOUNT` para añadir más standard accounts al contenedor de la organización. El organization account gestiona todo el lifecycle: crear, renombrar, eliminar, ver cuentas (`SHOW ACCOUNTS`), configurar database replication/failover, y monitorear usage/billing across cuentas vía organization usage views.

**Account:** Colección privada de servicios con URL única. Asociado a un single cloud provider (AWS/Azure/GCP) y una región geográfica específica. La región afecta pricing, regulatory certifications, features disponibles y network latency. Cada account se crea como una Snowflake edition (puede cambiarse después) y contiene system-defined roles como **account_admin** para gestionar account-level objects.

**Account URL format:** `organization_name-account_name` (strings alfanuméricos de 7 caracteres auto-generados, o nombres personalizados vía Snowflake Support). Para herramientas non-UI (SnowSQL, Python connectors), se usa el **account identifier**: combinación de ambos separados por guion.

### Database

Primer contenedor lógico para datos, agrupa schemas que a su vez contienen schema-level objects (tables, views). Un account puede tener muchas databases.

**Propiedades:** Nombre único en el account, debe empezar con carácter alfabético. No puede contener espacios/caracteres especiales a menos que esté entre comillas dobles (con comillas = case-sensitive, sin comillas = case-insensitive).

**Capacidades:** Pueden crearse desde un clone de otra database en el mismo account, replicarse a otro account (con sus child objects), o crearse desde un share object de otra cuenta Snowflake.

### Schema

Agrupación lógica de database objects (tables, views, stages). Un database puede contener muchos schemas, cada schema pertenece a un database. Permite segmentar aún más los databases.

**Propiedades:** Nombre único dentro del database, mismas reglas de naming que databases (empieza con alfabético, sin espacios/especiales salvo con comillas dobles, case-sensitivity según comillas).

**Namespace:** `database_name.schema_name` forma el namespace. Puedes usarlo como prefijo para referenciar globalmente una tabla, o establecerlo en el contexto (UI/command line) con `USE DATABASE` y `USE SCHEMA`. Una vez establecido, no necesitas calificar table names con database y schema.

**Cloning:** Los schemas pueden clonarse, produciendo una versión espejo del schema y sus child objects.

### Tables

Abstracción lógica sobre los datos en storage layer que describe la estructura de tus datos para consultarlos. Existen cuatro tipos con diferentes requisitos de data retention:

**Permanent Table (default):** Existe hasta que se dropea explícitamente. Time Travel hasta 90 días (Enterprise+) o 1 día (Standard). Tiene 7 días de Fail-Safe (no configurable) donde Snowflake puede restaurar datos borrados.

**Temporary Table:** Para datos transitorios no permanentes (ej: pasos en ETL complejo). Persiste solo durante la sesión, después se purga completamente del sistema. No se puede convertir a otro tipo. Max Time Travel de 1 día (cualquier edition), sin Fail-Safe.

**Transient Table:** Similar a permanent (existe hasta dropeo explícito) pero con max Time Travel de 1 día (cualquier edition) y sin Fail-Safe. Útil para reducir storage costs ya que Fail-Safe contribuye a costos.

**External Table:** Permite consultar datos fuera de Snowflake (ej: Azure container). Overlays estructura sobre datos externos, Snowflake registra metadata de los archivos. Read-only, más lentas, sin Time Travel ni Fail-Safe.

**Identificar tipo:** Comando `SHOW TABLES` (columna kind) o iconos en UI object browser.

### Views

**Standard View:** Almacena definición de query SELECT, no datos. Al consultar la view, recupera datos de la source table. Se usan como tables (joins, subqueries, ORDER BY, GROUP BY, WHERE). No contribuyen a storage costs. Si se dropea la source table, la view retorna error "object does not exist". Función común: restringir contenidos de tabla (subset de columnas/filas).

**Materialized View:** Almacena query definition pero mantiene activamente el resultado (pre-computed dataset). Snowflake cobra compute por proceso background que actualiza periódicamente la view (serverless feature - no usa user-managed virtual warehouses). Costos adicionales de storage para resultados. Puede usarse para boost performance de external tables.

**Secure View:** Tanto standard como materialized pueden designarse secure añadiendo keyword `SECURE`. La query definition solo es visible para usuarios autorizados (comandos como `GET DDL` o `DESCRIBE` no retornan definition a no autoriza

### User-Defined Functions (UDFs)

Schema-level objects que permiten escribir funciones propias en SQL, JavaScript, Python o Java, extendiendo las system-defined functions. Componentes: function name, input parameters (cero o más), return type, y function definition (delimitada por `$$`).

**Tipos de retorno:**
- **Scalar:** Retorna una fila con un valor de una sola columna
- **Tabular (UDTF - User-Defined Table Function):** Retorna cero, una o múltiples filas. Requiere cláusula RETURNS con keyword TABLE especificando nombres y tipos de columnas

**Características:** Pueden llamarse como parte de un SQL statement (a diferencia de stored procedures). Pueden overloadearse (múltiples funciones con mismo nombre si parámetros difieren).

**SQL UDFs:** Language por defecto, no requiere especificar LANGUAGE parameter.

**JavaScript UDFs:** Requiere `LANGUAGE JAVASCRIPT`. Permite branching, looping, error handling y funciones built-in de la librería estándar JavaScript. No puede incluir librerías adicionales. Soporta recursividad (auto-llamarse). Data type mapping necesario entre Snowflake y JavaScript (ej: números se representan como doubles en JS).

**Java UDFs:** (Preview mode) Requiere `LANGUAGE JAVA`. Snowflake spinnea JVM para ejecutar código. Soporta Java 8-11. Restringido a librerías estándar Java. Definition puede ser inline code o pre-compiled code (JAR file) usando parámetros HANDLER (clase y función) y TARGET_PATH (ubicación JAR). **No pueden designarse como secure** (SQL y JavaScript sí).

**External Functions:** UDF que llama código mantenido y ejecutado fuera de Snowflake. Funciona como UDF regular desde perspectiva del usuario. Permite usar lenguajes como Go, C#, y referenciar librerías third-party. Ejemplo: AWS Lambda function.

Requiere **API_INTEGRATION object** con información para autenticarse con cloud provider (AWS, Azure, GCP). Usa **proxy service** (ej: AWS API Gateway) entre Snowflake y código externo como capa adicional de seguridad. Flow: SQL call → API_INTEGRATION credentials → proxy service → remote service (Lambda/Azure Functions/GCP Cloud Functions) → resultado back.

**Limitaciones external functions:** Query optimizer no puede ver código remoto (menos optimizaciones, ejecución más lenta). Latencia por ser remotas. Solo scalar (no tabular). No se pueden compartir vía secure data sharing. Potenciales security concerns (librerías third-party podrían almacenar datos sensibles fuera). Posibles charges por data movement entre cloud platforms/regions.

