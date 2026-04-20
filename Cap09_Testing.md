# Capítulo 9. Pruebas Unitarias

---

🎯 **Objetivo del capítulo:** Aprender a escribir pruebas unitarias para los DAOs usando JUnit 5 y Mockito. Testear el código sin necesidad de una base de datos real, simulando (mockeando) las sesiones de Hibernate.

---

## 9.1. ¿Por qué son importantes las pruebas unitarias?

Las pruebas unitarias verifican que **cada pieza individual** de tu código funciona correctamente de forma aislada. Son la primera línea de defensa contra los bugs.

Sin tests, cada vez que cambias algo en un DAO tienes que arrancar la aplicación, abrir MySQL, insertar datos manualmente y comprobar los resultados. Con tests, ejecutas un comando y en segundos sabes si todo funciona.

> 📖 **Vocabulario:** Una **prueba unitaria** (*unit test*) es un test automatizado que verifica el comportamiento de una unidad pequeña de código (normalmente un método) de forma aislada.

---

## 9.2. Introducción a JUnit 5

JUnit 5 es el framework estándar para escribir y ejecutar pruebas en Java.

### 9.2.1. Dependencia Maven

Ya la tenemos en nuestro `pom.xml` (Capítulo 3):

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.9.2</version>
    <scope>test</scope>
</dependency>
```

### 9.2.2. Anotaciones principales

| Anotación | Cuándo se ejecuta | Para qué |
|-----------|-------------------|----------|
| `@Test` | Marca un método como prueba | Es la base de todo test |
| `@BeforeEach` | **Antes** de cada test | Preparar datos, inicializar mocks |
| `@AfterEach` | **Después** de cada test | Limpiar recursos |
| `@BeforeAll` | Una vez antes de **todos** los tests | Setup costoso (debe ser `static`) |
| `@AfterAll` | Una vez después de **todos** los tests | Cleanup final |

### 9.2.3. Assertions (verificaciones)

Las **assertions** verifican que el resultado de tu código es el esperado:

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(esperado, real);           // ¿Son iguales?
assertNotNull(objeto);                  // ¿No es null?
assertTrue(condicion);                  // ¿Es true?
assertFalse(condicion);                 // ¿Es false?
assertThrows(Exception.class, () -> {   // ¿Lanza esta excepción?
    // código que debería fallar
});
```

> 📖 **Vocabulario:** Un **assertion** (aserción) es una comprobación que verifica si un resultado cumple lo esperado. Si falla, el test se marca como fallido.

---

## 9.3. Crear tests en IntelliJ IDEA

IntelliJ facilita la creación de tests:

1. Abre la clase que quieres testear (ej: `BarcoDAOImpl`).
2. Pulsa `Ctrl + Shift + T` (o `Navigate > Test`).
3. Selecciona "Create New Test..."
4. Elige JUnit 5 y los métodos a testear.
5. IntelliJ crea la clase de test en `src/test/java`.

> 💡 **TIP:** La convención de nombres es: si la clase se llama `BarcoDAOImpl`, el test se llama `BarcoDAOImplTest`. Siempre con sufijo `Test`.

---

## 9.4. Introducción a Mockito

Nuestros DAOs dependen de Hibernate (`Session`, `SessionFactory`, `Transaction`). Si los tests usaran una BD real, serían lentos, frágiles y difíciles de configurar. La solución: **Mockito**.

### 9.4.1. Dependencia Maven

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.3.1</version>
    <scope>test</scope>
</dependency>
```

### 9.4.2. Concepto de mock

Un **mock** es un objeto "falso" que simula el comportamiento de un objeto real. En lugar de conectarte a MySQL, creas un mock de la `Session` y le dices: "cuando te pidan el barco con ID 1, devuelve este objeto".

> 📖 **Vocabulario:** Un **mock** (simulacro) es un objeto que imita el comportamiento de un objeto real. Se usa en tests para aislar la lógica que estás probando de sus dependencias externas (BD, servicios web, etc.).

### 9.4.3. Anotaciones de Mockito

```java
@ExtendWith(MockitoExtension.class)  // Habilita Mockito en JUnit 5
public class BarcoDAOImplTest {

    @Mock                             // Crea un mock de SessionFactory
    private SessionFactory sessionFactory;

    @Mock                             // Crea un mock de Session
    private Session session;

    @InjectMocks                      // Inyecta los mocks en esta clase
    private BarcoDAOImpl barcoDAO;
}
```

| Anotación | Qué hace |
|-----------|----------|
| `@ExtendWith(MockitoExtension.class)` | Activa Mockito en la clase de test |
| `@Mock` | Crea un mock (objeto simulado) |
| `@InjectMocks` | Crea una instancia real de la clase e inyecta los mocks como dependencias |

### 9.4.4. `when().thenReturn()` — definir comportamiento

```java
// "Cuando alguien llame a sessionFactory.openSession(), devuelve este mock de session"
when(sessionFactory.openSession()).thenReturn(session);

// "Cuando alguien llame a session.find(Barco.class, 1L), devuelve este barco"
Barco barcoPrueba = new Barco();
barcoPrueba.setNombre("Test");
when(session.find(Barco.class, 1L)).thenReturn(barcoPrueba);
```

### 9.4.5. `verify()` — verificar que se llamó a un método

```java
// Verificar que session.save() fue llamado con el barco correcto
verify(session).save(barco);

// Verificar que transaction.commit() fue llamado exactamente 1 vez
verify(transaction, times(1)).commit();
```

---

## 9.5. Patrón AAA: Arrange, Act, Assert

Todos los tests siguen el mismo patrón de tres pasos:

```java
@Test
void testFindById() {
    // ── ARRANGE (Preparar) ──
    Barco barcoEsperado = new Barco();
    barcoEsperado.setId(1L);
    barcoEsperado.setNombre("Test");
    when(session.find(Barco.class, 1L)).thenReturn(barcoEsperado);

    // ── ACT (Actuar) ──
    Barco resultado = barcoDAO.findById(1L);

    // ── ASSERT (Verificar) ──
    assertNotNull(resultado);
    assertEquals("Test", resultado.getNombre());
}
```

> 🏆 **Buena práctica:** Sigue siempre el patrón **AAA** (Arrange-Act-Assert). Hace que los tests sean claros y fáciles de leer: preparas los datos, ejecutas la acción, verificas el resultado.

---

## 9.6. Práctica completa: Tests para `BarcoDAOImpl`

```java
package com.mycompany.hibernate.dao.impl;

import com.mycompany.hibernate.modelo.Barco;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;
import org.hibernate.query.Query;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
public class BarcoDAOImplTest {

    @Mock
    private SessionFactory sessionFactory;

    @Mock
    private Session session;

    @Mock
    private Transaction transaction;

    @Mock
    private Query<Barco> query;

    @InjectMocks
    private BarcoDAOImpl barcoDAO;

    // ── SETUP: se ejecuta ANTES de cada test ──
    @BeforeEach
    void setUp() {
        when(sessionFactory.openSession()).thenReturn(session);
        when(session.beginTransaction()).thenReturn(transaction);
    }

    // ════════════════════════════════════════
    // TEST: save()
    // ════════════════════════════════════════
    @Test
    void testSave() {
        // Arrange
        Barco barco = new Barco();
        barco.setNombre("Barco Test");
        barco.setTipo("Velero");

        // Act
        barcoDAO.save(barco);

        // Assert: verificar que se llamó a los métodos correctos
        verify(session).save(barco);
        verify(transaction).commit();
    }

    // ════════════════════════════════════════
    // TEST: findById()
    // ════════════════════════════════════════
    @Test
    void testFindById() {
        // Arrange
        Barco barcoEsperado = new Barco();
        barcoEsperado.setId(1L);
        barcoEsperado.setNombre("Estrella del Mar");
        when(session.find(Barco.class, 1L)).thenReturn(barcoEsperado);

        // Act
        Barco resultado = barcoDAO.findById(1L);

        // Assert
        assertNotNull(resultado);
        assertEquals(1L, resultado.getId());
        assertEquals("Estrella del Mar", resultado.getNombre());
        verify(session).find(Barco.class, 1L);
    }

    // ════════════════════════════════════════
    // TEST: findById() cuando no existe
    // ════════════════════════════════════════
    @Test
    void testFindByIdNoExiste() {
        // Arrange
        when(session.find(Barco.class, 999L)).thenReturn(null);

        // Act
        Barco resultado = barcoDAO.findById(999L);

        // Assert
        assertNull(resultado);
    }

    // ════════════════════════════════════════
    // TEST: findAll()
    // ════════════════════════════════════════
    @Test
    void testFindAll() {
        // Arrange
        List<Barco> barcosEsperados = Arrays.asList(new Barco(), new Barco());
        when(session.createQuery("from Barco", Barco.class)).thenReturn(query);
        when(query.getResultList()).thenReturn(barcosEsperados);

        // Act
        List<Barco> resultado = barcoDAO.findAll();

        // Assert
        assertNotNull(resultado);
        assertEquals(2, resultado.size());
    }

    // ════════════════════════════════════════
    // TEST: delete()
    // ════════════════════════════════════════
    @Test
    void testDelete() {
        // Arrange
        Barco barco = new Barco();
        barco.setId(1L);

        // Act
        barcoDAO.delete(barco);

        // Assert
        verify(session).delete(barco);
        verify(transaction).commit();
    }

    // ════════════════════════════════════════
    // TEST: update()
    // ════════════════════════════════════════
    @Test
    void testUpdate() {
        // Arrange
        Barco barco = new Barco();
        barco.setId(1L);
        barco.setNombre("Nombre actualizado");

        // Act
        barcoDAO.update(barco);

        // Assert
        verify(session).update(barco);
        verify(transaction).commit();
    }
}
```

---

## 9.7. Ejecutar los tests

En IntelliJ IDEA:
- Haz clic derecho sobre la clase de test → `Run 'BarcoDAOImplTest'`.
- O pulsa `Ctrl + Shift + F10`.

Verás una barra **verde** si todos los tests pasan. Si hay un fallo, la barra será **roja** y JUnit te indica exactamente qué assertion falló.

> 🔍 **¿Sabías que?** En proyectos profesionales, es habitual tener cientos de tests y ejecutarlos con `mvn test` desde la terminal. Maven ejecuta automáticamente todos los tests del directorio `src/test/`.

---

## 📝 Glosario del Capítulo 9

| Término | Definición |
|---------|------------|
| **Prueba unitaria** | Test automatizado que verifica una unidad pequeña de código de forma aislada |
| **JUnit 5** | Framework estándar de Java para escribir y ejecutar pruebas |
| **Mockito** | Librería para crear mocks (objetos simulados) en pruebas |
| **Mock** | Objeto falso que simula el comportamiento de un objeto real |
| **`@Test`** | Anotación que marca un método como prueba unitaria |
| **`@BeforeEach`** | Método que se ejecuta antes de cada test (setup) |
| **`@Mock`** | Crea un mock de un objeto |
| **`@InjectMocks`** | Crea la instancia real e inyecta los mocks |
| **`when().thenReturn()`** | Define qué debe devolver un mock cuando se le llama |
| **`verify()`** | Comprueba que un método del mock fue invocado |
| **Assertion** | Verificación de que un resultado cumple lo esperado |
| **AAA** | Patrón Arrange-Act-Assert: preparar, ejecutar, verificar |
| **Barra verde** | Todos los tests han pasado correctamente |
| **Barra roja** | Al menos un test ha fallado |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Tener la clase `BarcoDAOImplTest` con 6 tests funcionales.
- Entender los mocks y por qué no necesitamos una BD real para testear.
- Saber usar `when().thenReturn()` y `verify()`.
- Aplicar el patrón AAA en todos tus tests.
- Ver la **barra verde** al ejecutar los tests. 🟢

---

*Capítulo anterior: [← Capítulo 8. Consultas Avanzadas](Cap08_Consultas.md)*

*Siguiente capítulo: [Capítulo 10. Introducción a Spring Boot →](Cap10_SpringBoot.md)*
