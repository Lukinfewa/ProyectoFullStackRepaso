# Capítulo 5. Relaciones entre Entidades

---

🎯 **Objetivo del capítulo:** Aprender a definir relaciones entre entidades (Uno a Uno, Uno a Muchos, Muchos a Muchos) usando anotaciones JPA, entender el concepto de entidad propietaria, y completar el modelo del proyecto añadiendo `Regata` y conectando todas las entidades.

---

## 5.1. Concepto de entidad propietaria de la relación

Cuando dos entidades se conocen mutuamente (relación **bidireccional**), Hibernate necesita saber **quién manda** en la relación, es decir, quién almacena la clave foránea en la base de datos.

> 📖 **Vocabulario:** La **entidad propietaria** (*owning side*) es la que contiene la columna de **clave foránea** en su tabla. La otra entidad es el **lado inverso** (*inverse side*) y usa `mappedBy` para señalar al propietario.

### 5.1.1. ¿Quién tiene la clave foránea?

En la base de datos, una relación se implementa con una **clave foránea** (FK): una columna en una tabla que referencia al ID de otra tabla.

```
  Tabla barco                    Tabla amarre
┌──────────────┐              ┌──────────────────────┐
│ id (PK)      │◄─────────────│ id (PK)              │
│ nombre       │              │ ubicacion            │
│ tipo         │              │ precio               │
│ eslora       │              │ barco_id (FK) ───────┤ ← La FK está en amarre
└──────────────┘              └──────────────────────┘
```

En este ejemplo, la tabla `amarre` tiene la columna `barco_id` que apunta a `barco`. Por tanto, **`Amarre` es la entidad propietaria** de la relación.

### 5.1.2. El atributo `mappedBy`

El atributo `mappedBy` se coloca **en el lado NO propietario** para decir: "la relación ya está definida en la otra entidad, en el atributo X".

```java
// En Barco (NO propietario): "la relación está mapeada por el atributo 'barco' de Amarre"
@OneToOne(mappedBy = "barco")
private Amarre amarre;

// En Amarre (propietario): tiene la FK
@OneToOne
private Barco barco;
```

> ⚠️ **CUIDADO:** Si pones `mappedBy` en ambos lados o en ninguno, Hibernate creará tablas de relación innecesarias o duplicadas. Solo el **lado inverso** lleva `mappedBy`.

> 💡 **TIP:** Regla rápida para recordar: **el que tiene `mappedBy` NO tiene la FK**. El que NO tiene `mappedBy` SÍ tiene la FK.

---

## 5.2. Relación Uno a Uno (`@OneToOne`)

Una relación Uno a Uno significa que **cada instancia de una entidad se asocia con exactamente una instancia de otra entidad**. En nuestro proyecto: cada barco tiene un amarre y cada amarre tiene un barco.

### 5.2.1. Unidireccional: solo una entidad conoce la relación

En una relación unidireccional, solo una entidad tiene la referencia. La otra no sabe nada.

```java
// Amarre conoce a Barco, pero Barco NO conoce a Amarre
@Entity
@Data
@NoArgsConstructor
public class Amarre {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String ubicacion;
    private double precio;

    @OneToOne       // ← Amarre CONOCE a su Barco
    private Barco barco;
}
```

Hibernate crea una columna `barco_id` en la tabla `amarre`.

### 5.2.2. Bidireccional: ambas entidades se conocen

En la bidireccional, **ambas entidades pueden navegar a la otra**. Es lo más habitual y lo que usaremos en nuestro proyecto:

```java
// BARCO (lado inverso — NO tiene la FK)
@Entity
@Data
@NoArgsConstructor
public class Barco {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;

    @OneToOne(mappedBy = "barco", cascade = CascadeType.ALL)
    @ToString.Exclude    // ← Evitar StackOverflow en toString()
    private Amarre amarre;
}
```

```java
// AMARRE (propietario — TIENE la FK barco_id)
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

    @OneToOne            // ← Propietario de la relación
    @ToString.Exclude
    private Barco barco;
}
```

> 🏆 **Buena práctica:** En relaciones bidireccionales, usa siempre `@ToString.Exclude` de Lombok en al menos un lado para evitar el `StackOverflowError` por referencia circular en el `toString()`.

---

## 5.3. Relación Uno a Muchos / Muchos a Uno

Una relación Uno a Muchos significa que **una instancia de una entidad se asocia con múltiples instancias de otra**. Por ejemplo: un departamento tiene muchos empleados.

### 5.3.1. Unidireccional de Uno a Muchos

```java
// Departamento tiene una lista de empleados
@OneToMany
private List<Empleado> empleados;
```

### 5.3.2. Bidireccional de Uno a Muchos

```java
// DEPARTAMENTO (lado "uno", inverso)
@OneToMany(mappedBy = "departamento")
private List<Empleado> empleados;

// EMPLEADO (lado "muchos", propietario — tiene la FK)
@ManyToOne
private Departamento departamento;
```

> 💡 **TIP:** En las relaciones Uno a Muchos, el lado "muchos" (`@ManyToOne`) es siempre el **propietario**, porque es el que tiene la clave foránea en su tabla.

> 🔍 **¿Sabías que?** No usamos esta relación directamente en nuestro proyecto de barcos, pero es la relación más común en aplicaciones reales (clientes-pedidos, categorías-productos, departamentos-empleados...).

---

## 5.4. Relación Muchos a Muchos (`@ManyToMany`)

Una relación Muchos a Muchos significa que **múltiples instancias de ambas entidades pueden asociarse entre sí**. En nuestro proyecto: un barco puede participar en varias regatas, y una regata puede tener varios barcos.

### 5.4.1. Tabla intermedia con `@JoinTable`

En la base de datos, las relaciones Muchos a Muchos se implementan con una **tabla intermedia** que contiene los IDs de ambas entidades:

```
  Tabla barco              Tabla barco_regata           Tabla regata
┌──────────────┐         ┌────────────────────┐       ┌──────────────┐
│ id (PK)      │◄────────│ barco_id (FK)      │       │ id (PK)      │
│ nombre       │         │ regata_id (FK) ────│──────▶│ nombre       │
└──────────────┘         └────────────────────┘       │ fecha        │
                                                       └──────────────┘
```

La anotación `@JoinTable` configura esta tabla intermedia:

```java
@ManyToMany
@JoinTable(
    name = "barco_regata",                              // Nombre de la tabla intermedia
    joinColumns = @JoinColumn(name = "barco_id"),       // FK hacia esta entidad (Barco)
    inverseJoinColumns = @JoinColumn(name = "regata_id") // FK hacia la otra (Regata)
)
private List<Regata> regatas;
```

> 📖 **Vocabulario:** La **tabla intermedia** (*join table*) es una tabla auxiliar que solo contiene las claves foráneas de las dos entidades relacionadas. No tiene entidad Java propia; Hibernate la gestiona automáticamente.

### 5.4.2. Bidireccional (lo que usaremos en el proyecto)

```java
// BARCO (propietario — define la @JoinTable)
@ManyToMany
@JoinTable(
    name = "barco_regata",
    joinColumns = @JoinColumn(name = "barco_id"),
    inverseJoinColumns = @JoinColumn(name = "regata_id")
)
@ToString.Exclude
private List<Regata> regatas;

// REGATA (lado inverso — usa mappedBy)
@ManyToMany(mappedBy = "regatas")
@ToString.Exclude
private List<Barco> barcos;
```

---

## 5.5. Operaciones en cascada (`cascade`)

Cuando guardas un `Barco` que tiene un `Amarre` asociado, ¿Hibernate debe guardar también el `Amarre` automáticamente? Eso depende del **cascade**.

```java
// CON cascade: al guardar el Barco, también se guarda el Amarre automáticamente
@OneToOne(mappedBy = "barco", cascade = CascadeType.ALL)
private Amarre amarre;

// SIN cascade: tendrías que guardar Barco y Amarre por separado
@OneToOne(mappedBy = "barco")
private Amarre amarre;
```

### Tipos de cascade

| Tipo | Propaga la operación... | Ejemplo |
|------|------------------------|---------|
| `CascadeType.PERSIST` | `save()` / `persist()` | Guardar barco → guarda amarre |
| `CascadeType.MERGE` | `merge()` | Actualizar barco → actualiza amarre |
| `CascadeType.REMOVE` | `delete()` | Borrar barco → borra amarre |
| `CascadeType.REFRESH` | `refresh()` | Recargar barco → recarga amarre |
| `CascadeType.DETACH` | `detach()` | Desconectar barco → desconecta amarre |
| `CascadeType.ALL` | **Todas las anteriores** | ✅ Lo más habitual en relaciones fuertes |

> 🏆 **Buena práctica:** Usa `CascadeType.ALL` cuando las entidades tienen una relación de **composición** fuerte (el amarre no tiene sentido sin su barco). No lo uses en relaciones débiles donde las entidades son independientes.

> ⚠️ **CUIDADO:** Con `CascadeType.REMOVE`, si borras un barco, se borra su amarre automáticamente. Asegúrate de que eso es lo que quieres antes de activarlo.

---

## 5.6. Práctica: Crear `Regata.java` y anotar todas las relaciones

### Paso 1: Crear la entidad `Regata.java`

```java
package com.mycompany.hibernate.modelo;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import javax.persistence.*;
import java.util.Date;
import java.util.List;

@Entity
@Data
@NoArgsConstructor
public class Regata {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;
    private String lugar;

    @Temporal(TemporalType.DATE)    // Solo fecha (sin hora)
    private Date fecha;

    private int distancia;

    @ManyToMany(mappedBy = "regatas")  // Lado inverso de la relación
    @ToString.Exclude
    private List<Barco> barcos;
}
```

### Paso 2: Actualizar `Barco.java` con las relaciones

```java
package com.mycompany.hibernate.modelo;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import javax.persistence.*;
import java.util.List;

@Entity
@Data
@NoArgsConstructor
public class Barco {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;

    // Relación 1:1 con Amarre (lado inverso)
    @OneToOne(mappedBy = "barco", cascade = CascadeType.ALL)
    @ToString.Exclude
    private Amarre amarre;

    // Relación N:M con Regata (propietario — define la tabla intermedia)
    @ManyToMany
    @JoinTable(
        name = "barco_regata",
        joinColumns = @JoinColumn(name = "barco_id"),
        inverseJoinColumns = @JoinColumn(name = "regata_id")
    )
    @ToString.Exclude
    private List<Regata> regatas;
}
```

### Paso 3: Actualizar `Amarre.java` con la relación

```java
package com.mycompany.hibernate.modelo;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.ToString;
import javax.persistence.*;

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

    // Relación 1:1 con Barco (propietario — tiene la FK)
    @OneToOne
    @ToString.Exclude
    private Barco barco;
}
```

### Paso 4: Registrar `Regata` en `hibernate.cfg.xml`

```xml
<mapping class="com.mycompany.hibernate.modelo.Barco" />
<mapping class="com.mycompany.hibernate.modelo.Amarre" />
<mapping class="com.mycompany.hibernate.modelo.Regata" />
```

### Paso 5: Verificar las tablas

Al ejecutar `Main.java`, Hibernate debería crear **tres tablas** más la **tabla intermedia**:

```
gestion_maritima
├── barco            ← Entidad Barco
├── amarre           ← Entidad Amarre (con columna barco_id FK)
├── regata           ← Entidad Regata
└── barco_regata     ← Tabla intermedia (barco_id, regata_id)
```

---

## 5.7. Resumen visual de las relaciones del proyecto

```
  ┌──────────────────────┐
  │      BARCO           │
  │  id (PK)             │
  │  nombre              │
  │  tipo                │         1:1           ┌──────────────────────┐
  │  eslora              │◄──────────────────────│      AMARRE          │
  │  manga               │    mappedBy="barco"   │  id (PK)             │
  │  capacidad           │                       │  ubicacion           │
  │                      │                       │  precio              │
  │  amarre ──────────── │                       │  barco_id (FK) ──────│
  │  regatas ─────┐      │                       └──────────────────────┘
  └───────────────│──────┘
                  │
                  │ N:M (tabla intermedia: barco_regata)
                  │
  ┌───────────────▼──────┐
  │      REGATA          │
  │  id (PK)             │
  │  nombre              │
  │  lugar               │
  │  fecha               │
  │  distancia           │
  │  barcos ─────────────│  mappedBy="regatas"
  └──────────────────────┘
```

---

## 5.8. Código completo de las entidades (versión Spring Boot)

> ⚠️ **IMPORTANTE:** La sección 5.6 muestra las entidades con los imports de **Hibernate puro**
> (`javax.persistence`). A partir del **Capítulo 10** usamos **Spring Boot** con `jakarta.persistence`.
> A continuación están las versiones completas y finales que usará el proyecto Spring Boot.

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/model/Barco.java`

```java
package com.example.marina.model;

import jakarta.persistence.*;
import lombok.*;
import java.util.ArrayList;
import java.util.List;

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

    @Column(nullable = false)
    private int eslora;

    @Column(nullable = false)
    private int manga;

    @Column(nullable = false)
    private int capacidad;

    @OneToOne(mappedBy = "barco", cascade = CascadeType.ALL, orphanRemoval = true)
    @ToString.Exclude
    private Amarre amarre;

    @ManyToMany(cascade = { CascadeType.PERSIST, CascadeType.MERGE })
    @JoinTable(name = "barco_regata",
            joinColumns = @JoinColumn(name = "barco_id"),
            inverseJoinColumns = @JoinColumn(name = "regata_id"))
    @ToString.Exclude
    private List<Regata> regatas = new ArrayList<>();

    public Barco(String nombre, String tipo, int eslora, int manga, int capacidad) {
        this.nombre = nombre;
        this.tipo = tipo;
        this.eslora = eslora;
        this.manga = manga;
        this.capacidad = capacidad;
    }
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/model/Amarre.java`

```java
package com.example.marina.model;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Amarre {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 20)
    private String ubicacion;

    @Column(nullable = false)
    private double precio;

    @Column(nullable = false)
    private int profundidad;

    @Column(nullable = false)
    private int longitud;

    @Column(nullable = false)
    private boolean electricidad;

    @OneToOne
    @JoinColumn(name = "barco_id", unique = true)
    @ToString.Exclude
    private Barco barco;

    public Amarre(String ubicacion, double precio, int profundidad,
            int longitud, boolean electricidad) {
        this.ubicacion = ubicacion;
        this.precio = precio;
        this.profundidad = profundidad;
        this.longitud = longitud;
        this.electricidad = electricidad;
    }
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/model/Regata.java`

```java
package com.example.marina.model;

import jakarta.persistence.*;
import lombok.*;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Regata {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(nullable = false, length = 100)
    private String lugar;

    @Temporal(TemporalType.DATE)
    @Column(nullable = false)
    private Date fecha;

    @Column(nullable = false)
    private int distancia;

    @ManyToMany(mappedBy = "regatas")
    @ToString.Exclude
    private List<Barco> barcos = new ArrayList<>();

    public Regata(String nombre, String lugar, Date fecha, int distancia) {
        this.nombre = nombre;
        this.lugar = lugar;
        this.fecha = fecha;
        this.distancia = distancia;
    }
}
```

> 💡 **TIP:** Las diferencias principales entre Hibernate puro (Cap. 5.6) y Spring Boot son:
> - **Imports**: `javax.persistence` → `jakarta.persistence`
> - **`@Column`**: añadimos `nullable = false` y `length` para validación a nivel de BD
> - **`@JoinColumn`**: explicitamos el nombre de la FK (`barco_id`)
> - **`orphanRemoval = true`**: si desvinculamos un amarre, se borra al guardar el barco

---

## 📝 Glosario del Capítulo 5

| Término | Definición |
|---------|------------|
| **Relación unidireccional** | Solo una entidad conoce a la otra |
| **Relación bidireccional** | Ambas entidades se conocen mutuamente |
| **Entidad propietaria** | La entidad que contiene la clave foránea en su tabla |
| **Lado inverso** | La entidad que usa `mappedBy`; no tiene la FK |
| **`mappedBy`** | Atributo que indica que la relación ya está definida en la otra entidad |
| **`@OneToOne`** | Relación uno a uno: cada instancia se asocia con exactamente una de la otra |
| **`@OneToMany`** | Relación uno a muchos: una instancia se asocia con múltiples de la otra |
| **`@ManyToOne`** | Relación muchos a uno: varias instancias se asocian con una de la otra |
| **`@ManyToMany`** | Relación muchos a muchos: múltiples instancias se asocian entre sí |
| **`@JoinTable`** | Define la tabla intermedia para relaciones ManyToMany |
| **`@JoinColumn`** | Define el nombre de la columna de clave foránea |
| **Tabla intermedia** | Tabla auxiliar que almacena los pares de IDs de una relación N:M |
| **Cascade** | Propagación automática de operaciones de una entidad a sus entidades relacionadas |
| **Clave foránea (FK)** | Columna que referencia a la clave primaria de otra tabla |

---

✅ **Checkpoint:** Al terminar este capítulo debes tener...
- Las tres entidades (`Barco`, `Amarre`, `Regata`) con todas sus relaciones anotadas.
- Entender la diferencia entre entidad propietaria y lado inverso.
- Saber cuándo usar `mappedBy` (siempre en el lado inverso).
- Saber cuándo usar `@JoinTable` (en relaciones ManyToMany).
- Las cuatro tablas (`barco`, `amarre`, `regata`, `barco_regata`) creadas en MySQL.

---

*Capítulo anterior: [← Capítulo 4. Entidades y Mapeo](Cap04_Entidades_Mapeo.md)*

*Siguiente capítulo: [Capítulo 6. Estados y Ciclo de Vida de las Entidades →](Cap06_Estados.md)*
