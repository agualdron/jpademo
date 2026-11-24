# Demostración paso a paso de JPA
En esta demostración se abordarán los siguientes conceptos clave:
- Estructura Maven, starter JPA, H2 en memoria, observabilidad SQL.
- `@Entity`, `@Id`, `JpaRepository`, Persistence Context, Dirty Checking.
- `@ManyToOne`, `@OneToMany`, `mappedBy`, `@JoinColumn`, `CascadeType.ALL`.
- `FetchType.LAZY` vs. `EAGER`, sesión abierta/cerrada, problema N+1.
- Consultas derivadas


# Paso a paso

## 1. Crear nuevo proyecto en VSCode (Spring Initializr)
1. En VSCode oprime `CTRL`+`SHIFT`+`P` y ejecuta `Java: Create Java Project`
2. Selecciona:
   - Tipo de proyecto: **Spring Boot** → **Maven Project**
   - Spring Boot version: **4.2.0**
   - Lenguaje del proyecto: **Java**
   - _Escribe Grup ID_: **com.trainer**
   - _Escribe Artifact ID_: **biblioteca**
   - _Escribe Package Name_: **com.trainer.biblioteca**
   - Tipo de empaquetado: **Jar**
   - Java version: **21**
   - Selecciona las dependencias: **Spring Data JPA**, **H2 Database**
   - _Selecciona la carpeta de trabajo_
   - Click en **Generar en esta carpeta**
   
Otra forma de generar el proyecto es mediante la aplicación web [start.spring.io](https://start.spring.io/) y seleccionando las mismas opciones descritas anteriormente. Esta aplicación genera un archivo comprimido con el proyecto inicializado. Se debe descomprimir y abrir la carpeta en VSCode.

3. Configura el archivo `src/main/resources/application.properties`
```properties
# Base de datos relacional en memoria
spring.datasource.url=jdbc:h2:mem:bibliotecadb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# Estrategia DDL: crea el esquema al iniciar y lo destruye al cerrar
spring.jpa.hibernate.ddl-auto=create-drop

# Visibilidad pedagogica del SQL generado por Hibernate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.highlight_sql=true

# Muestra los valores reales inyectados en los parametros '?'
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

4. Ejecuta el proyecto por primera vez
```bash
./mvnw spring-boot:run
```

Al detectar H2 y Spring Data JPA en el _classpath_, Spring Boot autoconfigura el `DataSource`, el `EntityManagerFactory` y el gestor de transacciones sin escribir archivos XML.
## 2. Crear primera Entidad, CRUD y Dirty Checking
1. Crea el paquete `model` con el archivo `src/main/java/com/trainer/biblioteca/model/Autor.java`. Escribe los atributos y genera los el constructor vacío, el constructor con parámetros y los _getters/setters_ y _toString()_
```java
package com.trainer.biblioteca.model;

import jakarta.persistence.*;

@Entity
@Table(name = "autores")
public class Autor {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nombre;

    @Column(length = 60)
    private String nacionalidad;

    // Constructor vacio obligatorio para que Hibernate instancie la entidad por reflexion
    public Autor() {
    }

    public Autor(String nombre, String nacionalidad) {
        this.nombre = nombre;
        this.nacionalidad = nacionalidad;
    }

    public Long getId() { return id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getNacionalidad() { return nacionalidad; }
    public void setNacionalidad(String nacionalidad) { this.nacionalidad = nacionalidad; }

    @Override
    public String toString() {
        return "Autor{id=" + id + ", nombre='" + nombre + "', nacionalidad='" + nacionalidad + "'}";
    }
}
```
2. Crea `src/main/java/com/trainer/biblioteca/repository/AutorRepository.java`
```java
package com.trainer.biblioteca.repository;

import com.trainer.biblioteca.model.Autor;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AutorRepository extends JpaRepository<Autor, Long> {
}
```
3. Crea `src/main/java/com/trainer/biblioteca/service/BibliotecaService.java` 
```java
package com.trainer.biblioteca.service;

import com.trainer.biblioteca.model.Autor;
import com.trainer.biblioteca.repository.AutorRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BibliotecaService {

    private final AutorRepository autorRepository;

    public BibliotecaService(AutorRepository autorRepository) {
        this.autorRepository = autorRepository;
    }

    @Transactional
    public void actualizarNacionalidadConDirtyChecking(Long id, String nuevaNacionalidad) {
        // 1. La entidad entra en estado MANAGED dentro del Persistence Context
        Autor autor = autorRepository.findById(id).orElseThrow();
        // 2. Modificamos el objeto en memoria Java (NO llamamos a autorRepository.save())
        autor.setNacionalidad(nuevaNacionalidad);
        // 3. Al terminar el metodo, Hibernate compara el snapshot inicial y dispara UPDATE solo
    }
}
```
4. Modifica `BibliotecaJpaApplication.java` para ejecutar un `CommandLineRunner`
```java
package com.trainer.biblioteca;

import com.trainer.biblioteca.model.Autor;
import com.trainer.biblioteca.repository.AutorRepository;
import com.trainer.biblioteca.service.BibliotecaService;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class BibliotecaJpaApplication {

    public static void main(String[] args) {
        SpringApplication.run(BibliotecaJpaApplication.class, args);
    }

    @Bean
    public CommandLineRunner demo(AutorRepository autorRepository, BibliotecaService service) {
        return args -> {
            System.out.println("\n--- 1. CREACION Y GUARDADO ---");
            Autor gabo = autorRepository.save(new Autor("Gabriel Garcia Marquez", "Colombiana"));
            Autor borges = autorRepository.save(new Autor("Jorge Luis Borges", "Argentina"));
            System.out.println("En BD: " + autorRepository.findAll());

            System.out.println("\n--- 2. DIRTY CHECKING EN TRANSACCION ---");
            service.actualizarNacionalidadConDirtyChecking(gabo.getId(), "Colombo-Mexicana");
            System.out.println("Actualizado: " + autorRepository.findById(gabo.getId()).orElseThrow());
        };
    }
}
```
5. Ejecuta el proyecto `./mvnw spring-boot:run`

**Conceptos aplicados:**
- `GenerationType.IDENTITY` delega el autoincremento a la base de datos (por eso Hibernate lanza el `INSERT` de inmediato para obtener el ID).
- **Persistence Context (Caché de primer nivel):** Mientras `@Transactional` está activo, Hibernate rastrea el estado de los objetos _Managed_. Al hacer `commit`, detecta cambios en memoria (_Dirty Checking_) y sincroniza con la base de datos sin necesidad de invocar `save()`.

## 3. Modelado relacional 1:N

<details> <summary>Si un Autor escribe muchos Libros, ¿en cuál de las dos tablas de la base de datos debe crearse la llave foránea?</summary> Respuesta: siempre en el lado _muchos_, es decir, en `libros` </details>

1. Crea `src/main/java/com/trainer/biblioteca/model/Libro.java`
	```java
	package com.trainer.biblioteca.model;

import jakarta.persistence.*;

@Entity
@Table(name = "libros")
public class Libro {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 150)
    private String titulo;

    @Column(name = "anio_publicacion")
    private Integer anioPublicacion;

    // Dueno de la relacion: crea la columna FK "autor_id" en la tabla "libros"
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "autor_id", nullable = false)
    private Autor autor;

    public Libro() {
    }

    public Libro(String titulo, Integer anioPublicacion) {
        this.titulo = titulo;
        this.anioPublicacion = anioPublicacion;
    }

    public Long getId() { return id; }
    public String getTitulo() { return titulo; }
    public void setTitulo(String titulo) { this.titulo = titulo; }
    public Integer getAnioPublicacion() { return anioPublicacion; }
    public void setAnioPublicacion(Integer anioPublicacion) { this.anioPublicacion = anioPublicacion; }
    public Autor getAutor() { return autor; }
    public void setAutor(Autor autor) { this.autor = autor; }

    @Override
    public String toString() {
        return "Libro{id=" + id + ", titulo='" + titulo + "', anio=" + anioPublicacion + "}";
    }
}
	```
2. Actualiza `Autor.java` añadiendo la colección `@OneToMany` y el método ayudante `agregarLibro`
	```java
	// Agregar imports: java.util.ArrayList y java.util.List
    @OneToMany(mappedBy = "autor", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Libro> libros = new ArrayList<>();

    public void agregarLibro(Libro libro) {
        libros.add(libro);
        libro.setAutor(this);
    }

    public List<Libro> getLibros() {
        return libros;
    }
	```
3. Actualiza `BibliotecaJpaApplication.java` para persistir autores con libros en cascada
	```java
	System.out.println("\n--- 3. PERSISTENCIA EN CASCADA 1:N ---");
            gabo.agregarLibro(new Libro("Cien anos de soledad", 1967));
            gabo.agregarLibro(new Libro("El coronel no tiene quien le escriba", 1961));
            borges.agregarLibro(new Libro("Ficciones", 1944));

            // Guardamos solo el Autor; CascadeType.ALL propaga los INSERT a 'libros'
            autorRepository.save(gabo);
            autorRepository.save(borges);
	```

**Conceptos aplicados:**
- **Dueño de la relación vs. Lado inverso:** `@JoinColumn` define la columna física en SQL (`autor_id`). `mappedBy = "autor"` le dice a Hibernate que no cree una tabla intermedia ni duplique la llave foránea, porque la relación ya está mapeada en el atributo `autor` de la clase `Libro`.
- **Sincronización en memoria:** En Java las referencias no se actualizan solas; el método `agregarLibro()` garantiza consistencia en ambos sentidos antes del `commit`.



## 4. `LazyInitializationException` y Problema N+1

1. En `AutorRepository.java`, añade la consulta **JPQL** con `JOIN FETCH`:
	```java
	package com.trainer.biblioteca.repository;

import com.trainer.biblioteca.model.Autor;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import java.util.List;

public interface AutorRepository extends JpaRepository<Autor, Long> {

    @Query("SELECT DISTINCT a FROM Autor a LEFT JOIN FETCH a.libros")
    List<Autor> findAllConLibros();
}	
	```
2. En `BibliotecaService.java`, añade los tres escenarios de carga
	```java
	// Escenario A: Sin @Transactional -> cierra la sesion al salir de findById()
    public Autor buscarAutorDetached(Long id) {
        return autorRepository.findById(id).orElseThrow();
    }

    // Escenario B: Con @Transactional -> permite carga LAZY bajo demanda
    @Transactional(readOnly = true)
    public void imprimirLibrosConSesion(Long id) {
        Autor autor = autorRepository.findById(id).orElseThrow();
        System.out.println("Libros de " + autor.getNombre() + ": " + autor.getLibros().size());
    }

    // Escenario C: Resuelve el problema N+1 en una sola consulta SQL
    @Transactional(readOnly = true)
    public void listarAutoresConLibrosOptimizado() {
        List<Autor> autores = autorRepository.findAllConLibros();
        autores.forEach(a -> System.out.println(a.getNombre() + " -> " + a.getLibros()));
    }
	```
3. Invócalos desde `BibliotecaJpaApplication.java` capturando intencionalmente la excepción
	```java
	System.out.println("\n--- 4. LAZY LOADING Y N+1 ---");
            try {
                Autor gaboDetached = service.buscarAutorDetached(gabo.getId());
                // Al estar en estado Detached, intentar acceder a la coleccion LAZY falla
                System.out.println(gaboDetached.getLibros().size());
            } catch (org.hibernate.LazyInitializationException ex) {
                System.out.println("ERROR ESPERADO: LazyInitializationException (no hay sesion abierta).");
            }

            service.imprimirLibrosConSesion(gabo.getId());
            service.listarAutoresConLibrosOptimizado();
	```
4. Ejecuta `./mvnw spring-boot:run`.

**Conceptos aplicados:**

- **Por qué falla:** Por defecto, `@OneToMany` es `FetchType.LAZY`. Hibernate coloca un _proxy_ en lugar de la lista real. Si la transacción termina, la entidad pasa a estado _Detached_ y el _proxy_ pierde su conexión a la base de datos.
    
- **Por qué nunca usar `FetchType.EAGER` como parche:** Si listaras 100 autores con `findAll()`, Hibernate lanzaría 1 consulta para los autores más 100 consultas individuales para sus libros (**Problema N+1**). La solución profesional es mantener `LAZY` en la entidad y usar `JOIN FETCH` en JPQL cuando el caso de uso realmente necesite los hijos.

## 5. Query Methods

1. Crea `src/main/java/com/trainer/biblioteca/repository/LibroRepository.java`
	```java
	package com.trainer.biblioteca.repository;

import com.trainer.biblioteca.model.Libro;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface LibroRepository extends JpaRepository<Libro, Long> {

    // Spring Data parsea el nombre del metodo y construye el WHERE automaticamente
    List<Libro> findByTituloContainingIgnoreCase(String palabraClave);
    
    // ¿cómo escribir la firma del método en `LibroRepository` para buscar libros publicados **a partir de un año dado** (`GreaterThanEqual`)?
    List<Libro> findByAnioPublicacionGreaterThanEqual(Integer anio);
}
	```
2. Ejecuta `./mvnw spring-boot:run`
Muestra la traducción de los nombres de método a SQL (`where upper(l1_0.titulo) like upper(?) escape '\'` y `where l1_0.anio_publicacion>=?`)
 
## Resume tres reglas de oro de producción para cerrar con nivel senior:

1. **Nunca expongas entidades JPA directamente en controladores REST** (evita recursión infinita en JSON y exposición accidental de datos; usa DTOs o proyecciones de Spring Data).
2. **En producción nunca uses `ddl-auto=create-drop` ni `update`**; usa `validate` acompañado de herramientas de migración versionada como **Flyway** o **Liquibase**.
3. **Usa `@Transactional(readOnly = true)`** en consultas de solo lectura para desactivar el rastreo de _Dirty Checking_ y ahorrar memoria en el _Persistence Context_.
