## Spring Boot 

### What is Spring

Spring is a framework for building Java applications. Its core idea is **Inversion of Control (IoC)** — instead of creating and wiring objects yourself, you describe what exists and Spring manages object creation and dependencies.

---

### Application Context & Beans

**Application Context** is Spring's internal registry of objects — a map of all managed instances:

```
Application Context:
{
  UserRepository  → <object>
  UserService     → <object>
  AuthController  → <object>
  ...
}
```

A **Bean** is any object managed by Spring. By default, Spring creates **one instance per bean** (Singleton scope) — the same object is shared everywhere it's needed.

Spring fills the context in two ways:

**1. Class annotations** — Spring scans for these and creates objects automatically:
```java
@Service        // business logic
@Repository     // data access
@RestController // HTTP controller
@Component      // anything else
@Configuration  // configuration class with @Bean methods
```

**2. `@Bean` methods** — for objects that require manual configuration:
```java
@Configuration
public class AppConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

> `@Entity` is not a Spring annotation — it belongs to Hibernate/JPA. DTOs and records are plain Java classes, not beans.

---

### Dependency Injection

When Spring creates a bean, it automatically provides its dependencies by looking up the Application Context by **type**.

```java
@Service
@RequiredArgsConstructor          // Lombok generates the constructor
public class UserService {
    private final UserRepository repo;  // Spring injects this
}
```

- `final` is required for Lombok's `@RequiredArgsConstructor` to include the field
- If Spring finds no matching bean → app crashes at startup with `No qualifying bean of type X`
- If Spring finds multiple matching beans → use `@Qualifier("beanName")` to specify which one

---

### `@Value` — inject config values

```java
@Value("${jwt.secret}")
private String secret;

@Value("${jwt.expiration}")
private long expiration;
```

> Import `org.springframework.beans.factory.annotation.Value`, not `lombok.Value`.

---

### Layers & separation of concerns

```
Controller  →  receive HTTP request, return response
Service     →  business logic, decisions
Repository  →  read/write data from database
```

Each layer does one thing only. Services talk to repositories. Controllers talk to services. Controllers never talk to repositories directly.

---

### REST Controllers

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable UUID id) { ... }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest req) { ... }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(@PathVariable UUID id, @RequestBody CreateUserRequest req) { ... }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable UUID id) { ... }
}
```

**HTTP method conventions (REST):**
- `GET` — read, no body
- `POST` — create, data in body, no ID in URL
- `PUT` — update, ID in URL + data in body
- `DELETE` — delete, ID in URL

**Common HTTP status codes:**
- `200 OK` — success
- `201 Created` — resource created (include `Location` header)
- `204 No Content` — success, nothing to return (delete)
- `400 Bad Request` — invalid input
- `401 Unauthorized` — not authenticated
- `403 Forbidden` — authenticated but not allowed
- `404 Not Found` — resource doesn't exist

**ResponseEntity:**
```java
ResponseEntity.ok(body)                          // 200
ResponseEntity.created(URI.create("/...")).body(x) // 201
ResponseEntity.noContent().build()               // 204
ResponseEntity.notFound().build()                // 404
ResponseEntity.status(200).body(x)              // explicit status
```

---

### DTO (Data Transfer Object)

Separate classes that define what goes in and out of the API — independent of entities.

```
Client → [CreateUserRequest] → Service → [User entity] → DB
DB     → [User entity]       → Service → [UserResponse] → Client
```

Use **Java Records** for DTOs — zero boilerplate:
```java
public record CreateUserRequest(String email, String username, String password) {}
public record UserResponse(UUID id, String email, String username) {}
```

Map between DTO and entity manually in the service:
```java
private User toEntity(CreateUserRequest req) {
    return User.builder().email(req.email()).build();
}

private UserResponse toResponse(User user) {
    return new UserResponse(user.getId(), user.getEmail(), user.getUsername());
}
```

---

### JPA & Hibernate

Hibernate translates between Java objects and database tables. JPA is the specification; Hibernate is the implementation.

**Entity annotations:**
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)   // store as string, not number
    private UserRole role;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

> Always use `@Table(name = "users")` — `user` is a reserved word in PostgreSQL.

**`ddl-auto` values:**
- `create` — drop and recreate tables on every start (dev only, loses data)
- `validate` — check schema matches entities, fail if not
- `none` — do nothing (use with Flyway in production)

---

### Spring Data JPA Repositories

```java
public interface UserRepository extends JpaRepository<User, UUID> {
    Optional<User> findByEmail(String email);  // auto-generated from method name
}
```

`JpaRepository<Entity, IdType>` provides: `save`, `findById`, `findAll`, `deleteById`, and more — inherited from `CrudRepository`.

Method naming generates SQL automatically:
```java
findByEmail(String email)           // WHERE email = ?
findByStatusAndPriority(...)        // WHERE status = ? AND priority = ?
findByDeadlineBefore(LocalDateTime) // WHERE deadline < ?
```

---

### Spring Security & JWT Filter

**SecurityConfig:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http, JwtFilter jwtFilter) throws Exception {
        http
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**JwtFilter:**
```java
@Component
@RequiredArgsConstructor
public class JwtFilter extends OncePerRequestFilter {
    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        String token = authHeader.substring("Bearer ".length());
        if (!jwtService.validateToken(token)) {
            filterChain.doFilter(request, response);
            return;
        }
        String email = jwtService.extractEmail(token);
        UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(email, null, List.of());
        SecurityContextHolder.getContext().setAuthentication(authentication);
        filterChain.doFilter(request, response);
    }
}
```

> Always call `filterChain.doFilter()` at the end — without it the request never reaches the controller.

---

### Builder Pattern

Used when objects have many fields — build step by step instead of a long constructor.

```java
User user = User.builder()
    .email("maciej@example.com")
    .username("maciej")
    .build();
```

`@Builder` from Lombok generates the builder class automatically. Each setter method returns `this` to allow chaining. `.build()` creates the final object.

---

### `Optional<T>`

Wraps a value that may or may not exist — safer than returning `null`.

```java
// map() — apply function if value present, skip if empty
// orElse() — fallback if empty
return userRepository.findById(id)
    .map(ResponseEntity::ok)
    .orElse(ResponseEntity.notFound().build());

// method reference — shorthand for user -> toResponse(user)
return userRepository.findById(id).map(this::toResponse);
```

---

### Lombok annotations

```java
@Getter               // generate getters
@NoArgsConstructor    // empty constructor (required by Hibernate)
@AllArgsConstructor   // constructor with all fields
@RequiredArgsConstructor // constructor for final fields only (use with DI)
@Builder              // builder pattern (requires @AllArgsConstructor when combined with @NoArgsConstructor)
```

---

### application.yml structure

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/taskmanager
    username: postgres
    password: ${DB_PASSWORD:haslo}
  jpa:
    hibernate:
      ddl-auto: create
    show-sql: false

jwt:
  secret: ${JWT_SECRET:local-dev-secret-change-in-production}
  expiration: 86400000

server:
  port: 8080
```

> `jwt` must be at the same level as `spring`, not nested inside it.
> Use `${VAR:default}` syntax — environment variable in production, default value in local dev.
> Use YAML over `.properties` — hierarchical structure is much more readable at scale.

---


Organize by **feature**, not by layer. Everything related to a user lives in `user/`, everything related to tasks in `task/`.
