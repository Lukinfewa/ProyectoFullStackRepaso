# Capítulo 3. Configuración del Entorno de Desarrollo

---

🎯 **Objetivo del capítulo:** Crear el proyecto Maven, configurar las dependencias necesarias, conectar con MySQL y crear la clase utilitaria `HibernateUtil`. Al terminar este capítulo tendrás el esqueleto del proyecto listo para empezar a crear entidades.

---

## 3.1. Requisitos previos

Antes de empezar, asegúrate de tener instalado:

| Herramienta | Versión | Para qué la necesitas |
|-------------|---------|----------------------|
| **Java JDK** | 17 o superior | Lenguaje de programación |
| **IntelliJ IDEA** | 2025.2.2 Ultimate Edition | IDE de desarrollo |
| **Docker Desktop** | 4.x o superior | Para ejecutar MySQL en contenedor |
| **Maven** | Incluido en IntelliJ | Gestión de dependencias |

> 💡 **TIP:** IntelliJ IDEA 2025 Ultimate incluye Maven integrado, soporte para Docker,
> herramientas de base de datos, y Spring Initializr. No necesitas instalar nada adicional.

### 3.1.1. Configurar el JDK en IntelliJ

1. Abre IntelliJ IDEA 2025.2.2
2. Ve a `File > Project Structure...` (o `Ctrl+Alt+Shift+S`)
3. En el panel izquierdo selecciona **SDKs**
4. Si no tienes JDK 17 configurado, haz clic en `+` > `Download JDK...`
5. Selecciona **Version: 17** y **Vendor: Eclipse Temurin** (o Amazon Corretto)
6. Haz clic en `Download`

> ⚠️ **CUIDADO:** Este proyecto usa **Java 17**, no Java 11. Asegúrate
> de tener configurado JDK 17 o superior como SDK del proyecto.

### 3.1.2. Verificar Maven integrado

1. Ve a `File > Settings` (o `Ctrl+Alt+S`)
2. Navega a `Build, Execution, Deployment > Build Tools > Maven`
3. Verifica que **Maven home path** apunta al Maven incluido en IntelliJ (*Bundled (Maven 3)*)
4. No necesitas cambiar nada — la configuración por defecto funciona

---

## 3.2. Crear un proyecto Maven en IntelliJ IDEA 2025

1. Abre IntelliJ IDEA y selecciona `File > New > Project...`
2. En el panel izquierdo selecciona **New Project**
3. Configura:
   - **Name:** `gestion-maritima`
   - **Location:** elige la carpeta donde quieres guardar el proyecto
   - **Language:** Java
   - **Build system:** Maven
   - **JDK:** 17 (el que configuraste en 3.1.1)
   - **GroupId:** `com.mycompany.hibernate`
   - **ArtifactId:** `gestion-maritima`
4. Haz clic en `Create`

> 📖 **Vocabulario:** **Maven** es una herramienta de gestión de proyectos Java que se encarga de descargar las librerías (dependencias) que necesita tu proyecto, compilar el código y empaquetar la aplicación. Todo se configura en un único archivo: `pom.xml`.

> 📖 **Vocabulario:** El **GroupId** identifica a tu organización o grupo (como un paquete Java). El **ArtifactId** es el nombre del proyecto.

---

## 3.3. Dependencias en `pom.xml`

El archivo `pom.xml` es el **corazón de Maven**. Aquí defines qué librerías necesita tu proyecto. Maven las descarga automáticamente de internet.

Edita tu `pom.xml` para que tenga este contenido:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.mycompany.hibernate</groupId>
    <artifactId>gestion-maritima</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- 1. Hibernate Core: el ORM que mapea objetos Java ↔ tablas SQL -->
        <dependency>
            <groupId>org.hibernate</groupId>
            <artifactId>hibernate-core</artifactId>
            <version>5.6.0.Final</version>
        </dependency>

        <!-- 2. MySQL Connector/J: driver JDBC para comunicarse con MySQL -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.49</version>
        </dependency>

        <!-- 3. Lombok: genera getters, setters, constructores automáticamente -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
        </dependency>

        <!-- 4. JUnit 5: framework para pruebas unitarias -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.9.2</version>
            <scope>test</scope>
        </dependency>

        <!-- 5. Mockito: crear objetos simulados (mocks) para tests -->
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>5.3.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

Vamos a explicar cada dependencia:

### 3.3.1. Hibernate Core

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.6.0.Final</version>
</dependency>
```

Es la librería principal del ORM. La versión `5.6.0.Final` indica que es una versión **estable** y lista para producción.

> 📖 **Vocabulario:** Las versiones marcadas como **Final** en Hibernate significan que son versiones estables, listas para usar en proyectos reales (a diferencia de versiones *Alpha*, *Beta* o *CR* que son de prueba).

### 3.3.2. MySQL Connector/J

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.49</version>
</dependency>
```

Es el **driver JDBC** oficial de MySQL para Java. Sin este conector, Java no puede "hablar" con MySQL. Hibernate lo necesita internamente para establecer la conexión.

> 📖 **Vocabulario:** Un **driver JDBC** es un adaptador que permite a Java comunicarse con un motor de base de datos específico. Cada base de datos tiene su propio driver (MySQL, PostgreSQL, Oracle...).

### 3.3.3. Lombok

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.20</version>
</dependency>
```

Lombok es una librería que **genera código automáticamente** en tiempo de compilación. En lugar de escribir manualmente los getters, setters, constructores, `equals()`, `hashCode()` y `toString()`, usas una anotación y Lombok los genera por ti.

> 💡 **TIP:** En **IntelliJ IDEA 2025.2**, el plugin de Lombok viene **preinstalado**.
> Solo necesitas verificar que la opción `Enable annotation processing` esté activada:
> `File > Settings > Build, Execution, Deployment > Compiler > Annotation Processors` → ✅ marcar.

### 3.3.4. JUnit 5 y Mockito

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.9.2</version>
    <scope>test</scope>
</dependency>
```

Fíjate en `<scope>test</scope>`: esto significa que estas librerías **solo se usan para las pruebas** y no se incluyen en la aplicación final. Las usaremos en el Capítulo 9.

> 📖 **Vocabulario:** El **scope** (ámbito) en Maven define cuándo está disponible una dependencia. `test` significa que solo está disponible para ejecutar tests, no en el código principal de la aplicación.

---

## 3.4. Archivo de configuración `hibernate.cfg.xml`

Hibernate necesita saber cómo conectarse a la base de datos, qué dialecto SQL usar, y qué entidades tiene que gestionar. Todo esto se configura en el archivo `hibernate.cfg.xml`.

Crea este archivo en la carpeta `src/main/resources/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
        "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
        "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>

        <!-- Conexión a la base de datos -->
        <property name="hibernate.connection.driver_class">
            com.mysql.cj.jdbc.Driver
        </property>
        <property name="hibernate.connection.url">
            jdbc:mysql://localhost:3306/gestion_maritima?serverTimezone=UTC
        </property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">root</property>

        <!-- Dialecto SQL -->
        <property name="hibernate.dialect">
            org.hibernate.dialect.MySQL57Dialect
        </property>

        <!-- Gestión automática de tablas -->
        <property name="hibernate.hbm2ddl.auto">update</property>

        <!-- Mostrar SQL en consola -->
        <property name="show_sql">true</property>

        <!-- Entidades mapeadas (las iremos añadiendo) -->
        <!-- <mapping class="com.mycompany.hibernate.modelo.Barco" /> -->
        <!-- <mapping class="com.mycompany.hibernate.modelo.Amarre" /> -->
        <!-- <mapping class="com.mycompany.hibernate.modelo.Regata" /> -->

    </session-factory>
</hibernate-configuration>
```

> ⚠️ **CUIDADO:** Antes de ejecutar el proyecto, debes crear la base de datos en MySQL. Abre el cliente de MySQL y ejecuta: `CREATE DATABASE gestion_maritima;`

Vamos a explicar cada propiedad:

### 3.4.1. Conexión a la base de datos

```xml
<property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
<property name="hibernate.connection.url">
    jdbc:mysql://localhost:3306/gestion_maritima?serverTimezone=UTC
</property>
<property name="hibernate.connection.username">root</property>
<property name="hibernate.connection.password">root</property>
```

| Propiedad | Valor | Significado |
|-----------|-------|-------------|
| `driver_class` | `com.mysql.cj.jdbc.Driver` | Clase del driver JDBC de MySQL |
| `url` | `jdbc:mysql://localhost:3306/gestion_maritima` | Dirección de la BD (servidor:puerto/nombre) |
| `username` | `root` | Usuario de la BD |
| `password` | `root` | Contraseña de la BD |

> 📖 **Vocabulario:** La **URL de conexión JDBC** sigue el formato `jdbc:motor://host:puerto/basededatos`. El parámetro `serverTimezone=UTC` evita problemas con las zonas horarias.

### 3.4.2. Dialecto SQL

```xml
<property name="hibernate.dialect">org.hibernate.dialect.MySQL57Dialect</property>
```

El **dialecto** le dice a Hibernate qué variante de SQL debe generar. Cada motor de base de datos tiene pequeñas diferencias en su SQL, y el dialecto se encarga de ajustar las consultas a cada motor.

> 📖 **Vocabulario:** Un **dialecto** (*dialect*) en Hibernate es una clase que traduce las operaciones de Hibernate al SQL específico de una base de datos concreta. Si cambias de MySQL a PostgreSQL, solo tienes que cambiar el dialecto.

| Base de datos | Dialecto |
|--------------|----------|
| MySQL 5.7+ | `org.hibernate.dialect.MySQL57Dialect` |
| MySQL 8+ | `org.hibernate.dialect.MySQL8Dialect` |
| PostgreSQL | `org.hibernate.dialect.PostgreSQLDialect` |
| Oracle | `org.hibernate.dialect.Oracle12cDialect` |
| H2 (en memoria) | `org.hibernate.dialect.H2Dialect` |

### 3.4.3. Propiedad `hbm2ddl.auto`

```xml
<property name="hibernate.hbm2ddl.auto">update</property>
```

Esta propiedad controla **qué hace Hibernate con las tablas de la base de datos al arrancar**:

| Valor | Comportamiento | Cuándo usarlo |
|-------|---------------|---------------|
| `update` | Crea tablas si no existen, modifica si cambian | ✅ **Desarrollo** (el que usaremos) |
| `create` | Borra y crea las tablas cada vez que arranca | Tests (datos se pierden) |
| `create-drop` | Crea al arrancar, borra al cerrar | Tests temporales |
| `validate` | Solo valida que las tablas coincidan con las entidades | Producción |
| `none` | No hace nada | Producción (gestión manual) |

> ⚠️ **CUIDADO:** Nunca uses `create` o `create-drop` en un entorno de producción. Borrarás todos los datos cada vez que la aplicación arranque.

> 💡 **TIP:** Durante el desarrollo, `update` es la opción más cómoda. Hibernate creará las tablas automáticamente a partir de tus entidades Java. No necesitas escribir `CREATE TABLE` manualmente.

### 3.4.4. Mostrar SQL en consola

```xml
<property name="show_sql">true</property>
```

Con esta propiedad activada, Hibernate imprime en la consola **todas las consultas SQL que genera**. Esto es muy útil para aprender y depurar.

Ejemplo de lo que verás en consola:

```
Hibernate: insert into barco (capacidad, eslora, manga, nombre, tipo) values (?, ?, ?, ?, ?)
Hibernate: select barco0_.id as id1_0_0_, barco0_.nombre as nombre2_0_0_ from barco barco0_ where barco0_.id=?
```

> 🏆 **Buena práctica:** Mantén `show_sql=true` durante el desarrollo para verificar que Hibernate genera las consultas que esperas. Desactívalo en producción para no saturar los logs.

### 3.4.5. Registro de entidades mapeadas

```xml
<mapping class="com.mycompany.hibernate.modelo.Barco" />
```

Cada entidad que quieras persistir debe estar registrada en este archivo. Lo haremos en el **Capítulo 4** cuando creemos las entidades.

---

## 3.5. Clase `HibernateUtil` (patrón Singleton)

Necesitamos una clase que se encargue de **crear y gestionar** la conexión de Hibernate con la base de datos. Esta clase usa el patrón **Singleton**, que garantiza que solo exista **una única instancia** de la fábrica de sesiones en toda la aplicación.

> 📖 **Vocabulario:** El patrón **Singleton** es un patrón de diseño que restringe la creación de una clase a una sola instancia. Esto se usa cuando solo necesitas un objeto compartido por toda la aplicación, como la conexión a la base de datos.

Crea la clase `HibernateUtil.java` en el paquete `com.mycompany.hibernate.util`:

```java
package com.mycompany.hibernate.util;

import org.hibernate.SessionFactory;
import org.hibernate.boot.Metadata;
import org.hibernate.boot.MetadataSources;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;

public class HibernateUtil {

    // Única instancia de SessionFactory (Singleton)
    private static final SessionFactory sessionFactory = buildSessionFactory();

    /**
     * Construye la SessionFactory leyendo hibernate.cfg.xml.
     * Se ejecuta UNA SOLA VEZ al cargar la clase.
     */
    private static SessionFactory buildSessionFactory() {
        try {
            // 1. Leer la configuración de hibernate.cfg.xml
            StandardServiceRegistry standardRegistry =
                    new StandardServiceRegistryBuilder()
                            .configure("hibernate.cfg.xml")
                            .build();

            // 2. Construir los metadatos (info sobre las entidades)
            Metadata metadata = new MetadataSources(standardRegistry)
                    .getMetadataBuilder()
                    .build();

            // 3. Crear la SessionFactory
            return metadata.getSessionFactoryBuilder().build();

        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException("Error al crear la SessionFactory de Hibernate.");
        }
    }

    /**
     * Devuelve la única instancia de SessionFactory.
     * Desde cualquier parte de la aplicación: HibernateUtil.getSessionFactory()
     */
    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }
}
```

Vamos a entender qué hace cada parte:

### 3.5.1. Construcción del `SessionFactory`

```java
private static final SessionFactory sessionFactory = buildSessionFactory();
```

La variable `sessionFactory` es **estática y final**: se crea una sola vez cuando la clase se carga en memoria, y no cambia nunca. El método `buildSessionFactory()` lee el archivo `hibernate.cfg.xml` y construye la fábrica de sesiones.

> 📖 **Vocabulario:** La **SessionFactory** es el objeto central de Hibernate. Es una fábrica que crea **sesiones** (Session) para interactuar con la base de datos. Crear una SessionFactory es costoso, por eso se crea una sola vez y se reutiliza.

> 📖 **Vocabulario:** Una **Session** (sesión) es una conexión de trabajo con la base de datos. Se abre, se realizan operaciones (guardar, leer, borrar...) y se cierra. Es ligera y se puede crear muchas veces.

```
  SessionFactory (1 sola, pesada)
       │
       ├── crea ──▶ Session 1 (ligera, temporal)
       ├── crea ──▶ Session 2
       └── crea ──▶ Session 3 ...
```

### 3.5.2. Método `getSessionFactory()`

```java
public static SessionFactory getSessionFactory() {
    return sessionFactory;
}
```

Este método público permite que cualquier parte de la aplicación obtenga la SessionFactory para crear sesiones:

```java
// Ejemplo de uso desde cualquier clase:
Session session = HibernateUtil.getSessionFactory().openSession();
```

> 🔍 **¿Sabías que?** En versiones antiguas de Hibernate se usaba `SessionFactory sf = new Configuration().configure().buildSessionFactory()`. Esa forma está **deprecada** (obsoleta). La forma moderna con `StandardServiceRegistryBuilder` es la que usamos aquí y es la **recomendada** desde Hibernate 5.

---

## 3.6. Estructura del proyecto

Con todo lo que hemos creado en este capítulo, tu proyecto debería tener esta estructura:

```
gestion-maritima/
├── pom.xml                              ← Dependencias Maven
└── src/
    ├── main/
    │   ├── java/
    │   │   └── com/mycompany/hibernate/
    │   │       ├── modelo/              ← Aquí irán las entidades (Cap. 4)
    │   │       ├── dao/                 ← Aquí irán las interfaces DAO (Cap. 7)
    │   │       ├── dao/impl/            ← Aquí irán las implementaciones DAO (Cap. 7)
    │   │       ├── util/
    │   │       │   └── HibernateUtil.java  ← ✅ Creado en este capítulo
    │   │       └── Main.java            ← Clase principal (Cap. 6)
    │   └── resources/
    │       └── hibernate.cfg.xml        ← ✅ Creado en este capítulo
    └── test/
        └── java/                        ← Tests unitarios (Cap. 9)
```

> 💡 **TIP:** Crea ya las carpetas `modelo`, `dao`, `dao/impl` y `util` aunque estén vacías. Tener la estructura preparada desde el principio te ayuda a mantener el proyecto organizado.

> 🏆 **Buena práctica:** La convención de nombres de paquetes en Java es usar el **dominio inverso** de tu organización: `com.mycompany.hibernate`. Esto evita conflictos con librerías de otros desarrolladores.

---

## 3.7. Docker: MySQL en contenedor

En lugar de instalar MySQL directamente en tu equipo, usaremos **Docker** para
ejecutar MySQL en un contenedor aislado. Esto tiene tres ventajas:

1. **No contaminas** tu sistema con instalaciones
2. **Reproducible**: cualquiera puede arrancar el mismo entorno
3. **Fácil de borrar**: si algo va mal, destruyes el contenedor y creas otro

📋 **ARCHIVO DEL PROYECTO** — Crea este archivo en la raíz del proyecto:
`docker-compose.yml`

```yaml
# Docker Compose para el proyecto de Gestión Marítima
# Uso: docker-compose up -d
# Acceso: mysql -h localhost -P 3306 -u root -p root

services:
  mysql:
    image: mysql:8.0
    container_name: mysql-maritima
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: gestion_maritima
      MYSQL_USER: maritima
      MYSQL_PASSWORD: maritima
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password

volumes:
  mysql_data:
```

Para arrancar MySQL:

```bash
docker-compose up -d
```

> 📖 **Vocabulario:** **Docker** es una plataforma que ejecuta aplicaciones en
> **contenedores** — entornos aislados y ligeros. **Docker Compose** permite
> definir servicios (como MySQL) en un fichero YAML y arrancarlos con un solo comando.

### 3.7.1. Docker desde IntelliJ Ultimate

IntelliJ 2025 Ultimate incluye soporte nativo para Docker:

1. Ve a `View > Tool Windows > Services`
2. Haz clic en `+` > `Docker connection`
3. Selecciona tu Docker Desktop local > `OK`
4. Haz clic derecho en `docker-compose.yml` > `Run 'docker-compose.yml'`

El contenedor MySQL arrancará y lo verás en la pestaña **Services** de IntelliJ.

### 3.7.2. Conectar a MySQL desde IntelliJ (Database Tool)

IntelliJ Ultimate incluye un cliente de base de datos integrado:

1. Ve a `View > Tool Windows > Database`
2. Haz clic en `+` > `Data Source` > `MySQL`
3. Configura:
   - **Host:** `localhost`
   - **Port:** `3306`
   - **User:** `root`
   - **Password:** `root`
   - **Database:** `gestion_maritima`
4. Haz clic en `Test Connection` para verificar
5. Si te pide descargar el driver, acepta
6. Haz clic en `OK`

Ahora puedes ver las tablas, ejecutar consultas SQL y explorar los datos
directamente desde IntelliJ sin necesidad de un cliente externo.

> 🏆 **Buena práctica:** Usar el Database Tool de IntelliJ te permite verificar
> rápidamente que Hibernate ha creado las tablas correctamente y que los datos
> se están guardando como esperas.

---

## 📝 Glosario del Capítulo 3

| Término | Definición |
|---------|------------|
| **Maven** | Herramienta de gestión de proyectos Java que maneja dependencias, compilación y empaquetado |
| **`pom.xml`** | *Project Object Model* — archivo central de Maven que define la configuración del proyecto y sus dependencias |
| **Dependencia** | Librería externa que tu proyecto necesita para funcionar (ej: Hibernate, MySQL Connector) |
| **Scope** | Ámbito de una dependencia Maven: `compile` (por defecto, siempre disponible) o `test` (solo para tests) |
| **Driver JDBC** | Adaptador que permite a Java comunicarse con una base de datos específica |
| **`hibernate.cfg.xml`** | Archivo de configuración de Hibernate con conexión BD, dialecto y entidades registradas |
| **Dialecto** | Clase que traduce las operaciones de Hibernate al SQL específico de una base de datos |
| **`hbm2ddl.auto`** | Propiedad que controla la gestión automática del esquema de tablas (create, update, validate) |
| **SessionFactory** | Fábrica central de Hibernate que crea sesiones. Costosa de crear, se reutiliza (Singleton) |
| **Session** | Conexión de trabajo con la BD. Ligera, se abre y cierra para cada operación o grupo de operaciones |
| **Singleton** | Patrón que garantiza que solo exista una instancia de una clase en toda la aplicación |
| **Deprecado** | API o método que ha quedado obsoleto y se desaconseja su uso en favor de una alternativa más moderna |

---

✅ **Checkpoint:** Al terminar este capítulo debes tener...
- Un proyecto Maven creado en IntelliJ IDEA con las dependencias en `pom.xml`.
- El archivo `hibernate.cfg.xml` configurado para conectar con MySQL.
- La base de datos `gestion_maritima` creada en MySQL.
- La clase `HibernateUtil.java` funcional.
- La estructura de carpetas del proyecto preparada.
- **Todavía no hay entidades** — las crearemos en el siguiente capítulo.

---

*Capítulo anterior: [← Capítulo 2. Patrones de Diseño para el Acceso a Datos](Cap02_Patrones_DAO.md)*

*Siguiente capítulo: [Capítulo 4. Entidades y Mapeo Objeto-Relacional →](Cap04_Entidades_Mapeo.md)*
