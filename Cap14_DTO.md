# Capítulo 14. DTO y Patrón Mapper

---

🎯 **Objetivo del capítulo:** Entender por qué NO debemos exponer las entidades JPA directamente en la API, y cómo usar DTOs (Data Transfer Objects) y Mappers para transferir datos de forma segura.

---

## 14.1. El problema: entidades JPA en la API

Si devuelves una entidad JPA directamente desde la API REST:

```java
// ❌ MAL: devolver la entidad directamente
@GetMapping("/api/barcos")
public List<Barco> getAll() {
    return barcoRepository.findAll();
}
```

Ocurren **dos problemas graves**:

### Problema 1: Bucle infinito en el JSON
```
Barco tiene → Amarre
Amarre tiene → Barco
Barco tiene → Amarre
... → StackOverflowError 💥
```

Jackson intenta serializar `Barco`, que tiene un `Amarre`, que tiene una referencia de vuelta a `Barco`, que tiene un `Amarre`... bucle infinito.

### Problema 2: Exposición de datos internos
```json
{
  "id": 1,
  "nombre": "Estrella del Mar",
  "tipo": "Velero",
  "eslora": 12,
  "amarre": {                    // ← ¡Datos internos expuestos!
    "id": 1,
    "precio": 150.00,
    "electricidad": true,
    "barco": {                   // ← ¡Referencia circular!
      "id": 1, ...
    }
  },
  "regatas": [...]               // ← ¡Carga TODA la colección!
}
```

> ⚠️ **CUIDADO:** Exponer entidades JPA directamente viola el principio de **encapsulación**. La estructura interna de la BD no debe ser visible para los clientes de la API.

---

## 14.2. La solución: DTOs

Un **DTO** (Data Transfer Object) es una clase simple que **solo contiene los datos que queremos transferir**:

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/dto/BarcoDTO.java`
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BarcoDTO {
    private Long id;
    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;
    // ← SIN relaciones (amarre, regatas)
    // ← SIN lógica de negocio
    // ← SIN anotaciones JPA
}
```

### Comparación: Entidad vs DTO

| Aspecto | Entidad (`Barco`) | DTO (`BarcoDTO`) |
|---------|-------------------|------------------|
| **Propósito** | Representar tabla en la BD | Transferir datos por la red |
| **Anotaciones** | `@Entity`, `@Table`, `@Column` | Solo `@Data` (Lombok) |
| **Relaciones** | `@OneToOne`, `@ManyToMany` | No tiene relaciones |
| **Usado en** | Servicio ↔ Repositorio | Controlador ↔ Cliente |
| **Visibilidad** | Interna (backend) | Externa (API) |

```
  Cliente     ↔  BarcoDTO    ↔  Controlador
  (Postman)                       |
                                  ↕
                              BarcoService
                                  |
                                  ↕
               Barco (Entidad) ↔ Repositorio ↔ BD
```

---

## 14.3. DTOs múltiples por entidad

A veces necesitas **diferentes vistas** de la misma entidad:

### DTO completo (para detalle):

💡 *Ejemplo didáctico — Ya presentado arriba como 📋 `BarcoDTO.java`*
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BarcoDTO {
    private Long id;
    private String nombre;
    private String tipo;
    private int eslora;
    private int manga;
    private int capacidad;
}
```

### DTO resumido (para listados):

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/dto/BarcoSummaryDTO.java`
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BarcoSummaryDTO {
    private Long id;
    private String nombre;
    private String tipo;
    // ← Solo 3 campos (menos tráfico de red)
}
```

> 💡 **TIP:** Tener un DTO "ligero" para listados reduce significativamente el tráfico de red cuando tienes miles de registros. No necesitas enviar los 6 campos de cada barco si solo muestras nombre y tipo en una tabla.

---

## 14.4. Patrón Mapper

El **Mapper** es la clase que convierte entre entidades y DTOs. Centraliza la conversión en un solo lugar:

📋 **ARCHIVO DEL PROYECTO** — `src/main/java/com/example/marina/mapper/BarcoMapper.java`

```java
public class BarcoMapper {

    // Entidad → DTO (para enviar al cliente)
    public static BarcoDTO toDTO(Barco barco) {
        return new BarcoDTO(
                barco.getId(),
                barco.getNombre(),
                barco.getTipo(),
                barco.getEslora(),
                barco.getManga(),
                barco.getCapacidad()
        );
    }

    // Entidad → DTO resumido (para listados)
    public static BarcoSummaryDTO toSummaryDTO(Barco barco) {
        return new BarcoSummaryDTO(
                barco.getId(),
                barco.getNombre(),
                barco.getTipo()
        );
    }

    // DTO → Entidad (para crear)
    public static Barco toEntity(BarcoDTO dto) {
        Barco barco = new Barco();
        barco.setNombre(dto.getNombre());
        barco.setTipo(dto.getTipo());
        barco.setEslora(dto.getEslora());
        barco.setManga(dto.getManga());
        barco.setCapacidad(dto.getCapacidad());
        // ← NO asigna ID: la BD lo genera con AUTO_INCREMENT
        return barco;
    }

    // DTO → Entidad existente (para actualizar)
    public static void updateEntityFromDTO(BarcoDTO dto, Barco barco) {
        barco.setNombre(dto.getNombre());
        barco.setTipo(dto.getTipo());
        barco.setEslora(dto.getEslora());
        barco.setManga(dto.getManga());
        barco.setCapacidad(dto.getCapacidad());
        // ← MANTIENE el ID y las relaciones intactos
    }
}
```

### ¿Por qué `updateEntityFromDTO` es diferente a `toEntity`?

```
toEntity():           DTO → NUEVA entidad (sin ID)    → INSERT
updateEntityFromDTO():  DTO → entidad EXISTENTE (con ID) → UPDATE
```

> 🏆 **Buena práctica:** No uses `toEntity()` para actualizar, porque perderías el ID y las relaciones. Usa siempre `updateEntityFromDTO()` para preservar los datos existentes.

---

## 14.5. Flujo completo con DTOs

### Crear un barco nuevo (POST):
```
1. Cliente envía JSON → {"nombre":"Rayo","tipo":"Motor","eslora":15}
2. Spring convierte JSON → BarcoDTO (Jackson)
3. Servicio llama BarcoMapper.toEntity(dto) → Barco (sin ID)
4. Repositorio save(barco) → Barco (con ID generado)
5. Servicio llama BarcoMapper.toDTO(barco) → BarcoDTO (con ID)
6. Spring convierte BarcoDTO → JSON (Jackson)
7. Cliente recibe → {"id":8,"nombre":"Rayo","tipo":"Motor","eslora":15}
```

### Actualizar un barco (PUT):
```
1. Cliente envía PUT /api/barcos/8 + JSON con datos nuevos
2. Servicio busca barco existente: findById(8) → Barco (con relaciones)
3. BarcoMapper.updateEntityFromDTO(dto, barco) → Actualiza campos
4. Repositorio save(barco) → Barco actualizado (relaciones intactas)
5. Servicio convierte a DTO y devuelve
```

---

## 14.6. Uso en el servicio

```java
@Service
public class BarcoService {

    public List<BarcoDTO> findAll() {
        return barcoRepository.findAll()       // List<Barco>
                .stream()                       // Stream<Barco>
                .map(BarcoMapper::toDTO)        // Stream<BarcoDTO>
                .collect(Collectors.toList());  // List<BarcoDTO>
    }

    @Transactional
    public BarcoDTO save(BarcoDTO dto) {
        Barco barco = BarcoMapper.toEntity(dto);        // DTO → Entidad
        Barco saved = barcoRepository.save(barco);       // Persistir
        return BarcoMapper.toDTO(saved);                 // Entidad → DTO
    }

    @Transactional
    public BarcoDTO update(Long id, BarcoDTO dto) {
        return barcoRepository.findById(id).map(barco -> {
            BarcoMapper.updateEntityFromDTO(dto, barco); // Actualizar
            return BarcoMapper.toDTO(barcoRepository.save(barco));
        }).orElse(null);
    }
}
```

---

## 📝 Glosario del Capítulo 14

| Término | Definición |
|---------|------------|
| **DTO** | Data Transfer Object — clase simple para transferir datos |
| **Mapper** | Clase que convierte entre entidades y DTOs |
| **Serialización** | Convertir objeto Java → JSON |
| **Deserialización** | Convertir JSON → objeto Java |
| **Bucle circular** | Referencia bidireccional que causa StackOverflowError al serializar |
| **Encapsulación** | Ocultar la estructura interna, exponer solo lo necesario |
| **SRP** | Single Responsibility Principle — cada clase tiene una sola responsabilidad |

---

✅ **Checkpoint:** Al terminar este capítulo debes...
- Entender por qué NO se deben exponer entidades JPA en la API
- Saber crear DTOs para diferentes vistas de una entidad
- Implementar un Mapper con conversiones bidireccionales
- Distinguir entre `toEntity()` (crear) y `updateEntityFromDTO()` (actualizar)
- Usar Streams y method references para convertir listas

---

*Capítulo anterior: [← Capítulo 13. API REST](Cap13_REST.md)*

*Siguiente capítulo: [Capítulo 15. Swagger / OpenAPI →](Cap15_Swagger.md)*
