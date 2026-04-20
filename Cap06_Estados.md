# Capítulo 6. Estados y Ciclo de Vida de las Entidades

---

🎯 **Objetivo del capítulo:** Entender los cuatro estados en los que puede encontrarse una entidad en Hibernate (Transient, Persistent, Detached, Removed), cómo transitar entre ellos, y probar la persistencia completa del proyecto en `Main.java`.

---

## 6.1. ¿Por qué importan los estados?

Cuando trabajas con Hibernate, los objetos Java no se comportan como objetos normales. Tienen un **ciclo de vida** gestionado por Hibernate. Dependiendo del estado en que se encuentre un objeto, Hibernate se comportará de manera diferente:

- ¿El objeto está **sincronizado** con la base de datos?
- ¿Hibernate **rastrea** los cambios que le haces?
- ¿Los cambios se **guardan automáticamente**?

Entender estos estados es fundamental para evitar errores y trabajar eficientemente con el framework.

> 📖 **Vocabulario:** El **ciclo de vida** de una entidad describe los diferentes estados por los que pasa un objeto Java desde que se crea con `new` hasta que se elimina de la base de datos. Hibernate gestiona este ciclo de vida a través de la **sesión** (Session).

---

## 6.2. Estado Transitorio (Transient)

### 6.2.1. Definición y características

Un objeto está en estado **Transitorio** cuando acaba de ser creado con el operador `new` pero **aún no está asociado** a ninguna sesión de Hibernate.

```java
// Este objeto está en estado TRANSIENT
Barco barco = new Barco();
barco.setNombre("Titanic");
barco.setTipo("Pasajeros");
barco.setEslora(269);

// Hibernate NO sabe que este objeto existe
// NO tiene representación en la base de datos
// NO tiene ID asignado (es null)
```

**Características:**
- ❌ No tiene ID (es `null`).
- ❌ No tiene representación en la base de datos.
- ❌ Hibernate **no lo rastrea** — si cambias algo, no pasa nada en la BD.
- ❌ Si el objeto sale del ámbito (scope), se pierde para siempre.

> 💡 **TIP:** Piensa en un objeto Transient como un objeto Java "normal". Solo existe en memoria y Hibernate no sabe nada de él.

---

## 6.3. Estado Persistente (Persistent)

### 6.3.1. Definición y características

Un objeto pasa a estado **Persistente** cuando se asocia con una sesión de Hibernate, ya sea porque lo has guardado con `save()` o porque lo has recuperado de la base de datos con `find()`.

```java
Session session = HibernateUtil.getSessionFactory().openSession();
Transaction tx = session.beginTransaction();

Barco barco = new Barco();        // TRANSIENT
barco.setNombre("Santa María");

session.save(barco);              // TRANSIENT → PERSISTENT
// Ahora Hibernate rastrea este objeto
// Tiene un ID asignado
// Cualquier cambio se sincronizará con la BD

barco.setEslora(25);              // Este cambio se guarda automáticamente

tx.commit();
session.close();
```

**Características:**
- ✅ Tiene un **ID asignado** por la base de datos.
- ✅ Está **asociado a una sesión** activa de Hibernate.
- ✅ Hibernate **rastrea los cambios** (*dirty checking*): si modificas un atributo, Hibernate genera un `UPDATE` automáticamente al hacer `commit()`.
- ✅ Su estado en memoria y en la BD está **sincronizado**.

### 6.3.2. Sincronización automática (dirty checking)

Esta es la característica más importante del estado Persistente: Hibernate **detecta automáticamente** si has cambiado algo en el objeto y genera la sentencia SQL necesaria.

```java
Session session = HibernateUtil.getSessionFactory().openSession();
Transaction tx = session.beginTransaction();

// Recuperar un barco de la BD (estado PERSISTENT)
Barco barco = session.find(Barco.class, 1L);
barco.setNombre("Nuevo nombre");   // Hibernate detecta este cambio

tx.commit();  // Hibernate genera: UPDATE barco SET nombre='Nuevo nombre' WHERE id=1
session.close();
```

> 📖 **Vocabulario:** **Dirty checking** (comprobación de cambios) es el mecanismo por el cual Hibernate compara el estado actual de un objeto persistente con su estado original y genera automáticamente las sentencias SQL necesarias para sincronizar los cambios.

> 🔍 **¿Sabías que?** No necesitas llamar a `session.update()` para un objeto que ya es persistente. Hibernate detecta los cambios automáticamente. El `update()` se usa principalmente para re-atachar objetos desconectados (Detached).

---

## 6.4. Estado Desconectado (Detached)

### 6.4.1. Definición y características

Un objeto pasa a estado **Desconectado** cuando estaba Persistente pero la sesión que lo gestionaba **se ha cerrado**. El objeto sigue existiendo en memoria con su ID, pero Hibernate ya no lo gestiona.

```java
Session session = HibernateUtil.getSessionFactory().openSession();
Barco barco = session.find(Barco.class, 1L);   // PERSISTENT
session.close();                                 // PERSISTENT → DETACHED

// El objeto sigue en memoria con id=1 y todos sus datos
// Pero Hibernate ya NO lo rastrea
barco.setNombre("Cambio invisible");  // Este cambio NO se guarda en la BD
```

**Características:**
- ✅ Tiene un **ID** (representa un registro real en la BD).
- ❌ **No está asociado** a ninguna sesión activa.
- ❌ Los cambios **NO se sincronizan** automáticamente con la BD.
- ✅ Puede **volver a ser Persistente** si se re-atacha a una nueva sesión.

### 6.4.2. Cómo re-atachar un objeto desconectado

```java
// Objeto desconectado con cambios
barco.setNombre("Nombre actualizado");

// Abrir nueva sesión y re-atachar
Session session2 = HibernateUtil.getSessionFactory().openSession();
Transaction tx = session2.beginTransaction();
session2.update(barco);    // DETACHED → PERSISTENT (los cambios se sincronizarán)
tx.commit();
session2.close();
```

---

## 6.5. Estado Eliminado (Removed)

Un objeto pasa a estado **Eliminado** cuando se ha llamado a `session.delete()` sobre un objeto Persistente. El objeto se marcará para eliminación de la BD al hacer `commit()`.

```java
Session session = HibernateUtil.getSessionFactory().openSession();
Transaction tx = session.beginTransaction();

Barco barco = session.find(Barco.class, 1L);   // PERSISTENT
session.delete(barco);                           // PERSISTENT → REMOVED

tx.commit();  // Hibernate genera: DELETE FROM barco WHERE id=1
session.close();
```

---

## 6.6. Transiciones entre estados

### Tabla resumen

| Transición | Métodos | SQL generado |
|-----------|---------|-------------|
| **Transient → Persistent** | `session.save(obj)`, `session.persist(obj)`, `session.saveOrUpdate(obj)` | `INSERT INTO ...` |
| **Persistent → Detached** | `session.close()`, `session.evict(obj)`, `session.clear()` | Ninguno |
| **Detached → Persistent** | `session.update(obj)`, `session.merge(obj)`, `session.saveOrUpdate(obj)` | `UPDATE ...` (al commit) |
| **Persistent → Removed** | `session.delete(obj)` | `DELETE FROM ...` |

### 6.6.1. Métodos de Transient → Persistent

| Método | Comportamiento |
|--------|---------------|
| `save(obj)` | Guarda el objeto y devuelve el ID generado inmediatamente |
| `persist(obj)` | Similar a `save` pero no garantiza la generación inmediata del ID |
| `saveOrUpdate(obj)` | Inteligente: si es Transient hace `INSERT`, si es Detached hace `UPDATE` |

### 6.6.2. Métodos de Persistent → Detached

| Método | Comportamiento |
|--------|---------------|
| `session.close()` | Cierra la sesión → todos los objetos pasan a Detached |
| `session.evict(obj)` | Desconecta **un objeto específico** de la sesión |
| `session.clear()` | Desconecta **todos los objetos** de la sesión |

### 6.6.3. Métodos de Detached → Persistent

| Método | Comportamiento |
|--------|---------------|
| `update(obj)` | Re-atacha el objeto a la sesión |
| `merge(obj)` | Copia los cambios del objeto Detached al Persistente (si existe uno con el mismo ID) |
| `saveOrUpdate(obj)` | `update` si tiene ID, `save` si no lo tiene |

> 💡 **TIP:** La diferencia entre `update()` y `merge()` es sutil: `update()` re-atacha el propio objeto, mientras que `merge()` crea una copia persistente y devuelve esa copia. Si ya hay un objeto persistente con el mismo ID en la sesión, `update()` lanza error y `merge()` no.

### 6.6.4. `delete()` — Persistent → Removed

```java
session.delete(obj);  // Marca el objeto para eliminación. Se ejecuta al commit()
```

---

## 6.7. Diagrama de transiciones de estado

```
                    new Barco()
                        │
                        ▼
               ┌─────────────────┐
               │   TRANSIENT     │
               │  (sin ID, sin   │
               │   sesión)       │
               └────────┬────────┘
                        │ save() / persist()
                        ▼
               ┌─────────────────┐        delete()       ┌──────────────┐
               │   PERSISTENT    │ ─────────────────────▶ │   REMOVED    │
               │  (con ID, con   │                        │  (marcado    │
               │   sesión activa)│                        │   para borrar│
               └────────┬────────┘                        └──────────────┘
                        │ close() / evict() / clear()
                        ▼
               ┌─────────────────┐
               │   DETACHED      │
               │  (con ID, sin   │
               │   sesión)       │
               └────────┬────────┘
                        │ update() / merge()
                        │ (vuelve a Persistent)
                        └────────▲
```

---

## 6.8. Práctica: `Main.java` — persistir Barcos, Amarres y Regatas

Ahora que entendemos los estados, vamos a escribir un `Main.java` completo que cree y persista datos del proyecto:

```java
package com.mycompany.hibernate;

import com.mycompany.hibernate.modelo.Amarre;
import com.mycompany.hibernate.modelo.Barco;
import com.mycompany.hibernate.modelo.Regata;
import com.mycompany.hibernate.util.HibernateUtil;
import org.hibernate.Session;
import org.hibernate.Transaction;

import java.util.ArrayList;
import java.util.Date;

public class Main {
    public static void main(String[] args) {

        // ════════════════════════════════════════════
        // 1. CREAR OBJETOS (estado TRANSIENT)
        // ════════════════════════════════════════════

        // Crear barcos
        Barco barco1 = new Barco();
        barco1.setNombre("Estrella del Mar");
        barco1.setTipo("Velero");
        barco1.setEslora(12);
        barco1.setManga(4);
        barco1.setCapacidad(8);

        Barco barco2 = new Barco();
        barco2.setNombre("Rayo Azul");
        barco2.setTipo("Motor");
        barco2.setEslora(15);
        barco2.setManga(5);
        barco2.setCapacidad(10);

        // Crear amarres
        Amarre amarre1 = new Amarre();
        amarre1.setUbicacion("A-1");
        amarre1.setPrecio(1500.0);
        amarre1.setProfundidad(5);
        amarre1.setLongitud(14);
        amarre1.setElectricidad(true);

        Amarre amarre2 = new Amarre();
        amarre2.setUbicacion("B-3");
        amarre2.setPrecio(2000.0);
        amarre2.setProfundidad(7);
        amarre2.setLongitud(18);
        amarre2.setElectricidad(true);

        // Crear regata
        Regata regata = new Regata();
        regata.setNombre("Copa Mediterráneo");
        regata.setLugar("Mar Mediterráneo");
        regata.setFecha(new Date());
        regata.setDistancia(100);

        // ════════════════════════════════════════════
        // 2. ESTABLECER RELACIONES (todavía TRANSIENT)
        // ════════════════════════════════════════════

        // Relación 1:1 Barco ↔ Amarre (bidireccional)
        barco1.setAmarre(amarre1);
        amarre1.setBarco(barco1);

        barco2.setAmarre(amarre2);
        amarre2.setBarco(barco2);

        // Relación N:M Barco ↔ Regata (bidireccional)
        barco1.setRegatas(new ArrayList<>());
        barco2.setRegatas(new ArrayList<>());
        regata.setBarcos(new ArrayList<>());

        barco1.getRegatas().add(regata);
        barco2.getRegatas().add(regata);
        regata.getBarcos().add(barco1);
        regata.getBarcos().add(barco2);

        // ════════════════════════════════════════════
        // 3. PERSISTIR (TRANSIENT → PERSISTENT)
        // ════════════════════════════════════════════

        try (Session session = HibernateUtil.getSessionFactory().openSession()) {
            Transaction transaction = session.beginTransaction();

            // Al guardar barco1, gracias a cascade=ALL,
            // también se guarda amarre1 automáticamente
            session.save(barco1);
            session.save(barco2);
            session.save(regata);

            transaction.commit();

            System.out.println("✅ Datos guardados correctamente.");
            System.out.println("   Barco 1 ID: " + barco1.getId());
            System.out.println("   Barco 2 ID: " + barco2.getId());
            System.out.println("   Regata ID: " + regata.getId());
        }
        // Al cerrar la sesión: PERSISTENT → DETACHED

        System.out.println("✅ Sesión cerrada. Los objetos ahora están DETACHED.");
    }
}
```

> ⚠️ **CUIDADO:** Es importante establecer **ambos lados** de una relación bidireccional. Si solo haces `barco1.setAmarre(amarre1)` pero no `amarre1.setBarco(barco1)`, puede haber inconsistencias.

> 🏆 **Buena práctica:** Inicializa las listas de relaciones N:M con `new ArrayList<>()` antes de añadir elementos. Si no, obtendrás un `NullPointerException`.

---

## 📝 Glosario del Capítulo 6

| Término | Definición |
|---------|------------|
| **Transient** | Estado de un objeto recién creado con `new`, sin ID y sin asociación a sesión |
| **Persistent** | Estado de un objeto gestionado por una sesión activa, sincronizado con la BD |
| **Detached** | Estado de un objeto que fue persistente pero cuya sesión se cerró |
| **Removed** | Estado de un objeto marcado para eliminación de la BD |
| **Dirty checking** | Mecanismo que detecta cambios en objetos persistentes y genera SQL automáticamente |
| **`save()`** | Guarda un objeto Transient → lo hace Persistente, genera INSERT |
| **`persist()`** | Similar a save, no garantiza generación inmediata de ID |
| **`update()`** | Re-atacha un objeto Detached → lo hace Persistente de nuevo |
| **`merge()`** | Copia los cambios de un objeto Detached al Persistente con el mismo ID |
| **`delete()`** | Marca un objeto Persistente para eliminación (Removed) |
| **`evict()`** | Desconecta un objeto específico de la sesión (Persistent → Detached) |
| **`clear()`** | Desconecta todos los objetos de la sesión |
| **Transacción** | Unidad de trabajo atómica: o se completa todo (commit) o no se aplica nada (rollback) |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Conocer los 4 estados de una entidad: Transient, Persistent, Detached, Removed.
- Saber qué método usar para cada transición.
- Entender el *dirty checking*: cambios automáticos en objetos persistentes.
- Haber ejecutado `Main.java` con datos de barcos, amarres y regatas persistidos en MySQL.
- Poder verificar los datos en MySQL: `SELECT * FROM barco; SELECT * FROM amarre; SELECT * FROM regata; SELECT * FROM barco_regata;`

---

*Capítulo anterior: [← Capítulo 5. Relaciones entre Entidades](Cap05_Relaciones.md)*

*Siguiente capítulo: [Capítulo 7. Implementación del Patrón DAO →](Cap07_DAO.md)*
