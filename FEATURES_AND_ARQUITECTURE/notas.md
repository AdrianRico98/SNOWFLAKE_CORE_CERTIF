# ¿Qué es Snowflake?

Snowflake es una **plataforma de datos cloud-native ofrecida como SaaS**. A diferencia de un data warehouse tradicional, soporta múltiples workloads: data warehouse (SQL, ACID), data lake (semi-estructurados, JSON, Avro), data engineering (Snowpipe, Tasks, Streams), data science (Snowpark, integración ML), data sharing (marketplace) y application development (UDFs, stored procedures, conectores).

**Cloud-native significa que fue construido desde cero para la nube**, no es un port de tecnología legacy (NO es "cloud-wash" - importante para el examen). Corre en AWS, Azure o GCP, heredando alta disponibilidad, escalabilidad, elasticidad y durabilidad de la nube.

Como **SaaS puro**, no hay gestión de infraestructura ni patches manuales. Updates semanales transparentes sin downtime. Modelo pay-as-you-go. Acceso simple vía UI o conectores programáticos.

**Key exam points:** Cloud-native (no cloud-wash), data platform (no solo warehouse), SaaS puro, optimizaciones/mejoras automáticas.

# Multi-Cluster Shared Data Architecture

Snowflake usa una arquitectura nueva construida específicamente para la nube, diferente a las tradicionales shared-disk y shared-nothing.

**Arquitecturas tradicionales:** Shared-disk tiene storage centralizado (simple pero con punto único de fallo, latencia de red, limitaciones de escalabilidad). Shared-nothing tiene storage y compute en cada nodo (mejor rendimiento pero acoplado, difícil redistribuir datos, tendencia a over-provisioning, scaling limitado).

**Arquitectura de Snowflake:** Multi-cluster Shared Data con tres capas físicamente separadas pero lógicamente integradas:

1. **Storage Layer** (amarillo): Almacenamiento centralizado en la nube con todos los datos de tablas, similar a shared-disk.
Profundizando más en la storage layer: 
La storage layer es el blob storage del cloud provider donde está deployada tu cuenta (S3 en AWS, Azure Blob en Azure, GCS en GCP). Heredamos la escalabilidad casi infinita, disponibilidad y durabilidad del proveedor. Por ejemplo, S3 replica datos en tres availability zones físicamente separadas.

Los datos se organizan en databases, schemas y tables. Snowflake soporta archivos estructurados (CSV, TSV) y semi-estructurados (JSON, Avro, Parquet). Al cargar datos, Snowflake los reorganiza transparentemente a su **formato columnar propietario**, que permite read optimization (saltar columnas no necesarias en queries OLAP). Los archivos se comprimen automáticamente (más eficiente en columnar) y se encriptan por defecto con AES-256.

Los datos se dividen en **micro-partitions** para que Snowflake ignore particiones innecesarias durante queries. Todo este proceso (encriptación, compresión, columnar storage, micro-partitioning) es automático y gestionado por Snowflake. Los datos solo son accesibles vía SQL, no directamente en blob storage correspondiente.

**Billing:** Tarifa plana por TB/mes de datos almacenados, calculado a fin de mes.

3. **Query Processing Layer**: Múltiples compute clusters llamados Virtual Warehouses, ejecutan queries, tienen acceso consistente al storage y cachean datos localmente, similar a shared-nothing.
4. **Cloud Services Layer**: Coordina todo - autenticación, infraestructura, optimización de queries.

**Ventajas clave:** Storage y compute **desacoplados** (se escalan independientemente), sin límites duros de escalabilidad, múltiples clusters aislados (workload isolation - analytics y ETL no compiten por recursos). Arquitectura service-oriented donde cada capa es un servicio físico separado comunicándose vía REST.

**Exam focus:** Este desacoplamiento y la capacidad multi-cluster son los diferenciadores principales de Snowflake, resultado directo de ser cloud-native.
