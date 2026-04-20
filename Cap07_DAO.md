# Capítulo 7. Implementación del Patrón DAO

---

🎯 **Objetivo del capítulo:** Implementar las interfaces y clases DAO para las tres entidades del proyecto (Barco, Amarre, Regata) usando Hibernate, aplicando todo lo aprendido en los capítulos anteriores sobre sesiones, transacciones y estados.

---

## 7.1. Definir las interfaces DAO

En el Capítulo 2 aprendimos que el patrón DAO separa el acceso a datos en dos partes: la **interfaz** (QUÉ se puede hacer) y la **implementación** (CÓMO se hace). Empecemos por las interfaces.

Crea estas interfaces en el paquete `com.mycompany.hibernate.dao`:

### 7.1.1. `BarcoDAO`

```java
package com.mycompany.hibernate.dao;

import com.mycompany.hibernate.modelo.Barco;
import java.util.List;

public interface BarcoDAO {

    // Buscar un barco por su ID
    Barco findById(Long id);

    // Obtener todos los barcos
    List<Barco> findAll();

    // Guardar un barco nuevo
    Barco save(Barco barco);

    // Actualizar un barco existente
    Barco update(Barco barco);

    // Eliminar un barco
    void delete(Barco barco);
}
```

### 7.1.2. `AmarreDAO`

```java
package com.mycompany.hibernate.dao;

import com.mycompany.hibernate.modelo.Amarre;
import java.util.List;

public interface AmarreDAO {
    Amarre findById(Long id);
    List<Amarre> findAll();
    Amarre save(Amarre amarre);
    Amarre update(Amarre amarre);
    void delete(Amarre amarre);
}
```

### 7.1.3. `RegataDAO`

```java
package com.mycompany.hibernate.dao;

import com.mycompany.hibernate.modelo.Regata;
import java.util.List;

public interface RegataDAO {
    Regata findById(Long id);
    List<Regata> findAll();
    Regata save(Regata regata);
    Regata update(Regata regata);
    void delete(Regata regata);
}
```

> 🏆 **Buena práctica:** Fíjate en que las tres interfaces son prácticamente iguales, solo cambia el tipo de entidad. Esto confirma lo que vimos en el Capítulo 2: podríamos crear un `GenericDAO<T>` para no repetir código. Pero por claridad didáctica, empezamos con interfaces específicas.

---

## 7.2. Implementar las clases DAO con Hibernate

Ahora creamos las **implementaciones** en el paquete `com.mycompany.hibernate.dao.impl`. Aquí es donde escribimos el código real que interactúa con la base de datos.

### 7.2.1. `BarcoDAOImpl`

```java
package com.mycompany.hibernate.dao.impl;

import com.mycompany.hibernate.dao.BarcoDAO;
import com.mycompany.hibernate.modelo.Barco;
import com.mycompany.hibernate.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.Transaction;

import java.util.List;

public class BarcoDAOImpl implements BarcoDAO {

    // ════════════════════════════════════════
    // READ: operaciones de lectura (no necesitan transacción)
    // ════════════════════════════════════════

    @Override
    public Barco findById(Long id) {
        // Abre una sesión y busca el barco por su clave primaria
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.find(Barco.class, id);
        }
    }

    @Override
    public List<Barco> findAll() {
        // Abre una sesión y ejecuta una consulta HQL para obtener todos los barcos
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.createQuery("from Barco", Barco.class).getResultList();
        }
    }

    // ════════════════════════════════════════
    // WRITE: operaciones de escritura (NECESITAN transacción)
    // ════════════════════════════════════════

    @Override
    public Barco save(Barco barco) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();  // 1. Iniciar transacción
            session.save(barco);                                    // 2. Guardar
            transaction.commit();                                   // 3. Confirmar
            return barco;
        }
    }

    @Override
    public Barco update(Barco barco) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.update(barco);
            transaction.commit();
            return barco;
        }
    }

    @Override
    public void delete(Barco barco) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.delete(barco);
            transaction.commit();
        }
    }
}
```

Vamos a analizar los patrones clave de esta implementación:

### 7.3. Gestión de sesiones y transacciones

#### 7.3.1. Patrón `try-with-resources` para sesiones

```java
try (Session session = HibernateUtil.getSessionFactory().openSession()) {
    // ... operaciones con la sesión
}
// La sesión se cierra automáticamente al salir del bloque
```

El `try-with-resources` de Java garantiza que la sesión se **cierra automáticamente**, incluso si ocurre una excepción. Esto evita fugas de conexiones.

> 📖 **Vocabulario:** **try-with-resources** es una construcción de Java que cierra automáticamente los recursos (sesiones, conexiones, ficheros...) al salir del bloque `try`. El recurso debe implementar `AutoCloseable`.

#### 7.3.2. Patrón: beginTransaction → operación → commit

Las operaciones de **escritura** (INSERT, UPDATE, DELETE) necesitan una transacción:

```java
Transaction transaction = session.beginTransaction();  // 1. Iniciar
session.save(barco);                                    // 2. Operar
transaction.commit();                                   // 3. Confirmar
```

Las operaciones de **lectura** (SELECT) no necesitan transacción explícita.

> 📖 **Vocabulario:** Una **transacción** es una unidad de trabajo atómica: o todas las operaciones se completan con éxito (`commit`) o ninguna se aplica (`rollback`). Esto garantiza la **integridad de los datos**.

#### 7.3.3. Manejo de errores y `rollback()`

En un proyecto real, deberías manejar los errores con rollback:

```java
Transaction transaction = null;
try (Session session = HibernateUtil.getSessionFactory().openSession()) {
    transaction = session.beginTransaction();
    session.save(barco);
    transaction.commit();
} catch (Exception e) {
    if (transaction != null) {
        transaction.rollback();  // ← Deshacer todo si hay error
    }
    e.printStackTrace();
}
```

> ⚠️ **CUIDADO:** Si no haces `rollback()` cuando falla una operación, la base de datos puede quedar en un estado inconsistente.

> 💡 **TIP:** Por simplicidad, en este tutorial no usamos rollback en todos los métodos, pero en un proyecto real es **obligatorio**.

---

### 7.2.2. `AmarreDAOImpl` y `RegataDAOImpl`

Las implementaciones de `AmarreDAO` y `RegataDAO` siguen **exactamente el mismo patrón**, cambiando solo la entidad:

```java
package com.mycompany.hibernate.dao.impl;

import com.mycompany.hibernate.dao.AmarreDAO;
import com.mycompany.hibernate.modelo.Amarre;
import com.mycompany.hibernate.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.Transaction;
import java.util.List;

public class AmarreDAOImpl implements AmarreDAO {

    @Override
    public Amarre findById(Long id) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.find(Amarre.class, id);
        }
    }

    @Override
    public List<Amarre> findAll() {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.createQuery("from Amarre", Amarre.class).getResultList();
        }
    }

    @Override
    public Amarre save(Amarre amarre) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.save(amarre);
            transaction.commit();
            return amarre;
        }
    }

    @Override
    public Amarre update(Amarre amarre) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.update(amarre);
            transaction.commit();
            return amarre;
        }
    }

    @Override
    public void delete(Amarre amarre) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.delete(amarre);
            transaction.commit();
        }
    }
}
```

```java
package com.mycompany.hibernate.dao.impl;

import com.mycompany.hibernate.dao.RegataDAO;
import com.mycompany.hibernate.modelo.Regata;
import com.mycompany.hibernate.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.Transaction;
import java.util.List;

public class RegataDAOImpl implements RegataDAO {

    @Override
    public Regata findById(Long id) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.find(Regata.class, id);
        }
    }

    @Override
    public List<Regata> findAll() {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            return session.createQuery("from Regata", Regata.class).getResultList();
        }
    }

    @Override
    public Regata save(Regata regata) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.save(regata);
            transaction.commit();
            return regata;
        }
    }

    @Override
    public Regata update(Regata regata) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.update(regata);
            transaction.commit();
            return regata;
        }
    }

    @Override
    public void delete(Regata regata) {
        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();
            session.delete(regata);
            transaction.commit();
        }
    }
}
```

> 🔍 **¿Sabías que?** Si te fijas, el código de las tres implementaciones DAO es casi idéntico. Solo cambia la clase de la entidad. Esto es exactamente lo que resolvería un **DAO genérico** con `GenericDAO<T>`.

---

## 7.4. DAO genérico con `GenericDAO<T>`

Para evitar la repetición, podemos crear una interfaz y clase genérica:

### 7.4.1. Interfaz genérica

```java
package com.mycompany.hibernate.dao;

import java.util.List;

/**
 * Interfaz DAO genérica. La T representa cualquier tipo de entidad.
 */
public interface GenericDAO<T> {
    T findById(Long id);
    List<T> findAll();
    T save(T entity);
    T update(T entity);
    void delete(T entity);
}
```

### 7.4.2. Uso

Las interfaces específicas ahora pueden heredar de la genérica:

```java
public interface BarcoDAO extends GenericDAO<Barco> {
    // Aquí solo añades los métodos ESPECÍFICOS de Barco
    // Los CRUD ya los hereda de GenericDAO
}
```

---

## 7.5. Estructura del proyecto actualizada

```
src/main/java/com/mycompany/hibernate/
├── modelo/
│   ├── Barco.java          ← ✅ Cap. 4-5
│   ├── Amarre.java         ← ✅ Cap. 4-5
│   └── Regata.java         ← ✅ Cap. 5
├── dao/
│   ├── GenericDAO.java     ← ✅ Este capítulo
│   ├── BarcoDAO.java       ← ✅ Este capítulo
│   ├── AmarreDAO.java      ← ✅ Este capítulo
│   └── RegataDAO.java      ← ✅ Este capítulo
├── dao/impl/
│   ├── BarcoDAOImpl.java   ← ✅ Este capítulo
│   ├── AmarreDAOImpl.java  ← ✅ Este capítulo
│   └── RegataDAOImpl.java  ← ✅ Este capítulo
├── util/
│   └── HibernateUtil.java  ← ✅ Cap. 3
└── Main.java               ← ✅ Cap. 6
```

---

## 📝 Glosario del Capítulo 7

| Término | Definición |
|---------|------------|
| **Interfaz DAO** | Contrato que define las operaciones disponibles sobre una entidad |
| **Implementación DAO** | Clase que proporciona el código real para las operaciones definidas en la interfaz |
| **`try-with-resources`** | Construcción Java que cierra automáticamente los recursos al finalizar |
| **Transacción** | Unidad de trabajo atómica: todo se aplica (commit) o nada se aplica (rollback) |
| **`commit()`** | Confirma los cambios de una transacción y los aplica a la BD |
| **`rollback()`** | Deshace todos los cambios de la transacción actual |
| **`session.find()`** | Busca una entidad por su clave primaria, devuelve `null` si no existe |
| **`session.createQuery()`** | Crea una consulta HQL para la sesión actual |
| **`GenericDAO<T>`** | Interfaz DAO genérica reutilizable para cualquier tipo de entidad |
| **`implements`** | Palabra clave Java que indica que una clase proporciona la implementación de una interfaz |

---

✅ **Checkpoint:** Al terminar este capítulo debes tener...
- Las interfaces `BarcoDAO`, `AmarreDAO`, `RegataDAO` creadas.
- Las implementaciones `BarcoDAOImpl`, `AmarreDAOImpl`, `RegataDAOImpl` funcionales.
- Entender el patrón `try-with-resources` para sesiones.
- Entender el flujo `beginTransaction → operación → commit`.
- La estructura de carpetas `dao/` y `dao/impl/` organizada.

---

*Capítulo anterior: [← Capítulo 6. Estados y Ciclo de Vida](Cap06_Estados.md)*

*Siguiente capítulo: [Capítulo 8. Consultas Avanzadas →](Cap08_Consultas.md)*
