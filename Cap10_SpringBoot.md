# Capítulo 10. Introducción a Spring Boot

---

🎯 **Objetivo del capítulo:** Entender qué es Spring Boot, por qué simplifica radicalmente el desarrollo con Hibernate, y migrar nuestro proyecto de "Hibernate puro" a Spring Boot paso a paso.

---

## 10.1. ¿Por qué Spring Boot?

En los capítulos 3-9 construimos un proyecto con **Hibernate puro**. Funcionaba, pero tenía mucha "fontanería":

| Concepto | Hibernate Puro | Spring Boot |
|---------|---------------|-------------|
| Configuración | `hibernate.cfg.xml` manual (74 líneas) | `application.properties` (10 líneas clave) |
| Conexiones | `HibernateUtil` Singleton manual (86 líneas) | Autoconfigurado |
| Acceso a datos | `BarcoDAOImpl` con Session/Transaction (156 líneas) | `BarcoRepository` interfaz (3 líneas) |
| Transacciones | `try { tx = session.beginTransaction(); ... tx.commit(); }` | `@Transactional` |
| Servidor web | No incluido | Tomcat embebido |
| API REST | No incluido | `@RestController` |

> 💡 **¿Sabías que?** Spring Boot fue creado en 2014 por Pivotal (ahora VMware). Su lema es **"Convention over Configuration"**: establece convenciones por defecto para que no tengas que configurar cada detalle. Solo configuras lo que es diferente a la convención.

---

## 10.2. Spring Framework vs Spring Boot

Es importante distinguir entre los dos:

### Spring Framework (2003)
Es el **framework base**. Proporciona:
- **Inyección de dependencias** (IoC Container)
- **AOP** (Programación orientada a aspectos)
- **Spring MVC** para aplicaciones web
- **Spring Data** para acceso a datos

Pero requería mucha configuración XML.

### Spring Boot (2014)
Es una **capa sobre Spring Framework** que añade:
- **Autoconfiguración**: detecta tus dependencias y configura todo automáticamente
- **Starters**: paquetes de dependencias preconfiguradas
- **Servidor embebido**: Tomcat incluido, no necesitas instalar servidor aparte
- **Actuator**: monitorización de la aplicación

```
   Spring Boot
   ┌─────────────────────────┐
   │  Autoconfiguration      │
   │  Starters               │
   │  Embedded Server        │
   │  ┌───────────────────┐  │
   │  │  Spring Framework  │  │ ← Spring MVC, Spring Data, IoC
   │  │  ┌─────────────┐  │  │
   │  │  │  Hibernate   │  │  │ ← JPA Implementation
   │  │  └─────────────┘  │  │
   │  └───────────────────┘  │
   └─────────────────────────┘
```

> 📖 **Vocabulario:**
> - **Starter**: paquete Maven que agrupa varias dependencias relacionadas. Por ejemplo, `spring-boot-starter-data-jpa` incluye Hibernate + HikariCP + Spring Data JPA.
> - **Autoconfiguración**: Spring Boot examina las dependencias del `pom.xml` y configura los beans necesarios automáticamente.
> - **Bean**: objeto gestionado por Spring (creado, inyectado y destruido por el contenedor).

---

## 10.3. Crear un proyecto Spring Boot

### 10.3.1. Desde IntelliJ IDEA 2025 Ultimate (Spring Initializr integrado)

IntelliJ Ultimate incluye **Spring Initializr integrado**. No necesitas ir a la web:

1. `File > New > Project...`
2. En el panel izquierdo selecciona **Spring Boot** (solo disponible en Ultimate)
3. Configura:
   - **Name:** `marina`
   - **Language:** Java
   - **Type:** Maven
   - **Group:** `com.example`
   - **Artifact:** `marina`
   - **JDK:** 17
   - **Java:** 17
   - **Spring Boot:** 3.2.0
4. Haz clic en `Next`
5. Selecciona las **dependencias** marcando las casillas:
   - ✅ **Spring Data JPA** (en *SQL*)
   - ✅ **MySQL Driver** (en *SQL*)
   - ✅ **Spring Web** (en *Web*)
   - ✅ **Thymeleaf** (en *Template Engines*)
   - ✅ **Validation** (en *I/O*)
   - ✅ **Lombok** (en *Developer Tools*)
6. Haz clic en `Create`

> 🏆 **Buena práctica:** Siempre usa Spring Initializr (integrado o web) para crear proyectos nuevos. Genera la estructura correcta con todas las dependencias compatibles entre sí.

> 💡 **TIP:** Si no tienes la edición Ultimate, puedes usar la web https://start.spring.io
> con la misma configuración y luego importar el proyecto descargado en IntelliJ con
> `File > Open`.

### 10.3.2. Estructura del proyecto generado

```
marina/
├── pom.xml                          ← Dependencias Maven
├── src/
│   ├── main/
│   │   ├── java/com/example/marina/
│   │   │   └── MarinaApplication.java  ← Clase principal
│   │   └── resources/
│   │       ├── application.properties  ← Configuración (¡reemplaza hibernate.cfg.xml!)
│   │       ├── static/                 ← CSS, JS, imágenes
│   │       └── templates/              ← Plantillas HTML (Thymeleaf)
│   └── test/
│       └── java/                       ← Tests
└── .mvn/                               ← Maven Wrapper
```

> ⚠️ **CUIDADO:** La clase principal (`MarinaApplication.java`) DEBE estar en el paquete raíz (`com.example.marina`). Spring Boot escanea ese paquete y todos sus subpaquetes. Si la pones en otro sitio, no encontrará tus clases.

---

## 10.4. El `pom.xml` de Spring Boot

Comparemos el `pom.xml` de Hibernate puro con el de Spring Boot:

### 💡 Ejemplo didáctico — Comparación de enfoques:

```xml
<!-- Hibernate Puro: cada versión se gestiona manualmente -->
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.6.15.Final</version>     <!-- ← Versión explícita -->
</dependency>
```

```xml
<!-- Spring Boot: los starters NO necesitan versión -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <!-- SIN <version>: el parent la gestiona -->
</dependency>
```

> 💡 **TIP:** El `spring-boot-starter-parent` es como un "director de orquesta" que se asegura de que todas las versiones de las librerías sean compatibles entre sí. Nunca mezcles versiones manualmente si usas Spring Boot.

### 📋 ARCHIVO DEL PROYECTO — `pom.xml` completo

Este es el `pom.xml` completo que genera Spring Initializr con nuestras dependencias:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- ═══════════════════════════════════════════ -->
    <!-- PARENT: hereda de Spring Boot Starter Parent -->
    <!-- Gestiona automáticamente las versiones de   -->
    <!-- todas las dependencias de Spring             -->
    <!-- ═══════════════════════════════════════════ -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>marina</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Gestión Marítima - Spring Boot</name>
    <description>Proyecto educativo de persistencia con Spring Boot, Spring Data JPA,
        API REST, Swagger y Thymeleaf</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- ═══════════════════════════════════════════ -->
        <!-- 1. SPRING DATA JPA: repositorios automáticos -->
        <!--    Incluye Hibernate como implementación JPA -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 2. SPRING WEB: controladores REST y MVC     -->
        <!--    Incluye Tomcat embebido + Jackson (JSON)  -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 3. THYMELEAF: motor de plantillas HTML       -->
        <!--    Para crear vistas web dinámicas           -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 4. SPRING BOOT VALIDATION: validar datos     -->
        <!--    @NotNull, @Size, @Min, @Max, etc.         -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 5. MYSQL DRIVER: conexión con MySQL Docker   -->
        <!--    scope=runtime: no se necesita en compilación -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 6. LOMBOK: genera código repetitivo          -->
        <!--    @Data, @NoArgsConstructor, @Builder...    -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 7. SPRINGDOC OPENAPI (Swagger UI)            -->
        <!--    Documentación automática e interactiva    -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.3.0</version>
        </dependency>

        <!-- ═══════════════════════════════════════════ -->
        <!-- 8. SPRING BOOT TEST: tests con Spring        -->
        <!--    Incluye JUnit 5, Mockito, AssertJ, etc.   -->
        <!-- ═══════════════════════════════════════════ -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Plugin de compilación con Lombok -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>1.18.42</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### ¿Qué incluye cada Starter?

| Starter | Incluye |
|---------|---------|
| `spring-boot-starter-data-jpa` | Hibernate + HikariCP (pool) + Spring Data JPA |
| `spring-boot-starter-web` | Spring MVC + Tomcat embebido + Jackson (JSON) |
| `spring-boot-starter-thymeleaf` | Motor de plantillas Thymeleaf |
| `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) |
| `spring-boot-starter-test` | JUnit 5 + Mockito + AssertJ + MockMvc |

---

## 10.5. `application.properties` — La nueva configuración

Este fichero **reemplaza** a `hibernate.cfg.xml` Y a `HibernateUtil`:

```properties
# ═══════════════════════════════════════════
# CONEXIÓN A LA BASE DE DATOS (MySQL Docker)
# ═══════════════════════════════════════════
spring.datasource.url=jdbc:mysql://localhost:3306/gestion_maritima?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# ═══════════════════════════════════════════
# CONFIGURACIÓN JPA / HIBERNATE
# ═══════════════════════════════════════════
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
```

### Equivalencia con `hibernate.cfg.xml`:

| hibernate.cfg.xml | application.properties |
|-------------------|----------------------|
| `hibernate.connection.url` | `spring.datasource.url` |
| `hibernate.connection.username` | `spring.datasource.username` |
| `hibernate.connection.password` | `spring.datasource.password` |
| `hibernate.dialect` | `spring.jpa.properties.hibernate.dialect` |
| `hibernate.hbm2ddl.auto` | `spring.jpa.hibernate.ddl-auto` |
| `show_sql` | `spring.jpa.show-sql` |
| `<mapping class="...">` | **No necesario** (escaneo automático) |

> 💡 **¿Sabías que?** Con Spring Boot NO necesitas registrar las entidades manualmente. Spring Boot escanea automáticamente todas las clases con `@Entity` en el paquete de la aplicación y sus subpaquetes.

### Archivo completo

📋 **ARCHIVO DEL PROYECTO** — `src/main/resources/application.properties`

```properties
# ═══════════════════════════════════════════
# CONFIGURACIÓN DE SPRING BOOT
# Archivo principal de configuración.
# Reemplaza a hibernate.cfg.xml + HibernateUtil
# ═══════════════════════════════════════════

# ── NOMBRE DE LA APLICACIÓN ──
spring.application.name=marina

# ═══════════════════════════════════════════
# CONEXIÓN A LA BASE DE DATOS (MySQL Docker)
# Las credenciales deben coincidir con docker-compose.yml
# ═══════════════════════════════════════════
spring.datasource.url=jdbc:mysql://localhost:3306/gestion_maritima?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# ═══════════════════════════════════════════
# CONFIGURACIÓN JPA / HIBERNATE
# Spring Boot configura Hibernate automáticamente
# usando estas propiedades
# ═══════════════════════════════════════════

# ddl-auto: cómo gestionar el esquema de la BD
#   update  = crea/modifica tablas sin borrar datos (desarrollo)
#   create  = borra y recrea cada vez (tests)
#   validate = solo comprueba que las tablas coinciden (producción)
#   none    = no toca el esquema
spring.jpa.hibernate.ddl-auto=update

# Mostrar las consultas SQL en la consola (útil para aprender)
spring.jpa.show-sql=true

# Formatear el SQL para que sea legible
spring.jpa.properties.hibernate.format_sql=true

# Dialecto: le dice a Hibernate qué variante de SQL generar
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect

# ═══════════════════════════════════════════
# CONFIGURACIÓN DEL SERVIDOR WEB
# Spring Boot incluye un Tomcat embebido
# ═══════════════════════════════════════════

# Puerto del servidor (por defecto 8080)
server.port=8080

# ═══════════════════════════════════════════
# CONFIGURACIÓN DE THYMELEAF
# ═══════════════════════════════════════════

# Prefijo: dónde buscar las plantillas HTML
spring.thymeleaf.prefix=classpath:/templates/

# Sufijo: extensión de las plantillas
spring.thymeleaf.suffix=.html

# Cache: desactivar en desarrollo para ver cambios sin reiniciar
spring.thymeleaf.cache=false

# ═══════════════════════════════════════════
# CONFIGURACIÓN DE SWAGGER / OPENAPI
# ═══════════════════════════════════════════

# Ruta personalizada para la documentación JSON
springdoc.api-docs.path=/api-docs

# Ruta de Swagger UI
springdoc.swagger-ui.path=/swagger-ui.html
```

---

## 10.6. La clase principal — `MarinaApplication.java`

```java
package com.example.marina;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication   // ← Esta anotación hace TODA la magia
public class MarinaApplication {

    public static void main(String[] args) {
        SpringApplication.run(MarinaApplication.class, args);
    }
}
```

### ¿Qué hace `@SpringBootApplication`?

Es una **meta-anotación** que combina tres anotaciones:

```java
@SpringBootApplication
    = @Configuration           // Esta clase define configuración
    + @EnableAutoConfiguration // Activa autoconfiguración
    + @ComponentScan           // Escanea el paquete actual y subpaquetes
```

### ¿Qué pasa cuando ejecutas `main()`?

```
1. SpringApplication.run()
      │
2. Escanea com.example.marina.* en busca de:
   ├── @Entity        → Registra entidades JPA
   ├── @Repository    → Crea repositorios
   ├── @Service       → Crea servicios
   ├── @Controller    → Registra controladores web
   └── @RestController → Registra controladores REST
      │
3. Autoconfiguración:
   ├── Detecta `spring-boot-starter-data-jpa` → Configura Hibernate + Pool
   ├── Detecta `spring-boot-starter-web`      → Arranca Tomcat en :8080
   ├── Detecta `spring-boot-starter-thymeleaf` → Configura motor plantillas
   └── Detecta MySQL Driver                   → Configura DataSource
      │
4. Crea las tablas en MySQL (hbm2ddl.auto=update)
      │
5. Ejecuta CommandLineRunner (DataLoader) → Carga datos de ejemplo
      │
6. Escucha peticiones HTTP en :8080
```

> ⚠️ **CUIDADO:** NO necesitas:
> - `hibernate.cfg.xml` → sustituido por `application.properties`
> - `HibernateUtil` → Spring crea y gestiona las conexiones automáticamente
> - Registrar entidades con `<mapping>` → escaneo automático

---

## 10.7. Lo que desaparece vs lo que aparece

### Archivos que YA NO NECESITAS:

| Archivo | ¿Por qué desaparece? |
|---------|---------------------|
| `hibernate.cfg.xml` | Sustituido por `application.properties` |
| `HibernateUtil.java` | Spring gestiona las conexiones (HikariCP) |
| `GenericDAO.java` | Sustituido por `JpaRepository` (Cap. 11) |
| `BarcoDAOImpl.java` | Sustituido por `BarcoRepository` interfaz (Cap. 11) |

### Archivos NUEVOS:

| Archivo | ¿Para qué? |
|---------|------------|
| `MarinaApplication.java` | Punto de entrada de la aplicación |
| `application.properties` | Configuración centralizada |
| `DataLoader.java` | Datos de ejemplo al arrancar |

### Archivos que SE MANTIENEN (pero cambian de paquete):

| Antes | Después | Cambio |
|-------|---------|--------|
| `com.mycompany.hibernate.modelo.Barco` | `com.example.marina.model.Barco` | Cambia `javax.persistence.*` a `jakarta.persistence.*` |

> 📖 **Vocabulario:**
> - **jakarta.persistence**: a partir de Spring Boot 3.x (Jakarta EE 10), el paquete `javax.persistence` se renombró a `jakarta.persistence`. Es exactamente lo mismo, solo cambia el nombre del paquete.

---

## 10.8. Cambio en las entidades: `javax` → `jakarta`

Nuestras entidades cambian **una sola línea** de imports:

```java
// ❌ Hibernate Puro (Spring Boot 2.x o anterior)
import javax.persistence.*;

// ✅ Spring Boot 3.x
import jakarta.persistence.*;
```

El resto de la entidad es **idéntico**:

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Barco {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(nullable = false, length = 50)
    private String tipo;

    // ... (exactamente igual que en Hibernate puro)
}
```

> 💡 **TIP:** Si estás migrando un proyecto de Spring Boot 2.x a 3.x, el único cambio obligatorio en las entidades es reemplazar `javax.persistence` por `jakarta.persistence`. Puedes hacerlo con un "buscar y reemplazar" global.

---

## 10.9. Datos de ejemplo con `CommandLineRunner`

Para que la aplicación tenga datos visibles desde el primer arranque, creamos un `DataLoader`.

> 📖 **Vocabulario:**
> - **`CommandLineRunner`**: interfaz que Spring ejecuta automáticamente al arrancar. Ideal para cargar datos iniciales o ejecutar tareas de inicialización.
> - **`@Component`**: marca una clase como "componente" gestionado por Spring. Spring la detecta en el escaneo automático y crea una instancia.

📋 **ARCHIVO DEL PROYECTO** — Crea este archivo:
`src/main/java/com/example/marina/config/DataLoader.java`

```java
package com.example.marina.config;

import com.example.marina.model.Amarre;
import com.example.marina.model.Barco;
import com.example.marina.model.Regata;
import com.example.marina.repository.AmarreRepository;
import com.example.marina.repository.BarcoRepository;
import com.example.marina.repository.RegataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * Cargador de datos iniciales.
 *
 * CommandLineRunner se ejecuta AUTOMÁTICAMENTE al arrancar la aplicación.
 * Spring detecta esta clase por la anotación @Component y ejecuta run().
 *
 * Carga datos de ejemplo para desarrollo y pruebas.
 * En producción se eliminaría o se controlaría con un perfil (@Profile("dev")).
 */
@Component
public class DataLoader implements CommandLineRunner {

    @Autowired
    private BarcoRepository barcoRepository;

    @Autowired
    private AmarreRepository amarreRepository;

    @Autowired
    private RegataRepository regataRepository;

    @Override
    public void run(String... args) {
        // Solo cargar datos si la BD está vacía
        if (barcoRepository.count() > 0) {
            System.out.println("📦 Datos existentes detectados. No se cargan datos de ejemplo.");
            return;
        }

        System.out.println("📦 Cargando datos de ejemplo...");

        // ═══════════════════════════════════════════
        // BARCOS
        // ═══════════════════════════════════════════
        Barco b1 = barcoRepository.save(new Barco("Estrella del Mar", "Velero", 12, 4, 8));
        Barco b2 = barcoRepository.save(new Barco("Rayo Azul", "Motor", 15, 5, 10));
        Barco b3 = barcoRepository.save(new Barco("Corsario Negro", "Velero", 18, 6, 12));
        Barco b4 = barcoRepository.save(new Barco("Mar Bravo", "Catamarán", 20, 8, 16));
        Barco b5 = barcoRepository.save(new Barco("Poseidón", "Motor", 22, 7, 14));
        Barco b6 = barcoRepository.save(new Barco("La Perla", "Velero", 10, 3, 6));
        Barco b7 = barcoRepository.save(new Barco("Viento Salvaje", "Catamarán", 16, 7, 12));

        // ═══════════════════════════════════════════
        // AMARRES
        // ═══════════════════════════════════════════
        Amarre a1 = new Amarre("A-1", 150.00, 3, 15, true);
        a1.setBarco(b1);
        amarreRepository.save(a1);

        Amarre a2 = new Amarre("A-2", 200.00, 4, 20, true);
        a2.setBarco(b2);
        amarreRepository.save(a2);

        Amarre a3 = new Amarre("B-1", 120.00, 3, 12, false);
        a3.setBarco(b3);
        amarreRepository.save(a3);

        Amarre a4 = new Amarre("B-2", 180.00, 4, 18, true);
        // Amarre libre (sin barco asignado)
        amarreRepository.save(a4);

        Amarre a5 = new Amarre("C-1", 250.00, 5, 25, true);
        a5.setBarco(b4);
        amarreRepository.save(a5);

        Amarre a6 = new Amarre("C-2", 300.00, 5, 30, true);
        // Amarre libre
        amarreRepository.save(a6);

        // ═══════════════════════════════════════════
        // REGATAS
        // ═══════════════════════════════════════════
        Regata r1 = regataRepository.save(new Regata("Copa Mediterráneo", "Valencia", new Date(), 25));
        Regata r2 = regataRepository.save(new Regata("Trofeo Atlántico", "Cádiz", new Date(), 40));
        Regata r3 = regataRepository.save(new Regata("Gran Regata del Sur", "Málaga", new Date(), 30));

        // ═══════════════════════════════════════════
        // INSCRIPCIONES EN REGATAS (N:M)
        // ═══════════════════════════════════════════
        b1.getRegatas().add(r1);
        b1.getRegatas().add(r2);
        barcoRepository.save(b1);

        b2.getRegatas().add(r1);
        barcoRepository.save(b2);

        b3.getRegatas().add(r2);
        b3.getRegatas().add(r3);
        barcoRepository.save(b3);

        b4.getRegatas().add(r1);
        b4.getRegatas().add(r2);
        b4.getRegatas().add(r3);
        barcoRepository.save(b4);

        b7.getRegatas().add(r3);
        barcoRepository.save(b7);

        System.out.println("✅ Datos de ejemplo cargados:");
        System.out.println("   🚢 " + barcoRepository.count() + " barcos");
        System.out.println("   ⚓ " + amarreRepository.count() + " amarres");
        System.out.println("   🏁 " + regataRepository.count() + " regatas");
    }
}
```

---

## 10.10. IoC y Inyección de Dependencias en Spring Boot

En el Capítulo 2 aprendimos los **conceptos teóricos** de la Inversión de Control (IoC)
y la Inyección de Dependencias (DI) usando Java puro. Ahora que tenemos Spring Boot
arrancado, es el momento de ver **cómo Spring implementa estos conceptos en la práctica**.

### 10.10.1. El Principio Hollywood: "No nos llames, ya te llamaremos"

El Principio Hollywood es una forma coloquial de expresar la Inversión de Control.
Su nombre viene de la famosa frase que dicen en los castings de Hollywood a los actores:

> 🎬 ***"Don't call us, we'll call you"***
> — "No nos llames, ya te llamaremos nosotros"

Aplicado a la programación:

```
  Programación TRADICIONAL (Capítulos 2-9):
  ┌─────────────┐          ┌─────────────┐
  │  Tu código  │ ──────►  │  Librería   │
  │  (llama)    │          │  (responde) │
  └─────────────┘          └─────────────┘
  "Tú llamas a la librería cuando quieres"

  Principio HOLLYWOOD (Spring Boot):
  ┌─────────────┐          ┌─────────────┐
  │  Tu código  │ ◄──────  │  Spring     │
  │  (responde) │          │  (te llama) │
  └─────────────┘          └─────────────┘
  "Spring te llama a ti cuando necesita tu código"
```

**Ejemplo concreto de nuestro proyecto:**

Cuando escribimos un `@Controller`, nosotros **nunca** creamos el controlador
ni decidimos cuándo ejecutar sus métodos. Spring nos dice:

> *"Tú implementa el método `listBarcos()` y anótalo con `@GetMapping`.
> Cuando llegue una petición HTTP GET, **yo te llamaré**.
> No te preocupes de cuándo ni cómo."*

Compara esto con Hibernate puro (Capítulo 7), donde tú decidías cuándo crear
la sesión, cuándo abrir la transacción, y cuándo llamar a cada método del DAO.
Con Spring, tú escribes la lógica y **Spring decide cuándo ejecutarla**.

### 10.10.2. El Contenedor IoC: ¿quién crea los objetos?

En el Capítulo 2, cuando inyectábamos dependencias manualmente, **tú** eras
quien creaba los objetos en `main()`:

```java
// Capítulo 2 — TÚ creas y conectas todo
BarcoDAO barcoDAO = new BarcoDAOImpl();
RegataService servicio = new RegataService(barcoDAO);
```

Con Spring Boot, quien crea y conecta los objetos es el **Contenedor IoC**
(*IoC Container*), también llamado **Application Context**:

```
  ┌─────────────────────────────────────────────────────┐
  │            CONTENEDOR IoC (Spring)                   │
  │                                                      │
  │  "Yo me encargo de todo. Tú solo dime QUÉ clases    │
  │   necesitas con @Service, @Controller, @Repository.  │
  │   Yo las creo, las conecto y las gestiono."          │
  │                                                      │
  │  1. Lee las anotaciones (@Service, @Autowired...)    │
  │  2. Crea instancias de las clases anotadas (beans)   │
  │  3. Conecta las dependencias entre ellas             │
  │  4. Gestiona su ciclo de vida                        │
  └─────────────────────────────────────────────────────┘
```

> 📖 **Vocabulario:** Un **bean** es un objeto gestionado por el contenedor
> IoC de Spring. Cualquier clase anotada con `@Service`, `@Controller`,
> `@Repository` o `@Component` se convierte en un bean.

Ya vimos esto en acción en la sección 10.6: cuando ejecutas `SpringApplication.run()`,
Spring escanea todas las clases anotadas, crea beans, y los conecta automáticamente.

### 10.10.3. Tres formas de inyectar dependencias en Spring

En el Capítulo 2 vimos la inyección por constructor con Java puro. Spring soporta
**tres formas** de inyectar dependencias:

| Tipo | Código | ¿Cuándo usar? |
|------|--------|---------------|
| **Por campo** | `@Autowired private DAO dao` | Cómodo, pero menos testeable |
| **Por constructor** | Constructor con parámetros | ✅ **Recomendado** — dependencia obligatoria |
| **Por setter** | Setter con `@Autowired` | Cuando la dependencia es opcional |

```java
// 1. POR CAMPO con @Autowired (lo que usamos en este tutorial)
@Service
public class RegataService {
    @Autowired
    private BarcoRepository barcoRepository;    // Spring inyecta automáticamente
}

// 2. POR CONSTRUCTOR (recomendado en código profesional)
@Service
public class RegataService {
    private final BarcoRepository barcoRepository;

    public RegataService(BarcoRepository barcoRepository) {
        this.barcoRepository = barcoRepository;
        // Spring detecta el constructor y lo usa para inyectar
    }
}

// 3. POR SETTER (poco común)
@Service
public class RegataService {
    private BarcoRepository barcoRepository;

    @Autowired
    public void setBarcoRepository(BarcoRepository repo) {
        this.barcoRepository = repo;
    }
}
```

> 🏆 **Buena práctica:** La inyección por **constructor** es la preferida
> porque hace que la dependencia sea **obligatoria** y el campo pueda ser `final`.
> En este tutorial usamos `@Autowired` en campos por simplicidad didáctica,
> pero en proyectos profesionales verás más la inyección por constructor.

### 10.10.4. Tabla evolutiva: del SQL directo a Spring

Ahora que conoces tanto Hibernate puro (Capítulos 2-9) como Spring Boot,
puedes ver la **evolución completa** del acceso a datos en nuestro proyecto:

```
  NIVEL 0 — SQL directo (Cap. 1, lo que evitamos)
  ────────────────────────────────────────────────
  Connection conn = DriverManager.getConnection(...);
  PreparedStatement stmt = conn.prepareStatement("INSERT ...");
  stmt.executeUpdate();
  ❌ Código SQL mezclado con lógica de negocio

  NIVEL 1 — DAO concreto (Cap. 2-7, Hibernate puro)
  ────────────────────────────────────────────────
  BarcoDAO dao = new BarcoDAOImpl();   // ← new concreto
  Barco barco = dao.findById(1L);
  ⚠️ Separación lograda, pero acoplado a la implementación

  NIVEL 2 — DI manual (Cap. 2, concepto teórico)
  ────────────────────────────────────────────────
  BarcoDAO dao = new BarcoDAOImpl();
  RegataService servicio = new RegataService(dao);  // ← inyección
  servicio.inscribirBarco(1L, 2L);
  ✅ Desacoplado, pero tú creas y conectas todo en main()

  NIVEL 3 — DI automática con Spring (Cap. 10-18, donde estamos)
  ────────────────────────────────────────────────
  @Service
  public class RegataService {
      @Autowired
      private BarcoRepository barcoRepository;   // ← Spring lo inyecta
  }
  ✅ Desacoplado + Spring controla todo (IoC)
```

| Nivel | ¿Quién crea los objetos? | ¿Acoplamiento? | Capítulos |
|-------|--------------------------|-----------------|-----------|
| 0 | Tu código + SQL manual | ❌ Total | 1 |
| 1 | Tu código con `new` | ⚠️ A la implementación | 2-7 |
| 2 | Tu código `main()` | ✅ Solo a la interfaz | 2 |
| 3 | Spring (Contenedor IoC) | ✅ Mínimo | 10-18 |

Estamos ahora en el **Nivel 3**. Spring crea los beans, inyecta las dependencias,
y gestiona el ciclo de vida de todos los objetos. Tú solo escribes la lógica de negocio.

---

## 10.11. Ejecutar la aplicación

### Paso 1: Arrancar MySQL con Docker

Si seguiste el Capítulo 3, ya tienes el contenedor MySQL arrancado.
Si no, ejecútalo desde la terminal o desde IntelliJ:

```bash
docker-compose up -d
```

### Paso 2: Ejecutar desde IntelliJ

**Opción A — Desde el IDE (recomendado):**
1. Abre `MarinaApplication.java`
2. Haz clic en el botón **▶ verde** junto al método `main()`
3. Selecciona `Run 'MarinaApplication'`

**Opción B — Desde terminal:**
```bash
cd proyecto-spring-boot
mvn spring-boot:run
```

### Paso 3: Verificar en la consola
```
═══════════════════════════════════════════
  🚢 Gestión Marítima — Spring Boot
  API REST:    http://localhost:8080/api/barcos
  Swagger UI:  http://localhost:8080/swagger-ui.html
  Vista Web:   http://localhost:8080/barcos
═══════════════════════════════════════════
```

### Paso 4: Probar en el navegador
- **API REST**: `http://localhost:8080/api/barcos` → JSON con los barcos
- **Swagger**: `http://localhost:8080/swagger-ui.html` → Documentación interactiva
- **Web**: `http://localhost:8080/barcos` → Listado visual con tabla HTML

---

## 10.12. Scopes de beans: Singleton vs Prototype

Cuando Spring crea un bean (un objeto gestionado), ¿crea **uno solo** o **uno nuevo cada vez** que se pide? Eso lo determina el **scope**.

### Singleton (por defecto)

```java
@Service   // ← Scope por defecto: SINGLETON
public class BarcoService {
    // ...
}
```

```
Inyectar BarcoService en BarcoController    → ┐
Inyectar BarcoService en DataLoader         → ├─ MISMO objeto
Inyectar BarcoService en BarcoWebController → ┘
```

Spring crea **UNA sola instancia** de `BarcoService` y la reutiliza en todas las inyecciones. Es el comportamiento por defecto y el más eficiente.

### Prototype

```java
@Service
@Scope("prototype")   // ← Un NUEVO objeto cada vez que se inyecta
public class InformeService {
    // ...
}
```

```
Inyectar InformeService en ControllerA → Objeto #1 (nuevo)
Inyectar InformeService en ControllerB → Objeto #2 (nuevo)
```

### Comparación

| Scope | Instancias | Cuándo usar | Ejemplo |
|-------|-----------|-------------|---------|
| **Singleton** (defecto) | 1 por toda la app | Servicios sin estado | `BarcoService` |
| **Prototype** | 1 nueva por cada inyección | Objetos con estado temporal | Informes, builders |

> 💡 **TIP:** El 99% de los beans en Spring son Singleton. Solo usa Prototype cuando cada consumidor necesite su propia instancia con estado independiente.

> ⚠️ **CUIDADO:** Un bean Singleton que guarda datos en sus campos (estado) puede causar **problemas de concurrencia** si varios usuarios acceden a la vez. Mantén tus servicios **sin estado** (stateless).

---

## 10.13. Spring Profiles: desarrollo vs producción

Los **profiles** permiten tener **diferentes configuraciones** según el entorno. Muy útil para cambiar la BD, URLs o comportamientos entre desarrollo y producción.

### Configuración por perfil

📋 **ARCHIVO DEL PROYECTO** — `src/main/resources/application-dev.properties`

```properties
# Perfil DESARROLLO
spring.datasource.url=jdbc:mysql://localhost:3306/gestion_maritima
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
logging.level.org.hibernate.SQL=DEBUG
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/resources/application-prod.properties`

```properties
# Perfil PRODUCCIÓN
spring.datasource.url=jdbc:mysql://servidor-produccion:3306/gestion_maritima
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
logging.level.org.hibernate.SQL=WARN
```

### Activar un perfil

En `application.properties` (el principal):

```properties
spring.profiles.active=dev   # ← Usa application-dev.properties
```

O al arrancar desde IntelliJ: **Run → Edit Configurations → Active profiles: `dev`**

### Diferencias clave entre perfiles

| Propiedad | `dev` | `prod` |
|-----------|-------|--------|
| `ddl-auto` | `update` (crea/modifica tablas) | `validate` (solo verifica) |
| `show-sql` | `true` (ver consultas SQL) | `false` (no mostrar) |
| Logging | `DEBUG` (máximo detalle) | `WARN` (solo problemas) |

> 🏆 **Buena práctica:** NUNCA uses `ddl-auto=create` o `update` en producción. Podría borrar o modificar datos reales. Usa `validate` y gestiona los cambios de esquema con scripts SQL o herramientas como Flyway.

---

## 📝 Glosario del Capítulo 10

| Término | Definición |
|---------|------------|
| **Spring Framework** | Framework Java para desarrollo empresarial (IoC, AOP, MVC) |
| **Spring Boot** | Extensión de Spring que añade autoconfiguración y servidor embebido |
| **Starter** | Paquete Maven que agrupa dependencias relacionadas y configuradas |
| **Autoconfiguración** | Spring Boot detecta las dependencias y configura todo automáticamente |
| **Bean** | Objeto gestionado por el contenedor de Spring |
| **Contenedor IoC** | Componente de Spring que crea, conecta y gestiona los beans automáticamente |
| **Principio Hollywood** | "No nos llames, ya te llamaremos" — Spring llama a tu código, no al revés |
| **`@Autowired`** | Anotación que indica a Spring que inyecte una dependencia automáticamente |
| **`@SpringBootApplication`** | Meta-anotación que combina Configuration + AutoConfiguration + ComponentScan |
| **`CommandLineRunner`** | Interfaz ejecutada al arrancar la aplicación |
| **`application.properties`** | Fichero de configuración centralizado |
| **`jakarta.persistence`** | Paquete de JPA desde Jakarta EE 10 (antes `javax.persistence`) |
| **Convention over Configuration** | Filosofía: convenciones por defecto, solo configuras las excepciones |
| **HikariCP** | Pool de conexiones de alto rendimiento (incluido en Spring Boot) |
| **Tomcat embebido** | Servidor web incluido en Spring Boot, no necesitas instalación externa |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Entender la diferencia entre Spring Framework y Spring Boot
- Saber qué son los Starters y la autoconfiguración
- Saber crear un proyecto con Spring Initializr
- Entender qué hace `@SpringBootApplication`
- Comprender el **Principio Hollywood** aplicado a Spring
- Saber qué es el **Contenedor IoC** y un **bean**
- Conocer las **tres formas** de inyectar dependencias (`@Autowired`, constructor, setter)
- Ver la **tabla evolutiva** completa: SQL → DAO → DI manual → Spring
- Haber ejecutado `mvn spring-boot:run` y visto la aplicación en el navegador
- Saber que `hibernate.cfg.xml` y `HibernateUtil` YA NO se necesitan

---

> 🏆 **Buena práctica:** Al migrar de Hibernate puro a Spring Boot, no reescribas todo de golpe. Migra por capas: primero las entidades (solo imports), luego los DAOs (a repositorios), y finalmente los controladores.

---

*Capítulo anterior: [← Capítulo 9. Pruebas Unitarias](Cap09_Testing.md)*

*Siguiente capítulo: [Capítulo 11. Repositorios Spring Data JPA →](Cap11_Repositorios.md)*
