# Spring Data JPA: Simple Notes + Interview Prep

> Topics: Spring Data JPA basics, JpaRepository, Entity Mapping, Fetching Strategies, JPQL & Native Queries, MySQL setup

---

## 1. Spring Data JPA Basics

### In one line
Spring Data JPA lets you do database CRUD by writing **just an interface**. Spring generates the implementation and the SQL for you.

### Key terms
| Term | Simple meaning |
|---|---|
| **JPA** | Java *specification* (rules) for mapping Java objects to DB tables (ORM) |
| **Hibernate** | The most common *implementation* of JPA |
| **Spring Data JPA** | A layer on top of JPA that removes boilerplate (repositories) |
| **ORM** | Object-Relational Mapping: object <-> table row |
| **Entity** | Java class mapped to a table; each object = one row |

> **Remember:** JPA = rules, Hibernate = the one who follows the rules, Spring Data JPA = makes it easy to use.

### Typical layered flow
```
Client -> Controller -> Service -> Repository -> Database
```

### Minimal example
```java
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private String author;
    private double price;
}

public interface BookRepository extends JpaRepository<Book, Long> { }
```
No SQL and no implementation class needed. `save`, `findById`, `findAll`, `deleteById` are ready to use.

### Query methods (derived queries)
Spring builds the query from the **method name**:
```java
List<Book> findByTitle(String title);
List<Book> findByAuthor(String author);
List<Book> findByTitleContaining(String keyword);
```

### Setup steps (Spring Boot)
1. Add dependencies: `spring-boot-starter-data-jpa` + a DB driver (H2 / MySQL / PostgreSQL)
2. Configure `application.properties`
3. Create Entity
4. Create Repository
5. (Optional) Service layer
6. Controller (REST)

### H2 config (in-memory)
```properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.jpa.hibernate.ddl-auto=update
```

### REST endpoints (typical CRUD)
| Method | URL | Action |
|---|---|---|
| GET | `/books` | Get all |
| POST | `/books` | Create |
| PUT | `/books/{id}` | Update |
| DELETE | `/books/{id}` | Delete |

### Interview Q&A
**Q: What is Spring Data JPA?**
A: A Spring module that provides a repository abstraction over JPA, so we can do CRUD without writing boilerplate or SQL.

**Q: Difference between JPA and Hibernate?**
A: JPA is a specification (a set of interfaces and rules). Hibernate is an implementation of that specification.

**Q: Difference between JPA and Spring Data JPA?**
A: JPA is the standard for ORM. Spring Data JPA is built on top of JPA and removes the need to write DAO/repository implementations.

**Q: What is ORM?**
A: A technique to map Java objects to relational DB tables, so we work with objects instead of SQL.

**Q: What does `ddl-auto=update` do?**
A: Hibernate updates the DB schema to match your entities (adds new tables/columns, never drops existing ones).

| Value | Behavior |
|---|---|
| `none` | Does nothing |
| `validate` | Only checks the schema matches entities |
| `update` | Updates schema, keeps data |
| `create` | Drops and recreates on startup |
| `create-drop` | Create on start, drop on shutdown |

> Use `update` for learning/dev. Use `validate` or `none` in production (with Flyway/Liquibase).

---

## 2. JpaRepository

### Hierarchy
```
Repository
  -> CrudRepository          (basic CRUD)
     -> PagingAndSortingRepository   (+ pagination & sorting)
        -> JpaRepository     (+ JPA-specific methods like flush, batch delete)
```

```java
public interface JpaRepository<T, ID> extends PagingAndSortingRepository<T, ID> { }
```
- `T` = entity type
- `ID` = type of the primary key (e.g. `Long`)

### Common methods
| Method | What it does |
|---|---|
| `save(entity)` | Insert if id is `null`, otherwise update |
| `findById(id)` | Returns `Optional<T>` |
| `findAll()` | Returns all rows as a `List` |
| `deleteById(id)` | Delete by id |
| `delete(entity)` | Delete by entity |
| `count()` | Number of rows |
| `existsById(id)` | true/false |

### Examples
```java
// save (create/update)
Book saved = bookRepository.save(new Book("Spring Boot", "John Doe", 29.99));

// findById returns Optional, so avoid null
Optional<Book> book = bookRepository.findById(1L);
book.ifPresent(b -> System.out.println(b.getTitle()));

// findAll
List<Book> books = bookRepository.findAll();

// delete
bookRepository.deleteById(1L);
```

### Pagination & sorting
```java
// First page, 10 records (page index starts at 0)
Page<Book> page = bookRepository.findAll(PageRequest.of(0, 10));

// Sorting
List<Book> sorted = bookRepository.findAll(Sort.by("title").ascending());

// Both together
Page<Book> sortedPage =
    bookRepository.findAll(PageRequest.of(0, 5, Sort.by("price").descending()));
```

### Custom queries
```java
List<Book> findByAuthor(String author);
List<Book> findByTitleContaining(String keyword);

@Query("SELECT b FROM Book b WHERE b.price > :price")
List<Book> findBooksCostlierThan(@Param("price") double price);
```

### Full CRUD flow
**Service**
```java
@Service
public class BookService {
    @Autowired private BookRepository bookRepository;

    public List<Book> getAllBooks() { return bookRepository.findAll(); }

    public Book getBookById(Long id) {
        return bookRepository.findById(id)
               .orElseThrow(() -> new RuntimeException("Book not found"));
    }

    public Book createOrUpdateBook(Book book) { return bookRepository.save(book); }

    public void deleteBook(Long id) { bookRepository.deleteById(id); }
}
```

**Controller**
```java
@RestController
@RequestMapping("/books")
public class BookController {
    @Autowired private BookService bookService;

    @GetMapping            public List<Book> getAll() { return bookService.getAllBooks(); }
    @GetMapping("/{id}")   public Book getById(@PathVariable Long id) { return bookService.getBookById(id); }
    @PostMapping           public Book create(@RequestBody Book b) { return bookService.createOrUpdateBook(b); }

    @PutMapping("/{id}")
    public Book update(@PathVariable Long id, @RequestBody Book b) {
        b.setId(id);   // important: an id makes save() do an UPDATE
        return bookService.createOrUpdateBook(b);
    }

    @DeleteMapping("/{id}") public void delete(@PathVariable Long id) { bookService.deleteBook(id); }
}
```

### Interview Q&A
**Q: Difference between `CrudRepository` and `JpaRepository`?**
A: `CrudRepository` has basic CRUD. `JpaRepository` adds pagination/sorting (via `PagingAndSortingRepository`) and JPA-specific features like `flush()` and batch operations. It also returns `List` instead of `Iterable` from `findAll()`.

**Q: How does `save()` know whether to insert or update?**
A: If the id is `null` (new entity), it inserts. If the id exists, it updates (merge).

**Q: Why does `findById` return `Optional`?**
A: The record may not exist. `Optional` forces you to handle the "not found" case and avoids `NullPointerException`.

**Q: `findById` vs `getReferenceById`?**
A: `findById` hits the DB immediately and returns `Optional`. `getReferenceById` returns a lazy proxy and hits the DB only when you access its fields.

**Q: Page index starts at 0 or 1?**
A: 0.

**Q: How do you write a custom query in a repository?**
A: Either by method naming (`findByAuthor`) or with `@Query` (JPQL or native).

**Q: Do we need `@Repository` on a JpaRepository interface?**
A: Not required. Spring Data detects it automatically.

---

## 3. Entity Mapping
Entity mapping is the process of mapping a Java object (an entity) to a corresponding database table using annotations. Each field in the entity class is mapped to a column in the database, and relationships between entities are mapped

### Core annotations
| Annotation | Purpose |
|---|---|
| `@Entity` | Marks class as a JPA entity (maps to a table) |
| `@Id` | Primary key (**mandatory**) |
| `@GeneratedValue` | Auto-generate the id |
| `@Column` | Customize column (name, length, nullable...) |
| `@Table` | Customize table name |

```java
@Entity
@Table(name = "library_books")        // table name instead of default "Book"
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "book_title", length = 100, nullable = false)
    private String title;

    @Column(nullable = false)
    private String author;
}
```

> Default table name = class name. Use `@Table` to change it.
> `@Column` defaults: column name = field name, nullable = true, length = 255.

### `GenerationType` options
| Strategy | Meaning |
|---|---|
| `IDENTITY` | DB auto-increment (MySQL, most common) |
| `SEQUENCE` | Uses a DB sequence (PostgreSQL, Oracle) |
| `TABLE` | Uses a separate table to generate ids (rare, slow) |
| `AUTO` | Provider picks the strategy |

### Relationships

| Annotation | Example |
|---|---|
| `@OneToOne` | Author <-> Address |
| `@OneToMany` | One Author -> many Books |
| `@ManyToOne` | Many Books -> one Author |
| `@ManyToMany` | Books <-> Categories |

#### OneToOne
```java
@Entity
public class Author {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne
    private Address address;
}
```

#### OneToMany / ManyToOne (the most common pair)
```java
@Entity
public class Author {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
    private List<Book> books;
}

@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToOne
    @JoinColumn(name = "author_id")     // foreign key column in the book table
    private Author author;
}
```
- The **`@ManyToOne` side owns the relationship** (it holds the foreign key `author_id`).
- `mappedBy = "author"` means "the `author` field in `Book` manages this relationship".

#### ManyToMany
```java
@Entity
public class Book {
    @ManyToMany
    @JoinTable(
        name = "book_category",
        joinColumns = @JoinColumn(name = "book_id"),
        inverseJoinColumns = @JoinColumn(name = "category_id"))
    private Set<Category> categories;
}

@Entity
public class Category {
    @ManyToMany(mappedBy = "categories")
    private Set<Book> books;
}
```
A **join table** (`book_category`) is created with two foreign keys.

### Owning side vs inverse side
| | Owning side | Inverse side |
|---|---|---|
| Has the FK / join table | Yes | No |
| Uses `mappedBy` | No | Yes |
| Example | `Book.author` | `Author.books` |

### Cascade
Cascade = "do the same operation on the children automatically".

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
private List<Book> books;
```

| Type | Effect |
|---|---|
| `PERSIST` | Saving parent also saves children |
| `MERGE` | Updating parent also updates children |
| `REMOVE` | Deleting parent also deletes children |
| `REFRESH` | Refreshing parent also refreshes children |
| `DETACH` | Detaching parent also detaches children |
| `ALL` | All of the above |

**Why use it?** Without cascade, you must save the Author, then save each Book. With `CascadeType.ALL`, saving the Author saves all its Books.

**Careful:** `REMOVE`/`ALL` on a parent deletes all its children. Use only when children can't exist without the parent.

> Related option: `orphanRemoval = true` deletes children that are removed from the parent's collection.

### Interview Q&A
**Q: Is `@Id` mandatory?**
A: Yes, every entity must have a primary key.

**Q: What does `mappedBy` do?**
A: It marks the inverse (non-owning) side of a bidirectional relationship and points to the field on the owning side. No extra FK or join table is created for it.

**Q: Who owns the relationship in OneToMany / ManyToOne?**
A: The `@ManyToOne` side, because the foreign key lives in its table.

**Q: Why does ManyToMany need a join table?**
A: A single FK column can't hold multiple references, so a third table stores the pairs (`book_id`, `category_id`).

**Q: What is `@JoinColumn`?**
A: It names the foreign key column in the owning entity's table.

**Q: What is `CascadeType.ALL`? Any risk?**
A: It propagates all operations (persist, merge, remove, refresh, detach) from parent to children. The risk is accidental deletion of related data.

**Q: Cascade vs fetch?**
A: Cascade controls what *operations* pass to related entities. Fetch controls *when related data is loaded*.

**Q: Default table name if no `@Table`?**
A: The entity class name (Hibernate with Spring Boot usually converts `BookDetails` to `book_details`).

**Q: `@Column(nullable = false)` vs `@NotNull`?**
A: `@Column(nullable=false)` affects the DB schema. `@NotNull` is bean validation, checked in Java before persisting. Use both for safety.

**Q: Common bidirectional mapping pitfall in REST?**
A: Infinite recursion in JSON (Author -> Books -> Author...). Fix with `@JsonIgnore`, `@JsonManagedReference`/`@JsonBackReference`, or return DTOs (best practice).

---

## 4. Fetching Strategies (Eager vs Lazy)

### Definition
Fetching strategy decides **when related entities are loaded** from the DB.

| | Eager | Lazy |
|---|---|---|
| Annotation | `FetchType.EAGER` | `FetchType.LAZY` |
| When loaded | Immediately with the parent | Only when first accessed |
| Pro | Fewer queries if you always need the data | Better performance, loads only what's needed |
| Con | Loads unnecessary data | Can cause the **N+1 problem** and `LazyInitializationException` |

```java
// Eager: books always loaded with the author
@OneToMany(fetch = FetchType.EAGER)
private List<Book> books;

// Lazy: books loaded only when you call author.getBooks()
@OneToMany(fetch = FetchType.LAZY)
private List<Book> books;
```

### Default fetch types (very common interview question)
| Relationship | Default |
|---|---|
| `@OneToOne` | **EAGER** |
| `@ManyToOne` | **EAGER** |
| `@OneToMany` | **LAZY** |
| `@ManyToMany` | **LAZY** |

> Trick: anything ending in **"One"** on the target side is EAGER; anything ending in **"Many"** is LAZY.

### The N+1 select problem
```java
List<Author> authors = entityManager
    .createQuery("SELECT a FROM Author a").getResultList();   // 1 query

for (Author author : authors) {
    System.out.println(author.getBooks());   // +1 query per author (N queries)
}
```
Total = **1 + N queries** (1 for authors + N for each author's books). Very slow for large data.

### How to fix N+1
1. **`JOIN FETCH`** in JPQL:
   ```java
   @Query("SELECT a FROM Author a JOIN FETCH a.books")
   List<Author> findAllWithBooks();
   ```
2. **`@EntityGraph`**:
   ```java
   @EntityGraph(attributePaths = "books")
   List<Author> findAll();
   ```
3. **`@BatchSize`** (loads children in batches instead of one by one).
4. Use **DTO projections** when you don't need full entities.

### Choosing the right one
- **Eager**: you *always* need the related data (e.g. User + Profile).
- **Lazy**: related data is large or not always needed (e.g. Order -> OrderItems).
- **Best practice:** keep everything `LAZY` and fetch explicitly (`JOIN FETCH` / `@EntityGraph`) when needed.

### Interview Q&A
**Q: Eager vs Lazy?**
A: Eager loads related entities together with the parent. Lazy loads them only when accessed.

**Q: What is the N+1 problem and how do you solve it?**
A: One query for the parent list plus one query per parent for its lazy children, giving N+1 queries. Solve it with `JOIN FETCH`, `@EntityGraph`, or batch fetching.

**Q: What is `LazyInitializationException`?**
A: Thrown when you access a lazy association after the session/transaction is closed (e.g. in the controller). Fix by fetching it inside the transaction (`JOIN FETCH`, `@Transactional`, or DTOs).

**Q: Default fetch type for `@ManyToOne`? For `@OneToMany`?**
A: `@ManyToOne` = EAGER, `@OneToMany` = LAZY.

**Q: Which is the better default and why?**
A: LAZY, because EAGER can pull in huge object graphs and hurt performance. Fetch explicitly when needed.

**Q: Is lazy loading guaranteed?**
A: `LAZY` is a hint that Hibernate normally respects. `EAGER` is a requirement.

---

## 5. JPQL and Native Queries

### JPQL
**JPQL (Java Persistence Query Language)** is like SQL but works on **entities and their fields**, not tables and columns. It is database-independent.

```java
// Keyword search
@Query("SELECT b FROM Book b WHERE b.title LIKE %:keyword%")
List<Book> searchBooksByTitleKeyword(@Param("keyword") String keyword);

// Multiple conditions
@Query("SELECT b FROM Book b WHERE b.author = :author AND b.price < :maxPrice")
List<Book> findBooksByAuthorAndPrice(@Param("author") String author,
                                     @Param("maxPrice") Double maxPrice);

// Ordering
@Query("SELECT b FROM Book b ORDER BY b.title ASC")
List<Book> findAllBooksOrderedByTitle();
```
- `Book` = entity name (not table name)
- `b.title` = field name (not column name)
- `:keyword` = named parameter bound with `@Param`

### Native queries
Plain **SQL**, run directly on the DB. Add `nativeQuery = true`.

```java
// Simple
@Query(value = "SELECT * FROM book WHERE title LIKE %:keyword%", nativeQuery = true)
List<Book> searchBooksByTitleNative(@Param("keyword") String keyword);

// Join
@Query(value = "SELECT b.* FROM book b INNER JOIN author a ON b.author_id = a.id WHERE a.name = :authorName",
       nativeQuery = true)
List<Book> findBooksByAuthorNative(@Param("authorName") String authorName);

// Price range
@Query(value = "SELECT * FROM book WHERE price BETWEEN :minPrice AND :maxPrice", nativeQuery = true)
List<Book> findBooksByPriceRangeNative(@Param("minPrice") Double minPrice,
                                       @Param("maxPrice") Double maxPrice);

// Aggregation: returns raw rows
@Query(value = "SELECT author, COUNT(*) FROM book GROUP BY author", nativeQuery = true)
List<Object[]> findAuthorBookCount();
```
Here `book` = table name, `title`/`price` = column names.

### JPQL vs Native
| | JPQL | Native SQL |
|---|---|---|
| Works on | Entities & fields | Tables & columns |
| Portable across DBs | Yes | No (DB-specific) |
| Use for | Simple to moderately complex queries | Complex queries, DB-specific functions, performance tuning |
| Result mapping | Automatic to entities | Entities, `Object[]`, or projections |

### Three ways to query (summary)
1. **Derived query methods**: `findByAuthor(String a)` (simplest)
2. **JPQL**: `@Query("SELECT ...")`
3. **Native SQL**: `@Query(value="...", nativeQuery=true)`

### Interview Q&A
**Q: What is JPQL?**
A: An object-oriented query language that operates on entities rather than tables. It is database independent.

**Q: JPQL vs SQL?**
A: JPQL uses entity/field names and is portable. SQL uses table/column names and is DB-specific.

**Q: When do you use native queries?**
A: For complex queries JPQL can't express, DB-specific functions, or performance-critical queries.

**Q: What does `@Param` do?**
A: Binds a method parameter to a named parameter (`:name`) in the query.

**Q: How do you avoid SQL injection with `@Query`?**
A: Always use parameter binding (`:param` / `?1`), never string concatenation.

**Q: What does a query returning `List<Object[]>` mean?**
A: Each row is an array of column values. Better to use an interface/DTO projection for readability.

**Q: Derived query vs `@Query`?**
A: Derived queries are quick for simple conditions. `@Query` is better when method names get too long or logic is complex.

---

## 6. Connecting Spring Boot to MySQL

### Steps
1. Install MySQL and verify it works.
2. Create a database (e.g. `springbootdb`).
3. Add dependencies: `spring-boot-starter-data-jpa` and `mysql-connector-j`.
4. Configure `application.properties`.
5. Create Entity, Repository, Controller.
6. Run and test with Postman/cURL.

### Dependencies (Maven)
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

### `application.properties`
```properties
spring.datasource.url=jdbc:mysql://localhost:3306/springbootdb
spring.datasource.username=root
spring.datasource.password=yourpassword

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```
| Property | Meaning |
|---|---|
| `datasource.url` | MySQL URL: host, port 3306, DB name |
| `username` / `password` | DB credentials |
| `ddl-auto=update` | Auto-update schema from entities |
| `show-sql=true` | Print SQL in the console |

### Entity
```java
import jakarta.persistence.*;   // Spring Boot 3+ (use javax.persistence for Boot 2.x)

@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String course;

    public Student() {}                       // JPA needs a no-arg constructor
    public Student(String name, String course) {
        this.name = name;
        this.course = course;
    }
    // getters and setters
}
```

### Repository
```java
public interface StudentRepository extends JpaRepository<Student, Long> { }
```

### Controller
```java
@RestController
@RequestMapping("/students")
public class StudentController {

    @Autowired
    private StudentRepository studentRepository;

    @PostMapping
    public Student createStudent(@RequestBody Student student) {
        return studentRepository.save(student);
    }

    @GetMapping
    public List<Student> getAllStudents() {
        return studentRepository.findAll();
    }
}
```

### Test
`POST /students`
```json
{ "name": "John Doe", "course": "Computer Science" }
```
`GET /students` returns all students, and the row appears in the MySQL `student` table.

### Interview Q&A
**Q: Which dependencies do you need for MySQL?**
A: `spring-boot-starter-data-jpa` and the MySQL driver (`mysql-connector-j`).

**Q: How does Spring Boot know which DB to connect to?**
A: From `spring.datasource.url` and the driver on the classpath. Boot auto-configures the `DataSource` and `EntityManagerFactory`.

**Q: `javax.persistence` vs `jakarta.persistence`?**
A: Spring Boot 2.x uses `javax.*`. Spring Boot 3+ (Jakarta EE 9+) uses `jakarta.*`.

**Q: Why does a JPA entity need a no-arg constructor?**
A: Hibernate creates entity objects via reflection and needs a default constructor.

**Q: Why not use `ddl-auto=update` in production?**
A: It can alter the schema unexpectedly. Use migration tools like Flyway/Liquibase instead.

**Q: How do you see the generated SQL?**
A: `spring.jpa.show-sql=true` (and optionally `spring.jpa.properties.hibernate.format_sql=true`).

**Q: Don't hardcode the DB password. What's the alternative?**
A: Use environment variables, profile-specific configs, or a secrets manager.

---

## Quick Revision Cheat Sheet

- **JPA** = spec, **Hibernate** = implementation, **Spring Data JPA** = abstraction on top.
- `JpaRepository<T, ID>` gives CRUD, pagination, sorting for free.
- `save()` = insert if id is null, else update.
- `findById()` returns `Optional`.
- `@Entity` + `@Id` are mandatory. `@Table`/`@Column` are optional customizations.
- `mappedBy` = inverse side. `@ManyToOne` side holds the FK.
- `ManyToMany` needs a join table (`@JoinTable`).
- `CascadeType.ALL` = propagate all operations. Be careful with `REMOVE`.
- Defaults: `ToOne` = EAGER, `ToMany` = LAZY.
- N+1 fix: `JOIN FETCH`, `@EntityGraph`, `@BatchSize`.
- JPQL = entity-based and portable. Native = SQL-based and DB-specific.
- Always bind query parameters; never concatenate strings.
- MySQL: driver + datasource URL/user/password + `ddl-auto`.

## Top 10 Rapid-Fire Interview Questions

1. What is the difference between JPA, Hibernate, and Spring Data JPA?
2. Explain `CrudRepository` vs `JpaRepository`.
3. How does `save()` decide between insert and update?
4. What are the relationship annotations in JPA?
5. What does `mappedBy` do, and who owns the relationship?
6. What is `CascadeType.ALL`?
7. Eager vs Lazy: defaults for each relationship?
8. What is the N+1 problem and how do you fix it?
9. JPQL vs Native query: when do you use each?
10. What does `spring.jpa.hibernate.ddl-auto` do, and which value for production?
