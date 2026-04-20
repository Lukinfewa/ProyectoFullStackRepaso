# Capítulo 13. API REST con Spring Boot

---

🎯 **Objetivo del capítulo:** Construir una API REST completa con `@RestController`, entender los verbos HTTP, los códigos de respuesta, y cómo Spring convierte automáticamente objetos Java en JSON.

---

## 13.1. ¿Qué es una API REST?

**REST** (Representational State Transfer) es un estilo de arquitectura para comunicar aplicaciones a través de HTTP.

```
  Cliente (Postman, móvil, web)          Servidor (Spring Boot)
  ┌─────────────────────┐               ┌──────────────────┐
  │                     │  HTTP Request  │                  │
  │  GET /api/barcos    │ ─────────────→ │  BarcoController │
  │                     │                │       │          │
  │                     │  HTTP Response │       ▼          │
  │  [JSON con barcos]  │ ←───────────── │  BarcoService    │
  │                     │  200 OK        │       │          │
  └─────────────────────┘               │       ▼          │
                                        │  BarcoRepository │
                                        │       │          │
                                        │       ▼          │
                                        │    MySQL BD      │
                                        └──────────────────┘
```

### Los 5 verbos HTTP fundamentales:

| Verbo | Acción | URL | Descripción |
|-------|--------|-----|-------------|
| `GET` | Leer | `/api/barcos` | Obtener todos los barcos |
| `GET` | Leer uno | `/api/barcos/1` | Obtener barco con ID 1 |
| `POST` | Crear | `/api/barcos` | Crear barco nuevo (datos en body) |
| `PUT` | Actualizar | `/api/barcos/1` | Actualizar barco con ID 1 |
| `DELETE` | Eliminar | `/api/barcos/1` | Eliminar barco con ID 1 |

### Códigos de respuesta HTTP:

| Código | Significado | ¿Cuándo se usa? |
|--------|------------|-----------------|
| `200 OK` | Éxito | GET, PUT exitosos |
| `201 Created` | Recurso creado | POST exitoso |
| `204 No Content` | Éxito sin cuerpo | DELETE exitoso |
| `400 Bad Request` | Datos inválidos | Validación fallida |
| `404 Not Found` | No encontrado | GET de un ID que no existe |
| `500 Internal Server Error` | Error del servidor | Excepción no controlada |

> 📖 **Vocabulario:**
> - **Endpoint**: URL específica que responde a peticiones HTTP (ej: `/api/barcos`).
> - **Body (cuerpo)**: datos enviados con la petición (en POST y PUT).
> - **JSON**: formato de datos ligero (JavaScript Object Notation). Es el estándar para APIs REST.

---

## 13.2. `@RestController` — Nuestro controlador REST

```java
@RestController                        // ← Controlador REST (devuelve JSON)
@RequestMapping("/api/barcos")         // ← URL base para todos los endpoints
@Tag(name = "Barcos", description = "Operaciones CRUD sobre barcos")
public class BarcoController {

    @Autowired
    private BarcoService barcoService; // ← Inyecta el servicio (NO el repositorio)
}
```

### Diferencia: `@Controller` vs `@RestController`

| `@Controller` | `@RestController` |
|--------------|-------------------|
| Devuelve **vistas HTML** (Thymeleaf) | Devuelve **JSON** |
| Necesita `return "nombre-vista"` | El objeto se serializa directamente |
| Usado para aplicaciones web con HTML | Usado para APIs REST |
| `@Controller` = `@Component` | `@RestController` = `@Controller` + `@ResponseBody` |

---

## 13.3. Implementar los 5 endpoints CRUD

### GET — Listar todos los barcos
```java
@GetMapping
public ResponseEntity<List<BarcoDTO>> getAllBarcos() {
    List<BarcoDTO> barcos = barcoService.findAll();
    return new ResponseEntity<>(barcos, HttpStatus.OK);   // 200 OK
}
```

**Prueba con curl:**
```bash
curl http://localhost:8080/api/barcos
```

**Respuesta JSON:**
```json
[
  {"id": 1, "nombre": "Estrella del Mar", "tipo": "Velero", "eslora": 12, "manga": 4, "capacidad": 8},
  {"id": 2, "nombre": "Rayo Azul", "tipo": "Motor", "eslora": 15, "manga": 5, "capacidad": 10},
  {"id": 3, "nombre": "Corsario Negro", "tipo": "Velero", "eslora": 18, "manga": 6, "capacidad": 12}
]
```

### GET por ID — Buscar un barco
```java
@GetMapping("/{id}")
public ResponseEntity<BarcoDTO> getBarcoById(@PathVariable Long id) {
    BarcoDTO barco = barcoService.findById(id);
    if (barco != null) {
        return new ResponseEntity<>(barco, HttpStatus.OK);         // 200 OK
    }
    return new ResponseEntity<>(HttpStatus.NOT_FOUND);             // 404
}
```

> 📖 **Vocabulario:**
> - **`@PathVariable`**: extrae un valor de la URL. En `/api/barcos/3`, el `{id}` se convierte en `Long id = 3`.
> - **`ResponseEntity<T>`**: envoltorio que permite controlar el código de estado HTTP + el cuerpo de la respuesta.

### POST — Crear barco nuevo
```java
@PostMapping
public ResponseEntity<BarcoDTO> createBarco(@RequestBody BarcoDTO barcoDTO) {
    BarcoDTO created = barcoService.save(barcoDTO);
    return new ResponseEntity<>(created, HttpStatus.CREATED);      // 201 Created
}
```

**Prueba con curl:**
```bash
curl -X POST http://localhost:8080/api/barcos \
     -H "Content-Type: application/json" \
     -d '{"nombre":"Nuevo Barco","tipo":"Velero","eslora":14,"manga":5,"capacidad":8}'
```

> 📖 **Vocabulario:**
> - **`@RequestBody`**: le dice a Spring que convierta el JSON del cuerpo de la petición en un objeto Java (BarcoDTO). Spring usa **Jackson** para esta conversión automática.

### PUT — Actualizar barco existente
```java
@PutMapping("/{id}")
public ResponseEntity<BarcoDTO> updateBarco(@PathVariable Long id,
                                             @RequestBody BarcoDTO barcoDTO) {
    BarcoDTO updated = barcoService.update(id, barcoDTO);
    if (updated != null) {
        return new ResponseEntity<>(updated, HttpStatus.OK);       // 200 OK
    }
    return new ResponseEntity<>(HttpStatus.NOT_FOUND);             // 404
}
```

### DELETE — Eliminar barco
```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteBarco(@PathVariable Long id) {
    barcoService.deleteById(id);
    return new ResponseEntity<>(HttpStatus.NO_CONTENT);            // 204
}
```

**Prueba con curl:**
```bash
curl -X DELETE http://localhost:8080/api/barcos/5
```

---

## 13.4. Serialización JSON automática (Jackson)

Spring Boot incluye **Jackson**, que convierte automáticamente objetos Java en JSON:

```
   Java Object                          JSON
   ┌──────────────────┐                ┌───────────────────────┐
   │ BarcoDTO          │   Jackson      │ {                     │
   │   id = 1          │ ──────────→    │   "id": 1,            │
   │   nombre = "..."  │               │   "nombre": "...",    │
   │   tipo = "Velero" │               │   "tipo": "Velero"    │
   └──────────────────┘                └───────────────────────┘
```

> ⚠️ **CUIDADO:** Si devuelves una **entidad JPA** directamente (en vez de un DTO), Jackson puede entrar en un **bucle infinito** serializando relaciones bidireccionales:
> ```
> Barco → Amarre → Barco → Amarre → ... → StackOverflowError
> ```
> Por eso es importante usar **DTOs** (Capítulo 14).

---

## 13.5. `ResponseEntity` en detalle

`ResponseEntity` es un contenedor que incluye:
1. **Cuerpo** (el objeto a devolver)
2. **Código de estado HTTP**
3. **Cabeceras** (headers) opcionales

```java
// Versión completa
return new ResponseEntity<>(barco, HttpStatus.OK);

// Versión fluida (builder)
return ResponseEntity.ok(barco);                     // 200 + body
return ResponseEntity.status(HttpStatus.CREATED)     // 201 + body
        .body(barco);
return ResponseEntity.notFound().build();            // 404 sin body
return ResponseEntity.noContent().build();           // 204 sin body
```

---

## 13.6. Endpoint adicional: filtrar por tipo
```java
@GetMapping("/tipo/{tipo}")
public ResponseEntity<List<BarcoDTO>> getBarcosByTipo(@PathVariable String tipo) {
    return new ResponseEntity<>(barcoService.findByTipo(tipo), HttpStatus.OK);
}
```

**Prueba:**
```bash
curl http://localhost:8080/api/barcos/tipo/Velero
```

```json
[
  {"id": 1, "nombre": "Estrella del Mar", "tipo": "Velero", "eslora": 12, ...},
  {"id": 3, "nombre": "Corsario Negro", "tipo": "Velero", "eslora": 18, ...}
]
```

---

## 13.7. Resumen de anotaciones de controlador

| Anotación | Verbo HTTP | Uso |
|-----------|-----------|-----|
| `@GetMapping` | GET | Obtener datos |
| `@PostMapping` | POST | Crear recurso nuevo |
| `@PutMapping` | PUT | Actualizar recurso existente |
| `@DeleteMapping` | DELETE | Eliminar recurso |
| `@RequestMapping("/api/barcos")` | Todos | URL base del controlador |
| `@PathVariable` | — | Extraer valor de la URL |
| `@RequestBody` | — | Convertir JSON del body a objeto |
| `@ResponseBody` | — | Convertir objeto a JSON (implícito en `@RestController`) |

---

## 13.8. Controladores REST de Amarre y Regata

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/controller/AmarreController.java`

```java
package com.example.marina.controller;

import com.example.marina.model.Amarre;
import com.example.marina.service.AmarreService;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/amarres")
@Tag(name = "Amarres", description = "Gestión de amarres del puerto")
public class AmarreController {

    @Autowired
    private AmarreService amarreService;

    @GetMapping
    public ResponseEntity<List<Amarre>> getAll() {
        return new ResponseEntity<>(amarreService.findAll(), HttpStatus.OK);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Amarre> getById(@PathVariable Long id) {
        Amarre amarre = amarreService.findById(id);
        return amarre != null
                ? new ResponseEntity<>(amarre, HttpStatus.OK)
                : new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }

    @PostMapping
    public ResponseEntity<Amarre> create(@RequestBody Amarre amarre) {
        return new ResponseEntity<>(amarreService.save(amarre), HttpStatus.CREATED);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        amarreService.deleteById(id);
        return new ResponseEntity<>(HttpStatus.NO_CONTENT);
    }

    @GetMapping("/libres")
    public ResponseEntity<List<Amarre>> getLibres() {
        return new ResponseEntity<>(amarreService.findLibres(), HttpStatus.OK);
    }
}
```

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/controller/RegataController.java`

```java
package com.example.marina.controller;

import com.example.marina.model.Regata;
import com.example.marina.service.RegataService;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/regatas")
@Tag(name = "Regatas", description = "Gestión de regatas")
public class RegataController {

    @Autowired
    private RegataService regataService;

    @GetMapping
    public ResponseEntity<List<Regata>> getAll() {
        return new ResponseEntity<>(regataService.findAll(), HttpStatus.OK);
    }

    @GetMapping("/{id}")
    public ResponseEntity<Regata> getById(@PathVariable Long id) {
        Regata regata = regataService.findById(id);
        return regata != null
                ? new ResponseEntity<>(regata, HttpStatus.OK)
                : new ResponseEntity<>(HttpStatus.NOT_FOUND);
    }

    @PostMapping
    public ResponseEntity<Regata> create(@RequestBody Regata regata) {
        return new ResponseEntity<>(regataService.save(regata), HttpStatus.CREATED);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        regataService.deleteById(id);
        return new ResponseEntity<>(HttpStatus.NO_CONTENT);
    }
}
```

> 💡 **TIP:** Fíjate que `AmarreController` y `RegataController` trabajan con **entidades directamente**
> (no con DTOs). Esto es aceptable cuando la entidad no tiene relaciones circulares expuestas.
> Para `Barco` usamos DTOs porque su relación bidireccional con `Amarre` causaría un bucle infinito en JSON.

---

## 📝 Glosario del Capítulo 13

| Término | Definición |
|---------|------------|
| **REST** | Estilo de arquitectura para APIs basadas en HTTP |
| **Endpoint** | URL que responde a peticiones HTTP |
| **JSON** | Formato de datos ligero (JavaScript Object Notation) |
| **`@RestController`** | Controlador que devuelve JSON automáticamente |
| **`@PathVariable`** | Extrae valores de la URL |
| **`@RequestBody`** | Convierte JSON del body a objeto Java |
| **`ResponseEntity`** | Envoltorio para controlar código HTTP + cuerpo |
| **Jackson** | Librería de serialización JSON (incluida en Spring Boot) |
| **Serialización** | Convertir un objeto Java en texto JSON |
| **Deserialización** | Convertir texto JSON en un objeto Java |
| **curl** | Herramienta de línea de comandos para hacer peticiones HTTP |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Entender qué es una API REST y los 5 verbos HTTP
- Saber implementar un `@RestController` con CRUD completo
- Usar `ResponseEntity` para controlar los códigos de respuesta
- Saber probar endpoints con `curl` o el navegador
- Entender cómo Jackson convierte Java ↔ JSON automáticamente

---

*Capítulo anterior: [← Capítulo 12. Capa de Servicio](Cap12_Servicios.md)*

*Siguiente capítulo: [Capítulo 14. DTO y Mapper Pattern →](Cap14_DTO.md)*
