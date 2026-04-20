# Capítulo 11. Repositorios con Spring Data JPA

---

🎯 **Objetivo del capítulo:** Reemplazar nuestros DAOs manuales (156 líneas de `BarcoDAOImpl`) por repositorios Spring Data JPA (3 líneas de interfaz). Entender cómo Spring genera las implementaciones automáticamente.

---

## 11.1. El gran cambio: de 156 líneas a 3

En Hibernate puro, implementamos **manualmente** cada operación CRUD:

```java
// BarcoDAOImpl.java — 156 líneas de código manual
public class BarcoDAOImpl implements BarcoDAO {

    @Override
    public Barco findById(Long id) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.find(Barco.class, id);
        }
    }

    @Override
    public Barco save(Barco barco) {
        Transaction transaction = null;
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            transaction = session.beginTransaction();
            session.save(barco);
            transaction.commit();
            return barco;
        } catch (Exception e) {
            if (transaction != null) transaction.rollback();
            throw new RuntimeException("Error al guardar barco", e);
        }
    }

    // ... findAll(), update(), delete() — más código repetitivo
}
```

Con Spring Data JPA, **todo eso se reduce a una interfaz**:

```java
// BarcoRepository.java — ¡3 líneas!
public interface BarcoRepository extends JpaRepository<Barco, Long> {
}
```

> 💡 **¿Sabías que?** No hay ninguna clase que implemente `BarcoRepository`. Spring Data JPA **genera la implementación automáticamente** en tiempo de ejecución usando *proxies dinámicos*. Crea un proxy que implementa los métodos heredados de `JpaRepository`.

---

## 11.2. ¿Qué proporciona `JpaRepository`?

`JpaRepository<Barco, Long>` recibe dos parámetros genéricos:
- `Barco` → el tipo de la entidad
- `Long` → el tipo de la clave primaria

Y proporciona **gratis** estos métodos:

| Método | Equivale a | SQL generado |
|--------|-----------|-------------|
| `findAll()` | `BarcoDAOImpl.findAll()` | `SELECT * FROM barco` |
| `findById(1L)` | `BarcoDAOImpl.findById(1L)` | `SELECT * FROM barco WHERE id = 1` |
| `save(barco)` | `BarcoDAOImpl.save(barco)` | `INSERT INTO barco ...` |
| `save(barcoExistente)` | `BarcoDAOImpl.update(barco)` | `UPDATE barco SET ... WHERE id = ?` |
| `deleteById(1L)` | `BarcoDAOImpl.delete(barco)` | `DELETE FROM barco WHERE id = 1` |
| `count()` | (no teníamos) | `SELECT COUNT(*) FROM barco` |
| `existsById(1L)` | (no teníamos) | `SELECT COUNT(*) FROM barco WHERE id = 1` |
| `findAll(Sort.by("nombre"))` | (no teníamos) | `SELECT * FROM barco ORDER BY nombre` |
| `findAll(PageRequest.of(0,10))` | (no teníamos) | `SELECT * FROM barco LIMIT 10 OFFSET 0` |

> ⚠️ **CUIDADO:** En Spring Data JPA, `save()` sirve tanto para **crear** como para **actualizar**. Si el objeto tiene ID nulo → INSERT. Si tiene ID → UPDATE. No hay método `update()` separado.

---

## 11.3. Consultas derivadas (Derived Queries)

Además del CRUD, puedes añadir métodos personalizados. Spring genera el SQL **a partir del nombre del método**:

```java
public interface BarcoRepository extends JpaRepository<Barco, Long> {

    // Spring genera: SELECT * FROM barco WHERE nombre = ?
    Barco findByNombre(String nombre);

    // Spring genera: SELECT * FROM barco WHERE tipo = ?
    List<Barco> findByTipo(String tipo);

    // Spring genera: SELECT * FROM barco WHERE eslora > ?
    List<Barco> findByEsloraGreaterThan(int eslora);

    // Spring genera: SELECT * FROM barco WHERE eslora BETWEEN ? AND ?
    List<Barco> findByEsloraBetween(int min, int max);

    // Spring genera: SELECT * FROM barco WHERE nombre LIKE '%?%'
    List<Barco> findByNombreContaining(String texto);

    // Spring genera: SELECT COUNT(*) FROM barco WHERE tipo = ?
    Long countByTipo(String tipo);
}
```

### Reglas de nombres para consultas derivadas:

| Palabra clave | Significado | Ejemplo |
|--------------|-------------|---------|
| `findBy` | Buscar por campo | `findByNombre(String)` |
| `countBy` | Contar por campo | `countByTipo(String)` |
| `GreaterThan` | Mayor que | `findByEsloraGreaterThan(int)` |
| `LessThan` | Menor que | `findByPrecioLessThan(double)` |
| `Between` | Entre dos valores | `findByEsloraBetween(int, int)` |
| `Containing` | Contiene texto (LIKE) | `findByNombreContaining(String)` |
| `IsNull` | Es nulo | `findByBarcoIsNull()` |
| `IsNotNull` | No es nulo | `findByBarcoIsNotNull()` |
| `OrderBy` | Ordenar | `findByTipoOrderByNombreAsc(String)` |

> 💡 **TIP:** Si el nombre del método no es válido, Spring **no arrancará** y te dará un error indicando que no puede derivar la consulta. Es un error en tiempo de arranque, no en tiempo de ejecución — lo que es mejor.

---

## 11.4. Consultas personalizadas con `@Query`

Para consultas complejas que no se pueden expresar con nombres de métodos, usa `@Query`:

```java
public interface BarcoRepository extends JpaRepository<Barco, Long> {

    // Consulta JPQL personalizada
    @Query("SELECT b FROM Barco b JOIN FETCH b.regatas WHERE b.id = :id")
    Barco findByIdWithRegatas(@Param("id") Long id);

    // Barcos sin amarre asignado
    @Query("SELECT b FROM Barco b WHERE b.amarre IS NULL")
    List<Barco> findSinAmarre();

    // Consulta nativa SQL (usa nombres de TABLAS, no de CLASES)
    @Query(value = "SELECT * FROM barco WHERE tipo = ?1", nativeQuery = true)
    List<Barco> buscarPorTipoNativo(String tipo);
}
```

### Comparación: Consulta derivada vs @Query

```java
// Consulta derivada: Spring deduce el SQL del nombre
List<Barco> findByTipoAndEsloraGreaterThan(String tipo, int eslora);

// @Query: tú escribes la consulta JPQL
@Query("SELECT b FROM Barco b WHERE b.tipo = :tipo AND b.eslora > :eslora")
List<Barco> buscarPorTipoYEslora(@Param("tipo") String tipo, @Param("eslora") int eslora);

// Ambos generan el mismo SQL — elige la que sea más legible
```

> 📖 **Vocabulario:**
> - **JPQL**: Java Persistence Query Language. Consultas orientadas a objetos (usa nombres de clases y atributos, no tablas y columnas).
> - **Consulta nativa**: SQL puro. Usa nombres de tablas de la BD. Se indica con `nativeQuery = true`.
> - **`@Param`**: vincula un parámetro del método a un parámetro de la consulta (`:nombre`).

---

## 11.5. Comparación completa: DAO manual vs Repositorio

| Aspecto | DAO Manual (Hibernate) | Repository (Spring Data) |
|---------|----------------------|--------------------------|
| **Definición** | Interfaz + Clase implementación | Solo interfaz |
| **CRUD** | Manual (156 líneas) | Automático (0 líneas) |
| **Transacciones** | `try { tx.begin(); ... tx.commit(); }` | Automáticas |
| **Sesiones** | `HibernateUtil.getSessionFactory().openSession()` | Transparentes |
| **Consultas custom** | Método con HQL manual | Nombre del método o `@Query` |
| **Errores** | RuntimeException manual | DataAccessException automática |
| **Testing** | Mock de Session, Transaction, Query | Mock del repositorio directamente |

---

## 11.6. Nuestros tres repositorios completos

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/repository/BarcoRepository.java`

```java
package com.example.marina.repository;

import com.example.marina.model.Barco;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

public interface BarcoRepository extends JpaRepository<Barco, Long> {
    Barco findByNombre(String nombre);
    List<Barco> findByTipo(String tipo);
    List<Barco> findByEsloraGreaterThan(int eslora);
    Long countByTipo(String tipo);

    @Query("SELECT b FROM Barco b JOIN FETCH b.regatas WHERE b.id = :id")
    Barco findByIdWithRegatas(@Param("id") Long id);

    @Query("SELECT b FROM Barco b WHERE b.amarre IS NULL")
    List<Barco> findSinAmarre();
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/repository/AmarreRepository.java`

```java
package com.example.marina.repository;

import com.example.marina.model.Amarre;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface AmarreRepository extends JpaRepository<Amarre, Long> {
    Amarre findByUbicacion(String ubicacion);
    List<Amarre> findByPrecioLessThan(double precio);
    List<Amarre> findByElectricidad(boolean electricidad);
    List<Amarre> findByBarcoIsNull();  // Amarres libres
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/repository/RegataRepository.java`

```java
package com.example.marina.repository;

import com.example.marina.model.Regata;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface RegataRepository extends JpaRepository<Regata, Long> {
    Regata findByNombre(String nombre);
    List<Regata> findByLugar(String lugar);
    List<Regata> findByDistanciaGreaterThan(int distancia);
}
```

---

## 📝 Glosario del Capítulo 11

| Término | Definición |
|---------|------------|
| **`JpaRepository`** | Interfaz de Spring Data que proporciona CRUD automático |
| **Proxy dinámico** | Objeto generado en tiempo de ejecución que implementa una interfaz |
| **Consulta derivada** | Consulta SQL generada automáticamente del nombre del método |
| **`@Query`** | Anotación para definir consultas JPQL o nativas personalizadas |
| **`@Param`** | Vincula un parámetro del método a un parámetro de la consulta |
| **JPQL** | Lenguaje de consultas orientado a objetos |
| **`nativeQuery`** | Flag para indicar que la consulta es SQL puro |
| **`JoinColumn`** | Define la columna de clave foránea en una relación |
| **HikariCP** | Pool de conexiones usado por Spring Boot (reemplaza el pool manual de Hibernate) |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Saber crear un repositorio extendiendo `JpaRepository`
- Entender que Spring genera la implementación automáticamente
- Saber crear consultas derivadas con nombres de método
- Saber usar `@Query` para consultas complejas
- Haber eliminado mentalmente los DAOs manuales del Bloque II

---

*Capítulo anterior: [← Capítulo 10. Intro Spring Boot](Cap10_SpringBoot.md)*

*Siguiente capítulo: [Capítulo 12. Capa de Servicio →](Cap12_Servicios.md)*
