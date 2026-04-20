# Capítulo 8. Consultas Avanzadas

---

🎯 **Objetivo del capítulo:** Ampliar las interfaces DAO con métodos de consulta personalizados e implementarlos usando tres tecnologías: HQL (Hibernate Query Language), SQL nativo y Criteria API. Entender cuándo usar cada una.

---

## 8.1. Ampliación de las interfaces DAO con métodos personalizados

Los métodos CRUD básicos (`findById`, `findAll`, `save`, `update`, `delete`) no siempre son suficientes. Necesitamos buscar barcos por nombre, filtrar por eslora, contar por tipo...

Ampliemos la interfaz `BarcoDAO`:

```java
package com.mycompany.hibernate.dao;

import com.mycompany.hibernate.modelo.Barco;
import java.util.List;

public interface BarcoDAO {
    // ── Métodos CRUD existentes ──
    Barco findById(Long id);
    List<Barco> findAll();
    Barco save(Barco barco);
    Barco update(Barco barco);
    void delete(Barco barco);

    // ── Métodos personalizados NUEVOS ──
    Barco findByNombre(String nombre);
    List<Barco> findByEsloraGreaterThan(Integer eslora);
    List<Barco> findByNombreOrderByCapacidadDesc(String nombre);
    List<Barco> findByTipo(String tipo);
    Long countByTipo(String tipo);
}
```

> 🏆 **Buena práctica:** Los nombres de los métodos deben ser **descriptivos** y seguir una convención: `findBy` + atributo + condición. Esta convención será muy importante cuando migremos a Spring Data (Capítulo 11), donde Spring genera las consultas automáticamente a partir del nombre del método.

---

## 8.2. HQL (Hibernate Query Language)

HQL es un lenguaje de consultas **orientado a objetos**, similar a SQL pero que opera sobre entidades Java en lugar de tablas.

> 📖 **Vocabulario:** **HQL (Hibernate Query Language)** es el lenguaje de consultas propio de Hibernate. En lugar de escribir `SELECT * FROM barco`, escribes `FROM Barco` (nombre de la clase Java, no de la tabla SQL).

### Diferencias clave entre HQL y SQL

| Aspecto | SQL | HQL |
|---------|-----|-----|
| Opera sobre | **Tablas** y columnas | **Clases** y atributos |
| Nombres | `tabla.columna` | `Entidad.atributo` |
| Case-sensitive | Depende de la BD | Los nombres de clase sí, los de atributo no |
| Ejemplo | `SELECT * FROM barco WHERE nombre = 'X'` | `FROM Barco WHERE nombre = 'X'` |

### 8.2.1. Sintaxis básica

```sql
-- Seleccionar todos los barcos
FROM Barco

-- Filtrar con WHERE
FROM Barco WHERE tipo = :tipo

-- Seleccionar campos específicos (proyección)
SELECT b.nombre, b.eslora FROM Barco b

-- Funciones de agregación
SELECT COUNT(*) FROM Barco WHERE tipo = :tipo
SELECT AVG(b.eslora) FROM Barco b
```

### 8.2.2. Parámetros con nombre

En lugar de concatenar valores en la consulta (¡peligro de SQL Injection!), usa **parámetros con nombre**:

```java
// ✅ BIEN: parámetro con nombre (:nombre)
session.createQuery("FROM Barco WHERE nombre = :nombre", Barco.class)
       .setParameter("nombre", "Estrella del Mar")
       .uniqueResult();

// ❌ MAL: concatenación directa (vulnerable a SQL Injection)
session.createQuery("FROM Barco WHERE nombre = '" + nombre + "'", Barco.class);
```

> ⚠️ **CUIDADO:** NUNCA concatenes valores del usuario directamente en una consulta. Siempre usa parámetros con nombre (`:nombre`). Esto previene ataques de **SQL Injection**.

> 📖 **Vocabulario:** **SQL Injection** es un ataque donde un usuario malintencionado introduce código SQL en los datos de entrada para manipular la consulta. Los parámetros con nombre evitan este ataque.

### 8.2.3. Implementación de los métodos personalizados con HQL

```java
@Override
public Barco findByNombre(String nombre) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createQuery("FROM Barco WHERE nombre = :nombre", Barco.class)
                .setParameter("nombre", nombre)
                .uniqueResult();   // Devuelve un solo resultado (o null)
    }
}

@Override
public List<Barco> findByEsloraGreaterThan(Integer eslora) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createQuery("FROM Barco WHERE eslora > :eslora", Barco.class)
                .setParameter("eslora", eslora)
                .getResultList();  // Devuelve una lista de resultados
    }
}

@Override
public List<Barco> findByNombreOrderByCapacidadDesc(String nombre) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createQuery(
                "FROM Barco WHERE nombre = :nombre ORDER BY capacidad DESC", Barco.class)
                .setParameter("nombre", nombre)
                .getResultList();
    }
}

@Override
public List<Barco> findByTipo(String tipo) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createQuery("FROM Barco WHERE tipo = :tipo", Barco.class)
                .setParameter("tipo", tipo)
                .getResultList();
    }
}

@Override
public Long countByTipo(String tipo) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createQuery(
                "SELECT COUNT(*) FROM Barco WHERE tipo = :tipo", Long.class)
                .setParameter("tipo", tipo)
                .uniqueResult();
    }
}
```

> 💡 **TIP:** Usa `uniqueResult()` cuando esperas **un solo resultado** (o null). Usa `getResultList()` cuando esperas **varios resultados** (puede ser una lista vacía).

### 8.2.4. JOIN entre entidades

HQL permite navegar las relaciones entre entidades:

```java
// Barcos que participan en una regata específica
session.createQuery(
    "FROM Barco b JOIN b.regatas r WHERE r.nombre = :nombreRegata", Barco.class)
    .setParameter("nombreRegata", "Copa Mediterráneo")
    .getResultList();
```

### 8.2.5. Funciones de agregación

```java
// Contar, promediar, máximo, mínimo
SELECT COUNT(b) FROM Barco b
SELECT AVG(b.eslora) FROM Barco b
SELECT MAX(b.capacidad) FROM Barco b
SELECT MIN(b.eslora) FROM Barco b WHERE b.tipo = :tipo
```

---

## 8.3. SQL Nativo con `createNativeQuery()`

A veces necesitas escribir SQL "puro", por ejemplo para consultas muy específicas de MySQL o para optimizar el rendimiento:

```java
@Override
public Barco findByNombre(String nombre) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return (Barco) session.createNativeQuery(
                "SELECT * FROM barco WHERE nombre = :nombre", Barco.class)
                .setParameter("nombre", nombre)
                .getSingleResult();
    }
}

@Override
public List<Barco> findByEsloraGreaterThan(Integer eslora) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        return session.createNativeQuery(
                "SELECT * FROM barco WHERE eslora > :eslora", Barco.class)
                .setParameter("eslora", eslora)
                .getResultList();
    }
}
```

> ⚠️ **CUIDADO:** Con SQL nativo, usas nombres de **tablas y columnas** (SQL), no de clases y atributos (HQL). Si cambias de MySQL a PostgreSQL, puede que tengas que reescribir las consultas.

> 🔍 **¿Sabías que?** El SQL nativo es la única opción cuando necesitas funciones específicas de tu motor de BD que no existen en HQL, como `MATCH ... AGAINST` de MySQL para búsqueda *full-text*.

---

## 8.4. Criteria API

La Criteria API permite construir consultas de forma **programática** usando objetos Java en lugar de cadenas de texto. Esto la hace **type-safe** (errores en tiempo de compilación, no en ejecución).

```java
@Override
public Barco findByNombre(String nombre) {
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        // 1. Obtener el constructor de criterios
        CriteriaBuilder cb = session.getCriteriaBuilder();

        // 2. Crear la consulta tipada
        CriteriaQuery<Barco> cr = cb.createQuery(Barco.class);

        // 3. Definir la raíz (FROM barco)
        Root<Barco> root = cr.from(Barco.class);

        // 4. Añadir la condición WHERE nombre = :nombre
        cr.select(root).where(cb.equal(root.get("nombre"), nombre));

        // 5. Ejecutar
        Query<Barco> query = session.createQuery(cr);
        return query.uniqueResult();
    }
}
```

> 📖 **Vocabulario:**
> - **`CriteriaBuilder`**: fábrica que crea las piezas de la consulta (condiciones, predicados).
> - **`CriteriaQuery<T>`**: la consulta en sí misma, tipada.
> - **`Root<T>`**: representa la tabla raíz del `FROM`.

> 💡 **TIP:** La Criteria API es más verbosa que HQL, pero tiene una ventaja clave: permite construir consultas **dinámicas**. Puedes ir añadiendo condiciones `WHERE` según los filtros que el usuario haya seleccionado, sin concatenar strings.

---

## 8.5. Tabla comparativa: HQL vs SQL nativo vs Criteria API

| Aspecto | HQL | SQL Nativo | Criteria API |
|---------|-----|-----------|--------------|
| **Portabilidad** | ✅ Independiente de BD | ❌ Específico de BD | ✅ Independiente de BD |
| **Legibilidad** | ✅ Clara y concisa | ✅ Familiar para quien sabe SQL | ⚠️ Verbosa |
| **Type-safe** | ❌ Strings (errores en ejecución) | ❌ Strings | ✅ Errores en compilación |
| **Consultas dinámicas** | ⚠️ Concatenación difícil | ⚠️ Concatenación difícil | ✅ Fácil y seguro |
| **Rendimiento** | ✅ Bueno | ✅ Mejor (directo) | ✅ Bueno |
| **Cuándo usarla** | ✅ Opción por defecto | Funciones específicas BD | Filtros dinámicos complejos |

> 🏆 **Buena práctica:** Usa **HQL** como opción por defecto (es la más legible). Recurre a **SQL nativo** solo cuando necesites funciones específicas de la base de datos. Usa **Criteria API** cuando necesites construir consultas dinámicas con muchos filtros opcionales.

---

## 8.6. Ejercicio

Implementa estos métodos adicionales en `AmarreDAO` y `RegataDAO`:

**AmarreDAO:**
- `findByUbicacion(String ubicacion)`
- `findByPrecioLessThan(double precio)`
- `findByElectricidad(boolean electricidad)`

**RegataDAO:**
- `findByLugar(String lugar)`
- `findByDistanciaGreaterThan(int distancia)`

---

## 📝 Glosario del Capítulo 8

| Término | Definición |
|---------|------------|
| **HQL** | Hibernate Query Language — lenguaje de consultas orientado a objetos propio de Hibernate |
| **SQL Nativo** | SQL estándar ejecutado directamente por Hibernate sin traducción |
| **Criteria API** | API para construir consultas programáticamente con objetos Java |
| **`CriteriaBuilder`** | Fábrica de predicados y condiciones para consultas Criteria |
| **`CriteriaQuery`** | Objeto que representa una consulta Criteria tipada |
| **`Root`** | Representa la tabla raíz de la consulta (el FROM) |
| **Parámetro con nombre** | Variable en una consulta (`:nombre`) que se sustituye por un valor real |
| **SQL Injection** | Ataque de seguridad que inyecta código SQL malicioso en consultas |
| **Proyección** | Seleccionar solo campos específicos en lugar de la entidad completa |
| **`uniqueResult()`** | Devuelve un solo resultado o null |
| **`getResultList()`** | Devuelve una lista de resultados (puede estar vacía) |
| **Type-safe** | Que los errores se detectan en tiempo de compilación, no de ejecución |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Tener la interfaz `BarcoDAO` ampliada con 5 métodos personalizados.
- Saber implementar consultas con HQL, SQL nativo y Criteria API.
- Entender las diferencias entre las tres tecnologías y cuándo usar cada una.
- Saber usar parámetros con nombre para evitar SQL Injection.

---

*Capítulo anterior: [← Capítulo 7. Implementación del Patrón DAO](Cap07_DAO.md)*

*Siguiente capítulo: [Capítulo 9. Pruebas Unitarias →](Cap09_Testing.md)*
