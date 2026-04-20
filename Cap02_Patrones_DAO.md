# Capítulo 2. Patrones de Diseño para el Acceso a Datos

---

🎯 **Objetivo del capítulo:** Entender qué es el patrón DAO, por qué se usa, y cómo organiza el acceso a la base de datos para que tu código sea limpio, reutilizable y fácil de testear.

---

## 2.1. ¿Qué es el patrón DAO (Data Access Object)?

Imagina que tienes tu aplicación Java con clases como `Barco`, `Amarre` y `Regata`. Necesitas guardar, leer, actualizar y borrar estos objetos en la base de datos. ¿Dónde pones ese código?

La opción más tentadora (y la peor) sería mezclar el código SQL **directamente** en la lógica de tu aplicación:

```java
// ❌ MAL: mezclar lógica de negocio con acceso a datos
public void inscribirBarcoEnRegata(Long barcoId, Long regataId) {
    Connection conn = DriverManager.getConnection(...);
    PreparedStatement stmt = conn.prepareStatement(
        "INSERT INTO barco_regata (barco_id, regata_id) VALUES (?, ?)");
    stmt.setLong(1, barcoId);
    stmt.setLong(2, regataId);
    stmt.executeUpdate();
    // ... más lógica de negocio mezclada con SQL
}
```

El **patrón DAO** propone una solución elegante: **separar todo el código de acceso a datos en clases especializadas**.

> 📖 **Vocabulario:** El **patrón DAO (Data Access Object)** es un patrón de diseño que crea una capa de objetos dedicados exclusivamente a la comunicación con la base de datos, separándola del resto de la aplicación.

### 2.1.1. Separación de la lógica de acceso a datos

La idea es que, en lugar de mezclar código SQL en tu lógica de negocio, creas una **capa intermedia** de clases que se encargan exclusivamente de las operaciones con la base de datos:

```
  ┌──────────────────────────────┐
  │   Tu lógica de negocio       │  ← Solo se preocupa de la lógica
  │   (Servicios, Controladores) │
  └──────────────┬───────────────┘
                 │ llama a
                 ▼
  ┌──────────────────────────────┐
  │   Capa DAO                   │  ← Se encarga de la base de datos
  │   (BarcoDAO, AmarreDAO...)   │
  └──────────────┬───────────────┘
                 │ usa
                 ▼
  ┌──────────────────────────────┐
  │   Base de Datos (MySQL)      │
  └──────────────────────────────┘
```

Cada clase DAO se encarga de **toda la interacción** con la base de datos para un tipo específico de entidad. Por ejemplo:

- `BarcoDAO` → todas las operaciones sobre la tabla `barco`.
- `AmarreDAO` → todas las operaciones sobre la tabla `amarre`.
- `RegataDAO` → todas las operaciones sobre la tabla `regata`.

> 💡 **TIP:** Piensa en cada DAO como un **especialista** en una sola tabla. Si necesitas algo de la tabla `barco`, hablas con `BarcoDAO`. Nunca vas directamente a la base de datos desde otra parte del código.

### 2.1.2. Estructura: Interfaz DAO → Implementación DAO

El patrón DAO se organiza típicamente en dos partes:

1. **Interfaz DAO:** Define **qué operaciones** se pueden hacer (el contrato).
2. **Implementación DAO:** Define **cómo** se hacen esas operaciones (el código real).

```java
// 1. INTERFAZ: define QUÉ operaciones existen
public interface BarcoDAO {
    Barco findById(Long id);
    List<Barco> findAll();
    Barco save(Barco barco);
    Barco update(Barco barco);
    void delete(Barco barco);
}

// 2. IMPLEMENTACIÓN: define CÓMO se hacen
public class BarcoDAOImpl implements BarcoDAO {
    @Override
    public Barco findById(Long id) {
        // ... código Hibernate para buscar por ID
    }
    // ... resto de métodos
}
```

> 🔍 **¿Sabías que?** Esta separación entre interfaz e implementación es clave para la **programación orientada a objetos**. Si el día de mañana cambias Hibernate por otro framework, solo tienes que crear una nueva implementación (`BarcoDAOOtroFramework`). La interfaz y todo el código que la usa permanecen intactos.

---

## 2.2. Responsabilidades del DAO

Un DAO tiene dos responsabilidades principales:

### 2.2.1. Abstracción y encapsulamiento

Los DAOs **ocultan** los detalles de cómo se accede a la base de datos. El resto de la aplicación no necesita saber si usas Hibernate, JDBC puro, o cualquier otra tecnología. Solo sabe que puede llamar a `barcoDAO.save(barco)` y el barco se guardará.

> 📖 **Vocabulario:** **Abstracción** es esconder la complejidad interna de algo y exponer solo lo necesario. **Encapsulamiento** es agrupar datos y operaciones relacionadas dentro de una misma unidad (clase).

### 2.2.2. Operaciones CRUD (Create, Read, Update, Delete)

Todo DAO proporciona, como mínimo, las **cuatro operaciones básicas** sobre los datos:

| Operación | Significado | Método típico |
|-----------|-------------|---------------|
| **C**reate | Crear un registro nuevo | `save(entidad)` |
| **R**ead | Leer/buscar registros | `findById(id)`, `findAll()` |
| **U**pdate | Actualizar un registro existente | `update(entidad)` |
| **D**elete | Eliminar un registro | `delete(entidad)` |

> 🏆 **Buena práctica:** Estas cuatro operaciones son tan comunes que a veces se las llama directamente "el CRUD" de una entidad. Cuando alguien dice "haz el CRUD de Barco", quiere decir: crea los métodos para crear, leer, actualizar y borrar barcos.

---

## 2.3. Beneficios del patrón DAO

### 2.3.1. Desacoplamiento

El patrón DAO **reduce las dependencias** entre las diferentes partes de tu aplicación. Tu lógica de negocio no depende directamente de Hibernate, MySQL, ni de ninguna tecnología de base de datos concreta. Solo depende de la **interfaz** del DAO.

```java
// ✅ BIEN: la lógica de negocio solo conoce la interfaz
public class RegataService {
    private BarcoDAO barcoDAO;  // ← interfaz, no implementación

    public void inscribirBarco(Long barcoId, Regata regata) {
        Barco barco = barcoDAO.findById(barcoId);
        barco.getRegatas().add(regata);
        barcoDAO.update(barco);
    }
}
```

> 📖 **Vocabulario:** **Desacoplamiento** (*loose coupling*) significa que los componentes de tu aplicación tienen la mínima dependencia posible entre sí. Si cambias uno, los demás no se ven afectados.

### 2.3.2. Reutilización y mantenibilidad

Al centralizar el acceso a datos en clases especializadas, evitas **duplicar código**. Si necesitas buscar un barco por ID desde tres partes diferentes de tu aplicación, las tres llaman al mismo `barcoDAO.findById(id)`.

Si algún día necesitas cambiar cómo se busca (por ejemplo, añadir caché), solo modificas un punto: la implementación del DAO.

### 2.3.3. Facilidad para testing

Esta es quizás la ventaja más potente. Si tu lógica de negocio depende de una **interfaz** DAO, puedes crear un DAO "falso" (**mock**) para las pruebas que no necesite una base de datos real:

```java
// En los tests, puedes "simular" el DAO sin base de datos
BarcoDAO mockDAO = mock(BarcoDAO.class);
when(mockDAO.findById(1L)).thenReturn(new Barco("Test", "Velero"));

// Ahora puedes testear tu lógica de negocio sin MySQL
```

> 💡 **TIP:** Aprenderemos a hacer esto en detalle en el **Capítulo 9** cuando veamos JUnit 5 y Mockito. Por ahora, quédate con la idea de que la separación en DAOs hace que tu código sea **fácilmente testeable**.

---

## 2.4. DAO genérico vs DAO específico

Cuando tienes muchas entidades (Barco, Amarre, Regata...), te das cuenta de que las operaciones CRUD son **prácticamente iguales** para todas. La única diferencia es el tipo de dato.

Para evitar repetir el mismo código, puedes crear un **DAO genérico**:

```java
// DAO genérico: sirve para CUALQUIER entidad
public interface GenericDAO<T> {
    T findById(Long id);
    List<T> findAll();
    T save(T entity);
    T update(T entity);
    void delete(T entity);
}
```

Y luego, para operaciones específicas de una entidad, creas interfaces que **extienden** la genérica:

```java
// DAO específico: hereda el CRUD genérico + añade métodos propios
public interface BarcoDAO extends GenericDAO<Barco> {
    Barco findByNombre(String nombre);
    List<Barco> findByTipo(String tipo);
    Long countByTipo(String tipo);
}
```

> 📖 **Vocabulario:** **Genéricos** (`<T>`) son una característica de Java que permite crear clases e interfaces que funcionan con cualquier tipo de dato. La `T` se sustituye por el tipo concreto cuando se usa (ej: `GenericDAO<Barco>` donde `T` = `Barco`).

> 🏆 **Buena práctica:** Usa un DAO genérico para las operaciones comunes y DAOs específicos para las operaciones que son propias de cada entidad. Así aplicas el principio **DRY** (Don't Repeat Yourself).

---

## 2.5. Inversión de Control e Inyección de Dependencias

Esta sección cubre dos conceptos fundamentales que están profundamente conectados
y que aparecen a lo largo de todo el tutorial. Entenderlos bien aquí te dará
una base sólida para comprender por qué organizamos el código como lo hacemos.

### 2.5.1. Analogía: el restaurante

Antes de entrar en definiciones técnicas, piensa en cómo funciona un **restaurante**:

```
  SIN inversión de control (cocinas tú):
  ┌──────────────────────────────────────────────┐
  │  TÚ (el cliente):                            │
  │  1. Vas al mercado a comprar ingredientes    │
  │  2. Buscas la receta                         │
  │  3. Cocinas el plato                         │
  │  4. Lo sirves en la mesa                     │
  │  5. Finalmente... ¡comes!                    │
  │                                              │
  │  → TÚ controlas TODO el proceso              │
  └──────────────────────────────────────────────┘

  CON inversión de control (vas al restaurante):
  ┌──────────────────────────────────────────────┐
  │  TÚ (el cliente):                            │
  │  1. Te sientas                               │
  │  2. Pides "paella"                           │
  │  3. Te la traen hecha                        │
  │  4. ¡Comes!                                  │
  │                                              │
  │  → El RESTAURANTE controla el proceso        │
  │  → Tú solo dices QUÉ quieres, no CÓMO       │
  └──────────────────────────────────────────────┘
```

En la primera versión, **tú controlas** cada paso. En la segunda, **delegas el control**
al restaurante. Tú no decides qué aceite de oliva usar, ni a qué temperatura freír.
El restaurante se encarga de todo — tú solo pides lo que necesitas.

Esto es exactamente lo que ocurre en programación con la **Inversión de Control**.

### 2.5.2. Inversión de Control (IoC — *Inversion of Control*)

> 📖 **Vocabulario:** **Inversión de Control** (*Inversion of Control*, IoC) es un principio
> de diseño en el que el **flujo de control** de un programa se invierte: en lugar de que
> tu código controle cuándo se crean los objetos y cómo se conectan entre sí, es un
> mecanismo externo quien lo hace por ti.

En nuestro proyecto con Hibernate puro, tu código `main()` controla todo:

```java
// TÚ creas todo y conectas todo manualmente
public class Main {
    public static void main(String[] args) {
        // Tú decides qué implementación usar
        BarcoDAO barcoDAO = new BarcoDAOImpl();

        // Tú creas el servicio y le pasas la dependencia
        RegataService servicio = new RegataService(barcoDAO);

        // Tú decides cuándo llamar al método
        servicio.inscribirBarco(1L, 2L);
    }
}
```

Tú controlas **tres cosas**: qué objetos crear, cómo conectarlos, y cuándo usarlos.

Con la Inversión de Control, un mecanismo externo se encarga de crear los objetos
y conectarlos. Tu código solo se ocupa de la **lógica de negocio**:

```java
// Con IoC: alguien "de fuera" crea y conecta los objetos por ti
public class RegataService {
    private BarcoDAO barcoDAO;   // ← NO creas la dependencia tú

    public RegataService(BarcoDAO barcoDAO) {
        this.barcoDAO = barcoDAO;  // ← te la pasan desde fuera
    }

    public void inscribirBarco(Long barcoId, Long regataId) {
        // Tu código solo contiene la LÓGICA de negocio
        Barco barco = barcoDAO.findById(barcoId);
        // ...
    }
}
```

La diferencia fundamental es: **¿quién tiene el control?**

| Aspecto | Sin IoC | Con IoC |
|---------|---------|---------|
| ¿Quién crea los objetos? | Tu código (`new`) | Un mecanismo externo |
| ¿Quién conecta las dependencias? | Tu código (manualmente) | El mecanismo externo |
| Tu código se encarga de... | Todo | Solo la lógica de negocio |

### 2.5.3. Principios SOLID — la "D": Inversión de Dependencias

**SOLID** es un acrónimo que agrupa los **cinco principios fundamentales** del diseño
orientado a objetos. Cada letra representa un principio:

| Letra | Principio | Significado resumido |
|-------|-----------|---------------------|
| **S** | *Single Responsibility* (Responsabilidad Única) | Cada clase tiene **una sola responsabilidad** |
| **O** | *Open/Closed* (Abierto/Cerrado) | Abierta a extensión, **cerrada a modificación** |
| **L** | *Liskov Substitution* (Sustitución de Liskov) | Una subclase debe poder sustituir a su clase padre |
| **I** | *Interface Segregation* (Segregación de Interfaces) | Mejor **varias interfaces pequeñas** que una gigante |
| **D** | *Dependency Inversion* (Inversión de Dependencias) | Depende de **abstracciones**, no de implementaciones |

Veamos cómo estos principios ya aparecen en nuestro proyecto:

| Principio | Ejemplo en nuestro proyecto |
|-----------|----------------------------|
| **S** (Responsabilidad única) | `BarcoDAO` solo accede a datos. `RegataService` solo contiene lógica de negocio. Cada uno hace una cosa. |
| **O** (Abierto/Cerrado) | Puedes añadir `BarcoDAOJdbcImpl` sin modificar la interfaz `BarcoDAO` ni `RegataService` |
| **I** (Segregación de interfaces) | `BarcoDAO` tiene sus métodos y `RegataDAO` los suyos — no una interfaz única gigante |
| **D** (Inversión de dependencias) | `RegataService` depende de la **interfaz** `BarcoDAO`, no de `BarcoDAOImpl` |

De los cinco, el que más nos importa ahora es el último: la **"D"** — el Principio
de Inversión de Dependencias (*Dependency Inversion Principle*).

> 📖 **Vocabulario:** El **Principio de Inversión de Dependencias**
> (*Dependency Inversion Principle*) establece dos reglas:
>
> 1. Los módulos de alto nivel **no deben depender** de módulos de bajo nivel.
>    Ambos deben depender de **abstracciones** (interfaces).
> 2. Las abstracciones **no deben depender** de los detalles.
>    Los detalles deben depender de las abstracciones.

¿Qué significa "alto nivel" y "bajo nivel"? En nuestra arquitectura:

- **Alto nivel:** `RegataService` — contiene la **lógica de negocio** (inscribir barcos, validar reglas)
- **Bajo nivel:** `BarcoDAOHibernate` — contiene los **detalles técnicos** (cómo hablar con MySQL)

El principio dice: el servicio no debe conocer los detalles de Hibernate.
Solo debe conocer la **interfaz** (la abstracción). Veámoslo en un diagrama:

```
  ❌ SIN Inversión de Dependencias:

  RegataService  ─────────►  BarcoDAOHibernate
  (alto nivel)                (bajo nivel, detalle técnico)

  "El servicio depende DIRECTAMENTE de Hibernate"
  → Si cambias Hibernate por otro framework, tienes que modificar el servicio

  ✅ CON Inversión de Dependencias:

  RegataService  ─────────►  BarcoDAO (interfaz)
  (alto nivel)                (abstracción)
                                  ▲
                                  │ implementa
                              BarcoDAOHibernate
                              (bajo nivel, detalle técnico)

  "El servicio depende de la INTERFAZ, no de Hibernate"
  → Si cambias Hibernate, creas otra implementación y el servicio NO se toca
```

Cuando en la sección 2.1 definimos `BarcoDAO` como interfaz y `BarcoDAOImpl`
como implementación, **ya estábamos aplicando este principio** sin saberlo.

> 💡 **TIP:** No necesitas memorizar los cinco principios SOLID ahora.
> Lo importante es entender la **"D"** porque es la base de todo lo que
> hacemos con el patrón DAO: programar contra **interfaces**, no contra
> **implementaciones concretas**.

### 2.5.4. Inyección de Dependencias (*Dependency Injection*)

Ahora que entendemos la Inversión de Control (el principio general) y la Inversión
de Dependencias (la regla de diseño), la **Inyección de Dependencias** es la
**técnica concreta** para ponerlos en práctica.

> 📖 **Vocabulario:** **Inyección de Dependencias** (*Dependency Injection*, DI)
> es un patrón en el que los objetos que una clase necesita para funcionar
> (sus dependencias) se proporcionan **desde fuera** en lugar de ser creados internamente.

Veamos la diferencia con un ejemplo claro:

```java
// ❌ MAL: el servicio CREA su propia dependencia
public class RegataService {
    private BarcoDAO barcoDAO = new BarcoDAOImpl();  // ← acoplado
    //                           ^^^^^^^^^^^^
    //  El servicio SABE que usa BarcoDAOImpl.
    //  Si quieres cambiar a otra implementación, tienes que
    //  modificar esta clase.
}
```

```java
// ✅ BIEN: la dependencia se INYECTA desde fuera (por constructor)
public class RegataService {
    private final BarcoDAO barcoDAO;    // ← solo conoce la INTERFAZ

    public RegataService(BarcoDAO barcoDAO) {   // ← se recibe por constructor
        this.barcoDAO = barcoDAO;
    }
}
```

¿Por qué es mejor? Tres razones:

1. **Desacoplamiento:** `RegataService` no sabe qué implementación usa — solo conoce la interfaz
2. **Flexibilidad:** Puedes cambiar la implementación sin tocar `RegataService`
3. **Testabilidad:** En los tests inyectas un mock; en producción, la implementación real

Y así es como se usa en nuestro `main()`:

```java
public static void main(String[] args) {
    // TÚ decides qué implementación inyectar
    BarcoDAO barcoDAO = new BarcoDAOImpl();   // ← implementación con Hibernate

    // Inyectas la dependencia por constructor
    RegataService servicio = new RegataService(barcoDAO);

    // Ahora puedes usar el servicio normalmente
    servicio.inscribirBarco(1L, 2L);
}
```

En los tests es igual de fácil:

```java
// En tests, inyectas un MOCK en lugar de la implementación real
BarcoDAO mockDAO = mock(BarcoDAO.class);
when(mockDAO.findById(1L)).thenReturn(new Barco("Test", "Velero"));

RegataService servicio = new RegataService(mockDAO);  // ← mock inyectado
// Ahora puedes testear sin necesidad de base de datos
```

> 🏆 **Buena práctica:** La inyección por **constructor** es la forma preferida
> porque hace que la dependencia sea **obligatoria** (si no la pasas, no compila)
> y permite declarar el campo como `final` (inmutable después de la creación).

---

## 2.6. Visión general de la arquitectura del proyecto

Con todo lo que hemos visto, nuestro proyecto tiene esta **arquitectura por capas**:

```
  ┌─────────────────────────────────────────────────────┐
  │  CAPA DE PRESENTACIÓN                               │
  │  (Vistas Thymeleaf / API REST)                      │
  │  → Capítulos 13-18                                  │
  └─────────────────────┬───────────────────────────────┘
                        │ usa
                        ▼
  ┌─────────────────────────────────────────────────────┐
  │  CAPA DE SERVICIO                                   │
  │  (BarcoService, RegataService...)                   │
  │  → Lógica de negocio                                │
  │  → Capítulo 12                                      │
  └─────────────────────┬───────────────────────────────┘
                        │ usa
                        ▼
  ┌─────────────────────────────────────────────────────┐
  │  CAPA DE ACCESO A DATOS (DAO / Repository)          │
  │  (BarcoDAO, AmarreDAO... / JpaRepository)           │
  │  → Capítulos 7, 11                                  │
  └─────────────────────┬───────────────────────────────┘
                        │ usa
                        ▼
  ┌─────────────────────────────────────────────────────┐
  │  CAPA DE MODELO                                     │
  │  (Barco.java, Amarre.java, Regata.java)             │
  │  → Entidades mapeadas con Hibernate                 │
  │  → Capítulos 4-6                                    │
  └─────────────────────┬───────────────────────────────┘
                        │ persiste en
                        ▼
  ┌─────────────────────────────────────────────────────┐
  │  BASE DE DATOS (MySQL)                              │
  │  → Tablas: barco, amarre, regata, barco_regata      │
  └─────────────────────────────────────────────────────┘
```

> 🏆 **Buena práctica:** Esta arquitectura por capas es la forma estándar de organizar aplicaciones profesionales en Java. Cada capa tiene una responsabilidad clara y solo se comunica con la capa inmediatamente inferior. A esto se le llama **separación de responsabilidades** (*Separation of Concerns*).

> ⚠️ **CUIDADO:** Una vista (Thymeleaf) **nunca** debe acceder directamente a un DAO. Siempre debe pasar por el servicio. Un DAO **nunca** debe contener lógica de negocio. Respetar esta separación es fundamental para un código mantenible.

---

## 📝 Glosario del Capítulo 2

| Término | Definición |
|---------|------------|
| **Patrón DAO** | Data Access Object — patrón de diseño que separa el acceso a datos en clases especializadas |
| **Interfaz** | Contrato que define qué métodos debe tener una clase, sin especificar cómo se implementan |
| **Implementación** | Clase concreta que proporciona el código real para los métodos definidos en una interfaz |
| **Desacoplamiento** | Minimizar las dependencias entre componentes para que los cambios en uno no afecten a otros |
| **Abstracción** | Ocultar la complejidad interna y exponer solo lo necesario |
| **Encapsulamiento** | Agrupar datos y operaciones relacionadas dentro de una clase |
| **Genéricos (`<T>`)** | Mecanismo de Java para crear código que funciona con cualquier tipo de dato |
| **SOLID** | Cinco principios de diseño orientado a objetos (S, O, L, I, D) |
| **Inversión de Control (IoC)** | Principio donde un mecanismo externo controla la creación y conexión de objetos |
| **Inversión de Dependencias** | Principio SOLID-D: depende de abstracciones (interfaces), no de implementaciones concretas |
| **Inyección de Dependencias (DI)** | Proporcionar las dependencias de un objeto desde fuera, en lugar de crearlas internamente |
| **DRY** | Don't Repeat Yourself — principio que dice que no debes duplicar código |
| **Arquitectura por capas** | Organización de la aplicación en niveles con responsabilidades separadas |
| **Separación de responsabilidades** | Principio que dice que cada componente debe tener una única función bien definida |
| **Mock** | Objeto simulado que imita el comportamiento de un objeto real, usado en pruebas unitarias |

---

✅ **Checkpoint:** Al terminar este capítulo debes entender...
- Qué es el patrón DAO y por qué separa el acceso a datos del resto de la aplicación.
- La diferencia entre la interfaz DAO (el contrato) y la implementación DAO (el código).
- Qué es un DAO genérico y cuándo usar uno específico.
- Qué es la **Inversión de Control** (IoC) y por qué es mejor delegar el control.
- Qué son los principios **SOLID** y en especial la **"D"** (Inversión de Dependencias).
- Qué es la **Inyección de Dependencias** y por qué inyectar por constructor.
- La arquitectura por capas de nuestro proyecto (Modelo → DAO → Servicio → Controlador → Vista).
- Que aún **no hemos escrito código del proyecto** — eso empieza en el **Capítulo 3**.

---

*Capítulo anterior: [← Capítulo 1. Introducción a la Persistencia en Java](Cap01_Fundamentos.md)*

*Siguiente capítulo: [Capítulo 3. Configuración del Entorno de Desarrollo →](Cap03_Configuracion.md)*
