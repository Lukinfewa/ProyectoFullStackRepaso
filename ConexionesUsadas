
# Entendiendo el Stack: De qué sirve realmente cada herramienta

> **Nota importante:** Este documento explica qué hace cada programa de tu stack, por qué está ahí y si realmente lo necesitas.  
> La mayoría de tutoriales tiran todo junto desde el minuto 1 sin explicar el porqué. Esto es el manual que debería venir antes del Capítulo 1.

---

## 📋 El problema real

Cuando empezamos, aparecen de golpe: Docker, Maven, Hibernate, Spring Boot, JPA, Lombok, Flyway...  
Y nadie explica nunca: *"Esto resuelve este problema específico, lo necesitas porque..."*

Resultado: Copias y pegas configuraciones sin entender qué pieza hace qué, y cuando falla algo no sabes ni por dónde empezar a mirar.

---

## 🔍 Tabla comparativa honesta

Ordenadas de bajo a alto nivel (de lo que realmente guarda datos a lo que solo te ahorra escribir código):

| Herramienta | Qué hace realmente | La verdad incómoda | ¿Lo necesito? |
|-------------|-------------------|-------------------|---------------|
| **🐘 PostgreSQL/MySQL** | Es el **único** programa aquí que realmente persiste tus datos. Todo lo demás es un intermediario para hablar con él. | Puedes eliminar todo el resto de esta lista y seguir tener una app funcional usando solo JDBC. | ✅ **Obligatorio** |
| **🐳 Docker Compose** | Levanta una base de datos limpia, aislada y configurada en 2 segundos. Cuando terminas, desaparece sin dejar rastro en tu sistema. | Es 100% comodidad para desarrollo. Tu aplicación funciona exactamente igual si instalas Postgres a mano. No es magia, es solo un "auto-instalador". | 🟡 **Opcional** (pero hoy en día es más trabajo no usarlo) |
| **🪶 Maven/Gradle** | Descarga librerías de internet, resuelve conflictos de versiones automáticamente y empaqueta tu app. | Antes de Maven existía el "Infierno de los JARs": descargar archivos a mano de páginas web y pasar 3 días arreglando `ClassNotFoundException`. | ✅ **Obligatorio** |
| **📡 JDBC** | El protocolo más básico de Java para hablar SQL con cualquier base de datos. | Es invisible. Nadie lo usa directamente desde 2008, pero **absolutamente todo** (Hibernate, Spring Data...) está construido encima de JDBC. | ⚫ **Invisible** (siempre está ahí, no lo tocas) |
| **🛏️ Hibernate** | Convierte objetos Java ↔ Filas de SQL automáticamente. Eso es todo. | Es la librería más odiada, más incomprendida y más usada de Java. No tiene competencia real. Escribir todo el SQL a mano es viable pero lentísimo. | 🟡 **Casi obligatorio** (99% de empleos lo usan) |
| **📄 JPA** | **No es código.** Es un documento (especificación) con reglas y anotaciones. Dice: *"Si quieres ser JPA, debes tener estos métodos"*. | Hibernate **es** una implementación de JPA. Decir "uso JPA" cuando usas Hibernate es como decir "uso electricidad" cuando enchufas una lamparita. Sinónimo universal pero técnicamente impreciso. | 🟡 **Teóricamente innecesario, prácticamente universal** |
| **📚 Spring Data JPA** | Genera automáticamente los DAOs/Repositorios. Tú defines una interfaz, él escribe el código SQL por ti. | No hace NADA real. Solo llama a Hibernate por ti. Es un ahorrador de código. Puedes usar Hibernate puro perfectamente (mucha gente senior lo prefiere). | 🟢 **Muy recomendado, opcional** |
| **🍃 Spring Boot** | Auto-configura todo lo anterior. Antes necesitabas 400 líneas de XML para que Hibernate + Tomcat arrancaran juntos. | Es la mayor "estafa de éxito" de la historia. No inventa nada nuevo, solo configura lo existente. Y todo el mundo le da el crédito de todo lo que hace Hibernate debajo. | ✅ **Prácticamente obligatorio hoy** |
| **✨ Lombok** | Genera getters, setters y constructores en tiempo de compilación. Te ahorra el 50% de líneas de código de tus entidades. | Es azúcar sintáctico puro. No aporta funcionalidad que no puedas escribir tú. Existen guerras santas entre quienes lo aman y quienes lo odian (arguyen que oculta código). | 🔴 **100% opcional** (pero todo el mundo lo usa) |
| **✈️ Flyway/Liquibase** | Lleva la cuenta de cambios en la base de datos. Cuando arrancas, aplica automáticamente los scripts nuevos (`ALTER TABLE`, etc.). | Todo el mundo lo pone en producción. En desarrollo local puedes perfectamente hacer `spring.jpa.hibernate.ddl-auto=create-drop` y que Hibernate rehaga la base en cada arranque. | ✅ **Obligatorio en producción, innecesario en pruebas locales** |

---

## 🎯 La gran verdad

Solo hay **dos** cosas en todo este stack que realmente hacen trabajo pesado:

1. **La base de datos** (guarda bits en disco)
2. **Hibernate** (traduce Java ↔ SQL)

**Todo lo demás** (Maven, Docker, Spring Boot, Lombok, Spring Data) son simplemente **herramientas para que escribas menos código y menos XML**.

Ninguna añade funcionalidad nueva. Todas se pueden eliminar y tu app seguiría funcionando (con más trabajo por tu parte).

---

## ❌ Por qué todos los tutoriales te confunden

**El error:** Te dan las 10 herramientas simultáneamente en el Capítulo 1.

**El resultado:** Crees que todo es magia, que Spring Boot guarda los datos (no, es Hibernate), o que Docker es parte de tu aplicación (no, solo levanta la base de datos).

**El orden correcto para aprender (que nadie enseña):**
1. Java puro + JDBC (para sufrir y entender el problema)
2. Reemplazar JDBC por Hibernate puro (para ver qué resuelve)
3. Agregar Spring Data (para ver qué ahorra)
4. Agregar Spring Boot (para ver qué configura)
5. Agregar Docker (para no instalar Postgres)
6. Agregar Flyway (para no romper producción)

---

## ✅ Checklist mental rápido

- **¿Sin esto no tengo aplicación?** → Solo la Base de Datos y Java.
- **¿Sin esto escribo más código repetitivo?** → Hibernate, Spring Data, Lombok.
- **¿Sin esto tengo que configurar a mano?** → Spring Boot, Maven.
- **¿Sin esto tengo que instalar software en mi PC?** → Docker.
- **¿Sin esto rompo producción al hacer deploy?** → Flyway.

---

> 💡 **Regla de oro:** Si no sabes para qué sirve una herramienta de tu stack, prueba a quitarla. Si todo sigue funcionando (aunque sea más incómodo), era opcional.
