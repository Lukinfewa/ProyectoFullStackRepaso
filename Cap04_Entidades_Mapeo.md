# Capítulo 4. Entidades y Mapeo Objeto-Relacional con Anotaciones

---

🎯 **Objetivo del capítulo:** Crear las primeras entidades del proyecto (`Barco` y `Amarre`), aprender a usar las anotaciones de Hibernate/JPA para mapearlas a tablas de la base de datos, y entender cómo Lombok reduce el código repetitivo.

---

## 4.1. Concepto de entidad en Hibernate

Una **entidad** es una clase Java que Hibernate mapea a una tabla en la base de datos. Cada atributo de la clase se corresponde con una columna de la tabla, y cada instancia del objeto se corresponde con una fila.

```
   Clase Java (Entidad)                 Tabla MySQL
┌──────────────────────┐          ┌─────────────────────────┐
│ @Entity              │          │ barco                   │
│ Barco                │  ──────▶ ├─────────────────────────┤
│   id: Long           │          │ id (PK, AUTO_INCREMENT) │
│   nombre: String     │          │ nombre VARCHAR(255)     │
│   tipo: String       │          │ tipo VARCHAR(255)       │
│   eslora: int        │          │ eslora INT              │
│   manga: int         │          │ manga INT               │
│   capacidad: int     │          │ capacidad INT           │
└──────────────────────┘          └─────────────────────────┘
```

> 📖 **Vocabulario:** Una **entidad** (*entity*) es un objeto Java cuyo estado se persiste en una tabla de la base de datos. Para que Hibernate la reconozca como entidad, debe llevar la anotación `@Entity`.

---

## 4.2. Anotaciones básicas de mapeo

Las anotaciones son **marcas** que pones en tu código Java para decirle a Hibernate cómo mapear cada clase y cada atributo. Todas vienen del paquete `javax.persistence` (estándar JPA).

### 4.2.1. `@Entity` — marcar una clase como entidad

```java
@Entity
public class Barco {
    // ...
}
```

Con esta anotación, Hibernate sabe que esta clase debe mapearse a una tabla. Por defecto, el nombre de la tabla será el mismo que el de la clase (en este caso, `barco`).

### 4.2.2. `@Table` — nombre de la tabla

Si quieres que la tabla tenga un nombre diferente al de la clase:

```java
@Entity
@Table(name = "barcos_puerto")
public class Barco {
    // ...
}
```

> 💡 **TIP:** En nuestro proyecto no usaremos `@Table` porque los nombres por defecto nos valen. Pero es bueno saber que existe para proyectos donde la base de datos ya existe y las tablas tienen nombres específicos.

### 4.2.3. `@Id` — clave primaria

```java
@Id
private Long id;
```

Marca el atributo como la **clave primaria** de la tabla. Toda entidad **debe** tener un `@Id`.

> 📖 **Vocabulario:** La **clave primaria** (*primary key*, PK) es la columna que identifica de forma única cada fila de una tabla. No puede haber dos filas con el mismo valor de clave primaria.

### 4.2.4. `@Column` — configuración de columnas

```java
@Column(name = "nombre_barco", unique = true, nullable = false, length = 100)
private String nombre;
```

| Atributo | Significado | Valor por defecto |
|----------|-------------|-------------------|
| `name` | Nombre de la columna en la BD | Nombre del atributo Java |
| `unique` | Si el valor debe ser único | `false` |
| `nullable` | Si permite valores `NULL` | `true` |
| `length` | Longitud máxima (para Strings) | `255` |

> 💡 **TIP:** `@Column` es **opcional**. Si no la pones, Hibernate crea la columna con el mismo nombre que el atributo y los valores por defecto. Solo la necesitas cuando quieres personalizar algo.

---

## 4.3. Generación automática de ID con `@GeneratedValue`

No quieres tener que asignar manualmente un ID cada vez que creas un barco. Para eso existe `@GeneratedValue`, que le dice a Hibernate (y a la base de datos) que **genere el ID automáticamente**:

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

### Estrategias de generación

| Estrategia | Cómo funciona | Cuándo usarla |
|------------|--------------|---------------|
| `IDENTITY` | La BD genera el ID (AUTO_INCREMENT en MySQL) | ✅ **MySQL** (la que usaremos) |
| `SEQUENCE` | Usa una secuencia de la BD | PostgreSQL, Oracle |
| `TABLE` | Usa una tabla auxiliar para generar IDs | Portabilidad máxima (poco eficiente) |
| `AUTO` | Hibernate elige automáticamente | Cuando no te importa la estrategia |

> 🏆 **Buena práctica:** Para MySQL, usa siempre `GenerationType.IDENTITY`. Es la forma más natural y eficiente de generar IDs auto-incrementales en este motor.

> 🔍 **¿Sabías que?** En la base de datos, esto se traduce en una columna `id INT AUTO_INCREMENT PRIMARY KEY`. MySQL se encarga de asignar el siguiente número disponible cada vez que insertas una fila.

---

## 4.4. Anotaciones adicionales útiles

Estas anotaciones no las usaremos todas en nuestro proyecto, pero es importante que las conozcas:

### 4.4.1. `@Temporal` — persistir fechas

Cuando tienes un atributo de tipo `java.util.Date`, necesitas indicar qué parte de la fecha quieres guardar:

```java
@Temporal(TemporalType.DATE)       // Solo fecha: 2024-03-15
private Date fechaRegistro;

@Temporal(TemporalType.TIME)       // Solo hora: 14:30:00
private Date horaEntrada;

@Temporal(TemporalType.TIMESTAMP)  // Fecha y hora: 2024-03-15 14:30:00
private Date fechaCreacion;
```

> 💡 **TIP:** Usaremos `@Temporal(TemporalType.DATE)` en la entidad `Regata` para guardar la fecha de la competición.

### 4.4.2. `@Enumerated` — persistir enums

Si tienes un campo con valores fijos (como el tipo de barco), puedes usar un enum:

```java
public enum TipoBarco {
    VELERO, MOTORA, CATAMARAN
}

@Enumerated(EnumType.STRING)  // Guarda el NOMBRE del enum ("VELERO")
private TipoBarco tipo;
```

| Estrategia | Guarda en BD | Ejemplo |
|-----------|-------------|---------|
| `EnumType.STRING` | El nombre del enum | `"VELERO"` |
| `EnumType.ORDINAL` | La posición numérica | `0` |

> ⚠️ **CUIDADO:** Evita `EnumType.ORDINAL`. Si alguien reordena los valores del enum, todos los datos de la BD se corrompen. Usa siempre `EnumType.STRING`.

### 4.4.3. `@Transient` — excluir atributos

Si un atributo **no debe persistirse** en la base de datos (es calculado, temporal, etc.):

```java
@Transient
private double precioConIVA;  // Se calcula, no se guarda en BD
```

> 📖 **Vocabulario:** **Transient** significa transitorio, temporal. Un atributo `@Transient` existe solo en memoria, Hibernate lo ignora por completo.

---

## 4.5. Lombok para reducir código repetitivo

Sin Lombok, una entidad sencilla necesita **mucho código repetitivo**:

```java
// ❌ Sin Lombok: unas 60+ líneas para una clase con 6 atributos
public class Barco {
    private Long id;
    private String nombre;
    // ... más atributos

    public Barco() {}              // Constructor vacío

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    // ... getters y setters para CADA atributo

    @Override
    public boolean equals(Object o) { /* ... */ }
    @Override
    public int hashCode() { /* ... */ }
    @Override
    public String toString() { /* ... */ }
}
```

Con Lombok, todo eso se reduce a **tres anotaciones**:

### 4.5.1. `@Data`

Genera automáticamente: todos los **getters**, todos los **setters**, `equals()`, `hashCode()` y `toString()`.

### 4.5.2. `@NoArgsConstructor`

Genera un **constructor vacío** (sin argumentos). Hibernate lo necesita internamente para crear instancias de las entidades.

> ⚠️ **CUIDADO:** Si una entidad no tiene constructor sin argumentos, Hibernate lanza un error al intentar crear objetos. `@NoArgsConstructor` de Lombok o un constructor vacío manual son **obligatorios**.

### 4.5.3. `@AllArgsConstructor`

Genera un constructor con **todos los atributos** como parámetros. Útil para crear objetos rápidamente en tests.

### 4.5.4. `@ToString.Exclude`

Cuando tengas entidades con **relaciones bidireccionales** (ej: Barco conoce a Amarre y Amarre conoce a Barco), el `toString()` generado por `@Data` puede entrar en un **bucle infinito**:

```
Barco.toString() → imprime Amarre → Amarre.toString() → imprime Barco → ... StackOverflowError!
```

La solución es excluir la relación del `toString()`:

```java
@ToString.Exclude
private Amarre amarre;
```

> ⚠️ **CUIDADO:** Este es un error MUY común con Lombok y relaciones bidireccionales. Si ves un `StackOverflowError`, revisa los `toString()`.

---

## 4.6. Práctica: Crear la entidad `Barco.java`

Crea la clase `Barco.java` en el paquete `com.mycompany.hibernate.modelo`:

### Paso 1: La clase sin anotaciones

Primero, imagina la clase "pura" con sus atributos:

```java
public class Barco {
    private Long id;
    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;
}
```

### Paso 2: Añadir las anotaciones JPA

Ahora le decimos a Hibernate que esta clase es una entidad y cómo mapear cada campo:

```java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity              // ← Esto es una entidad (mapea a tabla "barco")
public class Barco {

    @Id              // ← Clave primaria
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // ← Auto-incremento
    private Long id;

    private String nombre;   // ← Columna "nombre" (automático)
    private String tipo;     // ← Columna "tipo" (automático)
    private int eslora;      // ← Columna "eslora" (automático)
    private int manga;       // ← Columna "manga" (automático)
    private int capacidad;   // ← Columna "capacidad" (automático)
}
```

### Paso 3: Añadir Lombok

Finalmente, añadimos Lombok para no tener que escribir getters, setters ni constructores:

```java
package com.mycompany.hibernate.modelo;

import lombok.Data;
import lombok.NoArgsConstructor;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity                // Entidad JPA → tabla "barco"
@Data                  // Lombok: getters, setters, equals, hashCode, toString
@NoArgsConstructor     // Lombok: constructor vacío (requerido por Hibernate)
public class Barco {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;
}
```

**¡Listo!** Con estas pocas líneas, Hibernate creará automáticamente la tabla `barco` en MySQL con todas sus columnas.

> 🔍 **¿Sabías que?** Aunque no ves los getters y setters, sí existen. Lombok los genera en tiempo de compilación. Puedes usarlos normalmente: `barco.getNombre()`, `barco.setEslora(12)`, etc.

---

## 4.7. Práctica: Crear la entidad `Amarre.java`

Crea la clase `Amarre.java` en el mismo paquete:

```java
package com.mycompany.hibernate.modelo;

import lombok.Data;
import lombok.NoArgsConstructor;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@Data
@NoArgsConstructor
public class Amarre {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String ubicacion;
    private double precio;
    private int profundidad;
    private int longitud;
    private boolean electricidad;
}
```

> 💡 **TIP:** Fíjate en que todavía NO hemos añadido ninguna **relación** entre `Barco` y `Amarre`. Eso lo haremos en el Capítulo 5. Por ahora, son dos entidades independientes.

---

## 4.8. Registrar las entidades en `hibernate.cfg.xml`

Para que Hibernate conozca las entidades, debemos registrarlas. Abre `hibernate.cfg.xml` y descomenta (o añade) las líneas de mapping:

```xml
<!-- Entidades mapeadas -->
<mapping class="com.mycompany.hibernate.modelo.Barco" />
<mapping class="com.mycompany.hibernate.modelo.Amarre" />
```

> ⚠️ **CUIDADO:** Si te olvidas de registrar una entidad aquí, Hibernate no la reconocerá y no creará su tabla. Es un error muy común que genera confusión.

---

## 4.9. Verificar que todo funciona

Vamos a crear una clase `Main.java` muy sencilla para verificar que Hibernate se conecta a MySQL y crea las tablas:

```java
package com.mycompany.hibernate;

import com.mycompany.hibernate.util.HibernateUtil;
import org.hibernate.SessionFactory;

public class Main {
    public static void main(String[] args) {

        System.out.println("Inicializando Hibernate...");

        // Al llamar a getSessionFactory(), Hibernate:
        // 1. Lee hibernate.cfg.xml
        // 2. Se conecta a MySQL
        // 3. Crea las tablas que falten (porque hbm2ddl.auto = update)
        SessionFactory sf = HibernateUtil.getSessionFactory();

        System.out.println("✅ Hibernate inicializado correctamente.");
        System.out.println("✅ Las tablas 'barco' y 'amarre' deben existir en MySQL.");

        sf.close();
    }
}
```

**Ejecuta esta clase.** Si todo está bien configurado, verás en la consola las sentencias SQL de creación de tablas:

```
Hibernate: create table barco (id bigint not null auto_increment, capacidad integer not null, eslora integer not null, manga integer not null, nombre varchar(255), tipo varchar(255), primary key (id)) engine=InnoDB
Hibernate: create table amarre (id bigint not null auto_increment, electricidad boolean not null, longitud integer not null, precio double precision not null, profundidad integer not null, ubicacion varchar(255), primary key (id)) engine=InnoDB
✅ Hibernate inicializado correctamente.
✅ Las tablas 'barco' y 'amarre' deben existir en MySQL.
```

> 🔍 **¿Sabías que?** Puedes verificar que las tablas se han creado abriendo MySQL y ejecutando: `USE gestion_maritima; SHOW TABLES;`. Deberías ver las tablas `barco` y `amarre`.

---

## 4.10. Resumen visual del mapeo

```
 Código Java                          Base de datos MySQL
─────────────────                    ──────────────────────

 @Entity                             CREATE TABLE barco (
 class Barco {
   @Id                                 id BIGINT AUTO_INCREMENT PRIMARY KEY,
   @GeneratedValue(IDENTITY)

   String nombre;                      nombre VARCHAR(255),
   String tipo;                        tipo VARCHAR(255),
   int eslora;                         eslora INT,
   int manga;                          manga INT,
   int capacidad;                      capacidad INT
 }                                   ) ENGINE=InnoDB;
```

---

## 📝 Glosario del Capítulo 4

| Término | Definición |
|---------|------------|
| **Entidad** | Clase Java mapeada a una tabla de la BD mediante la anotación `@Entity` |
| **`@Entity`** | Anotación JPA que marca una clase como entidad persistente |
| **`@Table`** | Anotación opcional para especificar el nombre de la tabla en la BD |
| **`@Id`** | Anotación que marca un atributo como clave primaria de la entidad |
| **`@GeneratedValue`** | Anotación que configura la generación automática del ID |
| **`@Column`** | Anotación opcional para personalizar el mapeo de una columna (nombre, unique, nullable) |
| **`@Temporal`** | Anotación para indicar el tipo de dato temporal (DATE, TIME, TIMESTAMP) |
| **`@Enumerated`** | Anotación para persistir enums como STRING u ORDINAL |
| **`@Transient`** | Anotación para excluir un atributo de la persistencia |
| **`@Data`** | Anotación de Lombok que genera getters, setters, equals, hashCode y toString |
| **`@NoArgsConstructor`** | Anotación de Lombok que genera un constructor sin argumentos |
| **AUTO_INCREMENT** | Propiedad de MySQL que genera valores numéricos consecutivos automáticamente |
| **InnoDB** | Motor de almacenamiento de MySQL que soporta transacciones y claves foráneas |
| **Mapping** | Registro de una entidad en `hibernate.cfg.xml` para que Hibernate la reconozca |

---

✅ **Checkpoint:** Al terminar este capítulo debes tener...
- Las clases `Barco.java` y `Amarre.java` creadas con anotaciones JPA y Lombok.
- Las entidades registradas en `hibernate.cfg.xml`.
- La clase `Main.java` ejecutada correctamente.
- Las tablas `barco` y `amarre` creadas automáticamente en MySQL.
- Entender qué hace cada anotación: `@Entity`, `@Id`, `@GeneratedValue`, `@Data`, `@NoArgsConstructor`.

---

*Capítulo anterior: [← Capítulo 3. Configuración del Entorno de Desarrollo](Cap03_Configuracion.md)*

*Siguiente capítulo: [Capítulo 5. Relaciones entre Entidades →](Cap05_Relaciones.md)*
