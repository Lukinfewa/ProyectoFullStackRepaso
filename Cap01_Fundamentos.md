# Capítulo 1. Introducción a la Persistencia en Java

---

🎯 **Objetivo del capítulo:** Entender qué es la persistencia de datos, por qué necesitamos herramientas como Hibernate, y conocer el proyecto que vamos a construir a lo largo de este tutorial.

---

## 1.1. ¿Qué es la persistencia de datos?

Cuando trabajas con una aplicación, los datos que maneja (usuarios, productos, pedidos...) viven en la **memoria RAM** mientras la aplicación está en ejecución. El problema es que, cuando la aplicación se cierra, esos datos desaparecen.

**Persistir datos** significa guardarlos de forma permanente, normalmente en una **base de datos**, para que sobrevivan al cierre de la aplicación y puedan recuperarse más adelante.

> 📖 **Vocabulario:** **Persistencia** es el mecanismo por el cual los datos de una aplicación se almacenan de forma permanente en un medio de almacenamiento (base de datos, fichero, etc.).

En Java, existen varias formas de persistir datos. Vamos a verlas de menos a más sofisticadas.

---

## 1.2. JDBC: acceso tradicional a bases de datos

**JDBC (Java Database Connectivity)** es la API estándar de Java para conectarse a bases de datos relacionales. Con JDBC, puedes ejecutar consultas SQL directamente desde tu código Java.

Un ejemplo típico de JDBC sería algo así:

```java
// Ejemplo de JDBC puro (NO vamos a usar esto en el proyecto)
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/mibd", "root", "root");
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM barco WHERE id = ?");
stmt.setLong(1, 1L);
ResultSet rs = stmt.executeQuery();

if (rs.next()) {
    String nombre = rs.getString("nombre");
    int eslora = rs.getInt("eslora");
    // ... mapear cada columna a mano
}

rs.close();
stmt.close();
conn.close();
```

### 1.2.1. Limitaciones de JDBC puro

Como puedes ver, con JDBC tienes que:

- Escribir las consultas SQL **a mano**.
- **Mapear** cada columna del resultado a un atributo del objeto Java, uno a uno.
- Gestionar las **conexiones** (abrir, cerrar, controlar errores).
- Repetir todo este código para **cada operación** (insertar, actualizar, borrar, buscar...).

Esto funciona, pero es tedioso, propenso a errores y difícil de mantener cuando la aplicación crece.

> 🔍 **¿Sabías que?** En un proyecto grande con JDBC puro, puedes acabar con miles de líneas de código SQL repetitivo. A esto se le llamaba informalmente "código espagueti de acceso a datos".

---

## 1.3. ¿Qué es ORM (Object-Relational Mapping)?

Aquí es donde entra el concepto de **ORM**. La idea es sencilla: en lugar de escribir SQL y mapear los resultados a objetos manualmente, ¿por qué no dejar que una herramienta lo haga por ti?

> 📖 **Vocabulario:** **ORM (Mapeo Objeto-Relacional)** es una técnica que permite convertir automáticamente entre los objetos de tu aplicación Java y las filas de las tablas de una base de datos relacional.

### 1.3.1. El problema del desajuste objeto-relacional

El mundo de Java y el mundo de las bases de datos son **muy diferentes**:

| Mundo Java (Objetos) | Mundo SQL (Relacional) |
|---------------------|----------------------|
| Clases | Tablas |
| Atributos | Columnas |
| Objetos | Filas |
| Herencia | No existe directamente |
| Referencias entre objetos | Claves foráneas |
| Listas y colecciones | Tablas intermedias |

A esta diferencia entre ambos mundos se le llama el **desajuste objeto-relacional** (*object-relational impedance mismatch*). Un ORM resuelve exactamente este problema.

### 1.3.2. Cómo ORM resuelve este problema

Con un ORM, tú defines tus clases Java normalmente y le dices al ORM **cómo se corresponden** con las tablas de la base de datos (usando anotaciones). El ORM se encarga del resto:

```
   Objeto Java                         Tabla en BD
┌──────────────┐     ORM              ┌──────────────┐
│ Barco        │ ──────────────────▶  │ barco        │
│  id          │  genera SQL          │  id (PK)     │
│  nombre      │  automáticamente     │  nombre      │
│  eslora      │ ◀──────────────────  │  eslora      │
│  tipo        │  mapea resultados    │  tipo        │
└──────────────┘                      └──────────────┘
```

> 💡 **TIP:** Piensa en el ORM como un **traductor simultáneo** entre dos idiomas: el idioma de los objetos Java y el idioma SQL de las bases de datos.

---

## 1.4. JPA: la especificación estándar de Java

Antes de hablar de Hibernate, necesitas entender un concepto clave: **JPA**.

> 📖 **Vocabulario:** **JPA (Java Persistence API)** es una **especificación** (un estándar) de Java que define *cómo* debe funcionar un ORM. JPA no es un programa en sí mismo, es un conjunto de reglas e interfaces.

JPA dice **QUÉ se puede hacer** (por ejemplo: "debe existir una anotación `@Entity` para marcar las clases persistentes"), pero **no dice CÓMO hacerlo internamente**. Eso lo hace la implementación.

> 🔍 **¿Sabías que?** Es como la diferencia entre una **receta** y el **plato cocinado**. JPA es la receta (las instrucciones). Hibernate es el cocinero que sigue la receta y produce el plato.

---

## 1.5. Hibernate: la implementación de JPA

**Hibernate** es la implementación más popular de JPA. Es un **framework ORM** para Java que se encarga de todo el trabajo pesado de la persistencia.

> 📖 **Vocabulario:** **Hibernate** es un framework de mapeo objeto-relacional (ORM) que implementa la especificación JPA. Es el puente entre el mundo de las bases de datos y el mundo de los objetos en Java.

Dicho de forma sencilla: si tienes una aplicación Java y necesitas guardar la información de tus barcos en una base de datos, en lugar de escribir un montón de código SQL, Hibernate te permite trabajar con **objetos Java** directamente. Tú creas un objeto `Barco` con sus atributos (nombre, eslora, tipo...) y le dices a Hibernate que lo guarde. Hibernate se encarga de convertir ese objeto en una fila en la tabla de la base de datos, y viceversa.

Esto no solo te ahorra escribir código SQL, sino que también hace que tu código sea **más limpio, más fácil de entender y de mantener**.

Además, Hibernate gestiona la conexión a la base de datos y ejecuta las consultas necesarias de manera eficiente y segura. También ofrece características avanzadas como **caché**, **lazy loading** y más, que pueden hacer tu vida mucho más fácil cuando tu aplicación crece.

**En resumen:** Hibernate te permite enfocarte en la **lógica de negocio** y en trabajar con objetos Java, dejando de lado la complejidad que implica el manejo directo de la base de datos y el SQL.

```
  +──────────────────────────────────────────────+
  |              Tu aplicación Java              |
  |  (Trabajas con objetos: Barco, Amarre...)    |
  +───────────────────┬──────────────────────────+
                      │
                      ▼
  +──────────────────────────────────────────────+
  |              HIBERNATE (ORM)                 |
  |  Traduce objetos ↔ SQL automáticamente       |
  +───────────────────┬──────────────────────────+
                      │
                      ▼
  +──────────────────────────────────────────────+
  |           Base de Datos (MySQL)              |
  |  Tablas: barco, amarre, regata...            |
  +──────────────────────────────────────────────+
```

### 1.5.1. Ventajas de Hibernate

- **Abstracción de la base de datos:** Hibernate genera las consultas SQL necesarias, lo que te permite enfocarte en la lógica de negocio.
- **Independencia de la base de datos:** Permite cambiar el motor de base de datos (de MySQL a PostgreSQL, por ejemplo) sin modificar el código Java.
- **Caché:** Hibernate proporciona un sistema de caché interno para optimizar las consultas y evitar accesos innecesarios a la base de datos.
- **Lazy loading:** Carga diferida de los datos asociados, lo que mejora el rendimiento al no cargar toda la información innecesariamente.

### 1.5.2. Desventajas de Hibernate

- **Curva de aprendizaje:** Puede ser complejo de aprender al principio, precisamente lo que este tutorial pretende resolver.
- **Rendimiento:** En algunas situaciones muy específicas, las consultas SQL generadas automáticamente pueden no ser tan eficientes como las escritas manualmente. Para esos casos, Hibernate permite usar SQL nativo.

> ⚠️ **CUIDADO:** Que Hibernate genere el SQL no significa que puedas ignorar cómo funciona SQL. Un buen desarrollador con Hibernate **entiende** el SQL que se genera por debajo. Más adelante aprenderemos a activar `show_sql` para ver las consultas.

---

## 1.6. Presentación del proyecto: Sistema de Gestión Marítima y Regatas

A lo largo de todo este tutorial vamos a desarrollar un proyecto real: un **sistema de gestión marítima y regatas**. Este proyecto será nuestro hilo conductor desde el primer capítulo hasta el último.

### 1.6.1. Descripción del dominio

El sistema se basa en tres entidades principales:

**🚢 Barco**
- Representa un barco en el puerto.
- Atributos: nombre, eslora, tipo (velero, motor...), manga, capacidad.
- Cada barco está **asignado a un amarre** específico.
- Un barco puede **participar en varias regatas**.

**⚓ Amarre**
- Representa un puesto de amarre en el puerto.
- Atributos: ubicación, precio, profundidad, longitud, electricidad.
- Un amarre está **asignado a un único barco**.

**🏁 Regata**
- Representa una competición de barcos.
- Atributos: nombre, lugar, fecha, distancia.
- En una regata **participan varios barcos**, y un barco puede participar en **varias regatas**.

### 1.6.2. Relaciones entre las entidades

```
  ┌──────────┐   1:1    ┌──────────┐
  │  Barco   │◄────────►│  Amarre  │
  └────┬─────┘          └──────────┘
       │
       │ N:M
       │
  ┌────▼─────┐
  │  Regata  │
  └──────────┘
```

| Relación | Tipo | Significado |
|----------|------|-------------|
| Barco ↔ Amarre | **Uno a Uno** | Cada barco tiene un amarre, cada amarre tiene un barco |
| Barco ↔ Regata | **Muchos a Muchos** | Un barco participa en varias regatas, una regata tiene varios barcos |

> 🔍 **¿Sabías que?** Las relaciones Muchos a Muchos se implementan en la base de datos con una **tabla intermedia** (en nuestro caso `barco_regata`) que almacena los pares de IDs relacionados. Hibernate se encarga de gestionar esta tabla por ti.

### 1.6.3. Roadmap: qué construiremos capítulo a capítulo

Este proyecto irá creciendo progresivamente. No vamos a escribir todo el código de golpe, sino paso a paso:

| Fase | Capítulos | Qué construimos |
|------|-----------|-----------------|
| **Hibernate puro** | 3-9 | Entidades Java + DAO + consultas + tests |
| **Spring Boot** | 10-12 | Migración a Spring: repositorios + servicios |
| **API REST** | 13-15 | Endpoints HTTP + Postman + Swagger |
| **DTOs** | 16 | Objetos de transferencia de datos |
| **Interfaz web** | 17-18 | Vistas HTML con Thymeleaf |

> 🏆 **Buena práctica:** En proyectos reales, también se sigue este enfoque incremental: primero se construye la capa de datos, después la lógica de negocio, luego la API, y finalmente la interfaz de usuario.

### 1.6.4. Objetivos de aprendizaje del proyecto

Al completar este proyecto, tendrás una comprensión sólida de:

- ✅ Cómo funciona un ORM y cómo **mapear** clases Java a tablas de base de datos.
- ✅ Cómo **configurar** Hibernate y Spring Boot.
- ✅ Cómo **diseñar** la arquitectura de una aplicación por capas (Modelo → DAO → Servicio → Controlador).
- ✅ Cómo escribir **consultas** con HQL, SQL nativo y Criteria API.
- ✅ Cómo **testear** tu código con JUnit 5 y Mockito.
- ✅ Cómo crear una **API REST** y documentarla con Swagger.
- ✅ Cómo construir una **interfaz web** con Thymeleaf.

---

## 📝 Glosario del Capítulo 1

| Término | Definición |
|---------|------------|
| **Persistencia** | Almacenamiento permanente de datos más allá del ciclo de vida de la aplicación |
| **JDBC** | Java Database Connectivity — API estándar de Java para conectarse a bases de datos usando SQL directamente |
| **ORM** | Object-Relational Mapping — técnica que convierte automáticamente entre objetos Java y filas de tablas SQL |
| **JPA** | Java Persistence API — especificación estándar de Java que define cómo debe funcionar un ORM |
| **Hibernate** | Framework ORM que implementa JPA, actuando como puente entre objetos Java y bases de datos |
| **Especificación** | Documento que define reglas e interfaces pero no cómo implementarlas (ej: JPA) |
| **Implementación** | Código concreto que sigue una especificación (ej: Hibernate implementa JPA) |
| **Framework** | Conjunto de herramientas y librerías que proporcionan una estructura para desarrollar aplicaciones |
| **Mapeo** | Correspondencia entre un elemento de un mundo (objeto Java) y otro (fila en una tabla SQL) |
| **Caché** | Almacenamiento temporal en memoria para evitar repetir consultas costosas a la base de datos |
| **Lazy loading** | Carga diferida: los datos asociados a un objeto no se cargan hasta que realmente se necesitan |
| **CRUD** | Acrónimo de Create, Read, Update, Delete — las cuatro operaciones básicas sobre datos |
| **Clave primaria (PK)** | Columna de una tabla que identifica de forma única cada fila |
| **Clave foránea (FK)** | Columna que referencia a la clave primaria de otra tabla, estableciendo una relación |
| **Tabla intermedia** | Tabla usada para implementar relaciones Muchos a Muchos entre dos entidades |

---

✅ **Checkpoint:** Al terminar este capítulo debes entender...
- Qué problema resuelve un ORM frente a JDBC puro.
- La diferencia entre JPA (especificación) y Hibernate (implementación).
- Las ventajas y desventajas de usar Hibernate.
- Qué entidades tiene nuestro proyecto (Barco, Amarre, Regata) y cómo se relacionan.
- Que aún **no hemos escrito código** — eso empieza en el Capítulo 3.

---

*Siguiente capítulo: [Capítulo 2. Patrones de Diseño para el Acceso a Datos →](Cap02_Patrones_DAO.md)*
