# Capítulo 12. Capa de Servicio

---

🎯 **Objetivo del capítulo:** Entender por qué necesitamos una capa de servicio entre los controladores y los repositorios, y cómo implementarla con `@Service` y `@Transactional`.

---

## 12.1. ¿Por qué no usar el repositorio directamente?

Podrías pensar: "Si `BarcoRepository` ya tiene `findAll()` y `save()`, ¿por qué no lo uso directamente desde el controlador?"

```java
// ❌ MAL: Controlador accediendo al repositorio directamente
@RestController
public class BarcoController {
    @Autowired
    private BarcoRepository barcoRepository; // ← Acoplamiento directo

    @GetMapping("/api/barcos")
    public List<Barco> getAll() {
        return barcoRepository.findAll(); // ← Devuelve la entidad JPA directamente
    }
}
```

**Problemas de este enfoque:**

1. **Sin lógica de negocio**: ¿dónde validamos que un barco no se duplique?
2. **Sin conversión DTO**: exponemos las entidades JPA directamente (con sus relaciones circulares)
3. **Sin gestión transaccional**: operaciones complejas no son atómicas
4. **Acoplamiento**: si cambias la BD, tienes que cambiar todos los controladores
5. **Imposible testear**: no puedes hacer un mock limpio

### Arquitectura en capas (la correcta):

```
    Petición HTTP
         │
    ┌────▼────┐
    │Controller│  ← Recibe peticiones, devuelve respuestas
    └────┬────┘     Solo maneja la "vista" (JSON o HTML)
         │
    ┌────▼────┐
    │ Service  │  ← Lógica de negocio + Transacciones
    └────┬────┘     Convierte Entidad ↔ DTO
         │
    ┌────▼────┐
    │Repository│  ← Acceso a datos (CRUD automático)
    └────┬────┘     Habla con la BD a través de JPA
         │
    ┌────▼────┐
    │   BD    │  ← MySQL (Docker)
    └─────────┘
```

> 📖 **Vocabulario:**
> - **Capa de servicio (Service Layer)**: capa intermedia que contiene la lógica de negocio. Traduce entre lo que el controlador necesita y lo que el repositorio ofrece.
> - **Separación de responsabilidades (SoC)**: cada capa tiene UNA sola responsabilidad. El controlador NO toca la BD. El repositorio NO sabe de HTTP.

---

## 12.2. Crear el servicio con `@Service`

```java
package com.example.marina.service;

import com.example.marina.dto.BarcoDTO;
import com.example.marina.mapper.BarcoMapper;
import com.example.marina.model.Barco;
import com.example.marina.repository.BarcoRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.stream.Collectors;

@Service   // ← Spring detecta esta clase y crea un "bean" de servicio
public class BarcoService {

    @Autowired   // ← Inyección de dependencias: Spring inyecta el repositorio
    private BarcoRepository barcoRepository;
}
```

### ¿Qué hace `@Service`?

1. Marca la clase como un **bean de Spring** (componente gestionado)
2. Spring **crea una instancia** automáticamente al arrancar
3. Cualquier clase puede **inyectarla** con `@Autowired`
4. Semánticamente indica que contiene **lógica de negocio**

> 💡 **¿Sabías que?** `@Service` es en realidad un alias de `@Component`. Técnicamente hacen lo mismo, pero se usan diferentes nombres para indicar el **propósito**: `@Service` para lógica de negocio, `@Repository` para acceso a datos, `@Controller` para controladores web.

---

## 12.3. Operaciones CRUD del servicio

### Listar todos (con conversión a DTO):
```java
public List<BarcoDTO> findAll() {
    return barcoRepository.findAll()       // 1. Obtiene List<Barco> del repositorio
            .stream()                       // 2. Convierte a Stream (flujo de datos)
            .map(BarcoMapper::toDTO)        // 3. Transforma cada Barco → BarcoDTO
            .collect(Collectors.toList());  // 4. Recoge todo en una nueva lista
}
```

> 📖 **Vocabulario:**
> - **Stream**: flujo de datos que permite transformar colecciones de forma funcional (sin bucles `for`).
> - **`.map()`**: aplica una función a cada elemento del Stream.
> - **Method reference** (`BarcoMapper::toDTO`): referencia a un método estático. Equivale a `barco -> BarcoMapper.toDTO(barco)`.

### Buscar por ID:
```java
public BarcoDTO findById(Long id) {
    return barcoRepository.findById(id)   // Devuelve Optional<Barco>
            .map(BarcoMapper::toDTO)       // Si existe → convierte a DTO
            .orElse(null);                 // Si no existe → devuelve null
}
```

> 📖 **Vocabulario:**
> - **`Optional<T>`**: contenedor que puede o no tener un valor. Evita `NullPointerException`. Es como una "caja" que puede estar vacía o contener un objeto.

### Guardar (crear nuevo):
```java
@Transactional
public BarcoDTO save(BarcoDTO dto) {
    Barco barco = BarcoMapper.toEntity(dto);   // Convertir DTO → Entidad
    Barco saved = barcoRepository.save(barco);  // Persistir en la BD
    return BarcoMapper.toDTO(saved);             // Devolver DTO (con ID generado)
}
```

### Actualizar existente:
```java
@Transactional
public BarcoDTO update(Long id, BarcoDTO dto) {
    return barcoRepository.findById(id)
            .map(barco -> {
                BarcoMapper.updateEntityFromDTO(dto, barco);  // Actualizar campos
                Barco updated = barcoRepository.save(barco);   // Guardar cambios
                return BarcoMapper.toDTO(updated);
            })
            .orElse(null);   // Si no existe, devuelve null
}
```

### Eliminar:
```java
@Transactional
public void deleteById(Long id) {
    barcoRepository.deleteById(id);
}
```

---

## 12.4. `@Transactional` — Gestión automática

Comparamos cómo se gestionan las transacciones:

### Hibernate Puro (manual):
```java
public Barco save(Barco barco) {
    Transaction tx = null;
    try (Session session = HibernateUtil.getSessionFactory().openSession()) {
        tx = session.beginTransaction();       // 1. Abrir transacción
        session.save(barco);                    // 2. Guardar
        tx.commit();                            // 3. Confirmar
        return barco;
    } catch (Exception e) {
        if (tx != null) tx.rollback();          // 4. Si falla, deshacer
        throw new RuntimeException(e);
    }
}
```

### Spring Boot (automático):
```java
@Transactional   // ← Spring hace todo: begin, commit, rollback
public BarcoDTO save(BarcoDTO dto) {
    Barco barco = BarcoMapper.toEntity(dto);
    return BarcoMapper.toDTO(barcoRepository.save(barco));
}
```

### ¿Qué hace `@Transactional` exactamente?

```
1. Spring abre una transacción automáticamente (BEGIN)
        │
2. Ejecuta el método del servicio
        │
3a. Si el método termina bien → COMMIT (confirma los cambios)
3b. Si lanza una excepción → ROLLBACK (deshace todos los cambios)
```

> ⚠️ **CUIDADO:** `@Transactional` solo funciona sobre métodos **públicos** llamados desde **fuera de la clase**. Si un método privado llama a otro método de la misma clase, la transacción NO se aplica al segundo.

---

## 12.5. Inyección de dependencias (`@Autowired`)

### ¿Qué es la inyección de dependencias?

Es el mecanismo por el que Spring **proporciona automáticamente** las dependencias que una clase necesita.

```java
@Service
public class BarcoService {

    @Autowired   // ← Spring busca un bean de tipo BarcoRepository y lo inyecta aquí
    private BarcoRepository barcoRepository;
}
```

### Alternativa recomendada: inyección por constructor

```java
@Service
public class BarcoService {

    private final BarcoRepository barcoRepository;

    // Spring inyecta el repositorio a través del constructor
    public BarcoService(BarcoRepository barcoRepository) {
        this.barcoRepository = barcoRepository;
    }
}
```

> 🏆 **Buena práctica:** La inyección por constructor es **preferible** a `@Autowired` porque:
> 1. Los campos pueden ser `final` (inmutables)
> 2. Es más fácil de testear (le pasas mocks al constructor)
> 3. El compilador te avisa si falta una dependencia

---

## 12.6. Métodos para Thymeleaf vs REST

El servicio tiene dos "versiones" de algunos métodos:

```java
// Para la API REST: devuelve DTOs
public List<BarcoDTO> findAll() { ... }
public BarcoDTO findById(Long id) { ... }

// Para Thymeleaf: devuelve entidades directamente
public List<Barco> findAllEntities() { return barcoRepository.findAll(); }
public Barco findEntityById(Long id) { return barcoRepository.findById(id).orElse(null); }
```

> 💡 **TIP:** En un proyecto real, lo ideal sería que **todo** usara DTOs. Pero para un proyecto educativo, a veces es más claro pasar la entidad directamente a la vista Thymeleaf.

---

## 12.7. Servicios completos del proyecto

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/service/BarcoService.java`

💡 *Ejemplo didáctico — Este servicio ya se explicó en las secciones 12.2-12.6 arriba.*

```java
package com.example.marina.service;

import com.example.marina.dto.BarcoDTO;
import com.example.marina.mapper.BarcoMapper;
import com.example.marina.model.Barco;
import com.example.marina.repository.BarcoRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class BarcoService {

    @Autowired
    private BarcoRepository barcoRepository;

    // ── Para API REST (con DTOs) ──
    public List<BarcoDTO> findAll() {
        return barcoRepository.findAll().stream()
                .map(BarcoMapper::toDTO).collect(Collectors.toList());
    }

    public BarcoDTO findById(Long id) {
        return barcoRepository.findById(id)
                .map(BarcoMapper::toDTO).orElse(null);
    }

    @Transactional
    public BarcoDTO save(BarcoDTO dto) {
        Barco barco = BarcoMapper.toEntity(dto);
        return BarcoMapper.toDTO(barcoRepository.save(barco));
    }

    @Transactional
    public BarcoDTO update(Long id, BarcoDTO dto) {
        return barcoRepository.findById(id).map(barco -> {
            BarcoMapper.updateEntityFromDTO(dto, barco);
            return BarcoMapper.toDTO(barcoRepository.save(barco));
        }).orElse(null);
    }

    @Transactional
    public void deleteById(Long id) { barcoRepository.deleteById(id); }

    // ── Para Thymeleaf (con entidades) ──
    public List<Barco> findAllEntities() { return barcoRepository.findAll(); }
    public Barco findEntityById(Long id) { return barcoRepository.findById(id).orElse(null); }
    @Transactional
    public Barco saveEntity(Barco barco) { return barcoRepository.save(barco); }

    // ── Consultas personalizadas ──
    public List<BarcoDTO> findByTipo(String tipo) {
        return barcoRepository.findByTipo(tipo).stream()
                .map(BarcoMapper::toDTO).collect(Collectors.toList());
    }

    public Long countByTipo(String tipo) { return barcoRepository.countByTipo(tipo); }
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/service/AmarreService.java`

```java
package com.example.marina.service;

import com.example.marina.model.Amarre;
import com.example.marina.repository.AmarreRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class AmarreService {

    @Autowired
    private AmarreRepository amarreRepository;

    public List<Amarre> findAll() {
        return amarreRepository.findAll();
    }

    public Amarre findById(Long id) {
        return amarreRepository.findById(id).orElse(null);
    }

    @Transactional
    public Amarre save(Amarre amarre) {
        return amarreRepository.save(amarre);
    }

    @Transactional
    public void deleteById(Long id) {
        amarreRepository.deleteById(id);
    }

    public List<Amarre> findLibres() {
        return amarreRepository.findByBarcoIsNull();
    }

    public List<Amarre> findConElectricidad() {
        return amarreRepository.findByElectricidad(true);
    }
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/service/RegataService.java`

```java
package com.example.marina.service;

import com.example.marina.model.Barco;
import com.example.marina.model.Regata;
import com.example.marina.repository.BarcoRepository;
import com.example.marina.repository.RegataRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class RegataService {

    @Autowired
    private RegataRepository regataRepository;

    @Autowired
    private BarcoRepository barcoRepository;

    public List<Regata> findAll() {
        return regataRepository.findAll();
    }

    public Regata findById(Long id) {
        return regataRepository.findById(id).orElse(null);
    }

    @Transactional
    public Regata save(Regata regata) {
        return regataRepository.save(regata);
    }

    @Transactional
    public void deleteById(Long id) {
        regataRepository.deleteById(id);
    }

    public List<Regata> findByLugar(String lugar) {
        return regataRepository.findByLugar(lugar);
    }

    /**
     * Inscribe un barco en una regata (relación N:M).
     * ¡IMPORTANTE! Barco es el lado OWNING del @ManyToMany
     * (tiene @JoinTable), así que debemos modificar barco.regatas.
     */
    @Transactional
    public void inscribirBarco(Long regataId, Long barcoId) {
        Regata regata = regataRepository.findById(regataId).orElse(null);
        Barco barco = barcoRepository.findById(barcoId).orElse(null);
        if (regata != null && barco != null) {
            // Modificar el lado OWNING (Barco)
            barco.getRegatas().add(regata);
            barcoRepository.save(barco);
        }
    }

    /**
     * Quita un barco de una regata.
     * Usamos iteración por ID para evitar problemas con
     * equals/hashCode de Lombok en entidades con colecciones lazy.
     */
    @Transactional
    public void desinscribirBarco(Long regataId, Long barcoId) {
        Barco barco = barcoRepository.findById(barcoId).orElse(null);
        if (barco != null) {
            List<Regata> regatas = barco.getRegatas();
            regatas.removeIf(r -> r.getId() != null && r.getId().equals(regataId));
            barcoRepository.save(barco);
        }
    }

    /**
     * Elimina una regata limpiando primero sus inscripciones N:M.
     */
    @Transactional
    public void deleteWithCleanup(Long id) {
        Regata regata = regataRepository.findById(id).orElse(null);
        if (regata != null) {
            // Quitar la regata de todos los barcos (lado owning)
            for (Barco barco : regata.getBarcos()) {
                barco.getRegatas().remove(regata);
                barcoRepository.save(barco);
            }
            regataRepository.deleteById(id);
        }
    }
}
```

> 💡 **TIP:** Fíjate que `RegataService` necesita AMBOS repositorios: `RegataRepository`
> para gestionar las regatas, y `BarcoRepository` para gestionar las inscripciones N:M
> (porque `Barco` es el lado **owning** de la relación `@ManyToMany`).

---

## 12.8. Introducción a AOP (Programación Orientada a Aspectos)

### ¿Qué es un aspecto?

Imagina que quieres **registrar en log** cada vez que se llama a cualquier método de cualquier servicio. Una opción sería añadir `System.out.println()` al principio de cada método... pero eso es repetitivo y ensucia el código.

**AOP** permite definir ese comportamiento **una sola vez** y aplicarlo automáticamente a todos los métodos que quieras:

```
  SIN AOP                              CON AOP
  ┌──────────────────────────┐         ┌──────────────────────────┐
  │ BarcoService.findAll()   │         │ BarcoService.findAll()   │
  │   log("entrando...")     │         │   // Solo lógica         │
  │   // lógica              │         │   // de negocio          │
  │   log("saliendo...")     │         └──────────────────────────┘
  ├──────────────────────────┤                     ↑
  │ AmarreService.findAll()  │         ┌───────────┴──────────────┐
  │   log("entrando...")     │         │   LoggingAspect           │
  │   // lógica              │         │   @Before: log("→ ...")   │
  │   log("saliendo...")     │         │   @After:  log("← ...")   │
  └──────────────────────────┘         └──────────────────────────┘
```

> 📖 **Vocabulario:**
> - **Aspecto** (*aspect*): módulo que encapsula un comportamiento transversal (logging, seguridad, métricas).
> - **Cross-cutting concern**: funcionalidad que afecta a múltiples capas (ej: logging en todos los servicios).

### Ejemplo: Aspecto de logging

```java
@Aspect
@Component
public class LoggingAspect {

    // Se ejecuta ANTES de cualquier método en el paquete service
    @Before("execution(* com.example.marina.service.*.*(..))")
    public void logAntes(JoinPoint joinPoint) {
        System.out.println("→ Llamando a: " + joinPoint.getSignature().getName());
    }

    // Se ejecuta DESPUÉS de cualquier método en el paquete service
    @After("execution(* com.example.marina.service.*.*(..))")
    public void logDespues(JoinPoint joinPoint) {
        System.out.println("← Terminó: " + joinPoint.getSignature().getName());
    }
}
```

### Tipos de advice (cuándo actúa el aspecto)

| Advice | Cuándo se ejecuta | Ejemplo de uso |
|--------|-------------------|----------------|
| `@Before` | **Antes** del método | Logging, validación |
| `@After` | **Después** del método (siempre) | Liberar recursos |
| `@AfterReturning` | Después, si **no** hubo excepción | Auditoría |
| `@AfterThrowing` | Después, si **hubo** excepción | Manejo de errores |
| `@Around` | **Envuelve** el método (antes + después) | Medir tiempo de ejecución |

### ¿Dónde ya usamos AOP sin saberlo?

```
@Transactional  ← Spring usa un aspecto para gestionar transacciones
@Autowired      ← Spring usa AOP para inyectar dependencias en runtime
```

¡`@Transactional` del apartado 12.4 es un aspecto! Spring lo implementa con un **proxy** que envuelve tu servicio y añade `begin()`, `commit()` o `rollback()` automáticamente.

> 💡 **TIP:** No necesitas crear aspectos personalizados en este proyecto, pero es importante entender que `@Transactional` y la inyección de dependencias funcionan gracias a AOP internamente.

---

## 📝 Glosario del Capítulo 12

| Término | Definición |
|---------|------------|
| **`@Service`** | Anotación que marca una clase como servicio de lógica de negocio |
| **`@Transactional`** | Gestiona transacciones automáticamente (begin/commit/rollback) |
| **`@Autowired`** | Inyección automática de dependencias por Spring |
| **Inyección de dependencias** | Patrón donde las dependencias se "inyectan" desde el exterior |
| **IoC (Inversión de Control)** | El framework controla el ciclo de vida de los objetos, no tú |
| **Stream** | Flujo funcional de datos para transformar colecciones |
| **Optional** | Contenedor que puede o no contener un valor (evita null) |
| **Method Reference** | `Clase::metodo` — referencia a un método existente |
| **SoC** | Separation of Concerns — cada capa tiene una sola responsabilidad |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Entender por qué el controlador NO debe acceder al repositorio directamente
- Saber crear un servicio con `@Service`
- Usar `@Transactional` para gestionar transacciones
- Usar Streams y Optional en tus métodos de servicio
- Entender la inyección de dependencias con `@Autowired`

---

*Capítulo anterior: [← Capítulo 11. Repositorios](Cap11_Repositorios.md)*

*Siguiente capítulo: [Capítulo 13. API REST →](Cap13_REST.md)*
