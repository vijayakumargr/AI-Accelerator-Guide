# Java Instructions

## General Principles
- **ALWAYS follow Java naming conventions** (Oracle Code Conventions)
- **ALWAYS use meaningful names** for classes, methods, and variables
- **ALWAYS prefer composition over inheritance**
- **ALWAYS close resources** using try-with-resources
- **NEVER catch and ignore exceptions** silently
- **NEVER use raw types** - always specify generic parameters

---

# Code Style

## Naming Conventions
```java
// Classes: PascalCase, nouns
public class UserAccount { }
public class OrderService { }

// Interfaces: PascalCase, often adjectives or nouns
public interface Serializable { }
public interface UserRepository { }

// Methods: camelCase, verbs
public void calculateTotal() { }
public User findById(Long id) { }

// Variables: camelCase
private String userName;
private int orderCount;

// Constants: UPPER_SNAKE_CASE
public static final int MAX_RETRY_COUNT = 3;
public static final String DEFAULT_CHARSET = "UTF-8";

// Packages: lowercase, dot-separated
package com.company.project.service;
```

## Class Structure
```java
public class UserService {
    // 1. Static fields
    private static final Logger LOG = LoggerFactory.getLogger(UserService.class);

    // 2. Instance fields
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    // 3. Constructors
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = Objects.requireNonNull(userRepository);
        this.passwordEncoder = Objects.requireNonNull(passwordEncoder);
    }

    // 4. Public methods
    public User createUser(UserCreateRequest request) {
        // ...
    }

    // 5. Private methods
    private void validateEmail(String email) {
        // ...
    }
}
```

## Modern Java Features (17+)
```java
// Records for immutable data
public record User(Long id, String name, String email) { }

// Sealed classes for controlled inheritance
public sealed interface Shape permits Circle, Rectangle, Triangle { }

// Pattern matching
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Switch expressions
String result = switch (status) {
    case ACTIVE -> "User is active";
    case INACTIVE -> "User is inactive";
    case PENDING -> "User is pending";
};

// Text blocks
String json = """
    {
        "name": "%s",
        "email": "%s"
    }
    """.formatted(name, email);
```

---

# Error Handling

## Exception Patterns
```java
// GOOD: Specific exception handling
public User findUser(Long id) {
    try {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    } catch (DataAccessException e) {
        LOG.error("Database error while finding user {}", id, e);
        throw new ServiceException("Failed to find user", e);
    }
}

// BAD: Catching generic Exception
try {
    // ...
} catch (Exception e) {
    // Don't do this - too broad
}

// BAD: Empty catch block
try {
    // ...
} catch (IOException e) {
    // Silent failure - never do this
}
```

## Custom Exceptions
```java
// Base exception for domain
public class DomainException extends RuntimeException {
    public DomainException(String message) {
        super(message);
    }

    public DomainException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Specific exceptions
public class UserNotFoundException extends DomainException {
    private final Long userId;

    public UserNotFoundException(Long userId) {
        super("User not found: " + userId);
        this.userId = userId;
    }

    public Long getUserId() {
        return userId;
    }
}

public class ValidationException extends DomainException {
    private final Map<String, String> errors;

    public ValidationException(Map<String, String> errors) {
        super("Validation failed");
        this.errors = Map.copyOf(errors);
    }

    public Map<String, String> getErrors() {
        return errors;
    }
}
```

---

# Project Structure

## Maven/Gradle Layout
```
project/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/project/
│   │   │       ├── Application.java
│   │   │       ├── config/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── model/
│   │   │       │   ├── entity/
│   │   │       │   └── dto/
│   │   │       └── exception/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   └── test/
│       ├── java/
│       └── resources/
├── pom.xml (or build.gradle)
└── README.md
```

## Package Organization
```java
// Domain-driven package structure (preferred)
com.company.project.user
    ├── User.java           // Entity
    ├── UserService.java    // Business logic
    ├── UserRepository.java // Data access
    ├── UserController.java // API endpoint
    └── UserDto.java        // Data transfer object

// Layer-based (alternative)
com.company.project
    ├── controller/
    ├── service/
    ├── repository/
    └── model/
```

---

# Spring Boot Patterns

## REST Controller
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(UserResponse::from)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody UserCreateRequest request) {
        User user = userService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(user.getId())
            .toUri();
        return ResponseEntity.created(location).body(UserResponse.from(user));
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }
}
```

## Service Layer
```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final ApplicationEventPublisher eventPublisher;

    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }

    @Transactional
    public User create(UserCreateRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new ValidationException(Map.of("email", "Email already exists"));
        }

        User user = User.builder()
            .name(request.name())
            .email(request.email())
            .passwordHash(passwordEncoder.encode(request.password()))
            .build();

        User saved = userRepository.save(user);
        eventPublisher.publishEvent(new UserCreatedEvent(saved));
        return saved;
    }
}
```

---

# Testing

## Unit Tests
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Test
    void createUser_withValidData_shouldSucceed() {
        // Arrange
        var request = new UserCreateRequest("John", "john@example.com", "password");
        when(userRepository.existsByEmail(anyString())).thenReturn(false);
        when(passwordEncoder.encode(anyString())).thenReturn("hashedPassword");
        when(userRepository.save(any(User.class))).thenAnswer(inv -> {
            User user = inv.getArgument(0);
            return user.toBuilder().id(1L).build();
        });

        // Act
        User result = userService.create(request);

        // Assert
        assertThat(result.getId()).isNotNull();
        assertThat(result.getName()).isEqualTo("John");
        verify(userRepository).save(any(User.class));
    }

    @Test
    void createUser_withDuplicateEmail_shouldThrow() {
        // Arrange
        var request = new UserCreateRequest("John", "existing@example.com", "password");
        when(userRepository.existsByEmail("existing@example.com")).thenReturn(true);

        // Act & Assert
        assertThatThrownBy(() -> userService.create(request))
            .isInstanceOf(ValidationException.class)
            .satisfies(e -> {
                var ve = (ValidationException) e;
                assertThat(ve.getErrors()).containsKey("email");
            });

        verify(userRepository, never()).save(any());
    }
}
```

## Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void createUser_shouldReturnCreated() throws Exception {
        var request = Map.of(
            "name", "John",
            "email", "john@example.com",
            "password", "securePassword123"
        );

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }
}
```

---

# Security

## Input Validation
```java
// Using Bean Validation
public record UserCreateRequest(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8, max = 100) String password
) { }

// Custom validator
@Constraint(validatedBy = PasswordValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface StrongPassword {
    String message() default "Password must contain uppercase, lowercase, digit, and special char";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

## SQL Injection Prevention
```java
// GOOD: Using JPA/Hibernate with parameters
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// GOOD: Using Criteria API
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<User> query = cb.createQuery(User.class);
Root<User> root = query.from(User.class);
query.where(cb.equal(root.get("email"), email));

// BAD: String concatenation (SQL injection vulnerable)
String sql = "SELECT * FROM users WHERE email = '" + email + "'"; // NEVER
```

---

# Anti-Patterns to Avoid

```java
// BAD: Raw types
List users = new ArrayList(); // Avoid

// GOOD: Parameterized types
List<User> users = new ArrayList<>();

// BAD: Null returns
public User findUser(Long id) {
    return null; // Avoid returning null
}

// GOOD: Use Optional
public Optional<User> findUser(Long id) {
    return userRepository.findById(id);
}

// BAD: Checked exception abuse
public void process() throws Exception { } // Too broad

// GOOD: Specific or runtime exceptions
public void process() throws ProcessingException { }

// BAD: Mutable data exposed
public List<String> getItems() {
    return items; // Exposes internal state
}

// GOOD: Return defensive copy or unmodifiable
public List<String> getItems() {
    return List.copyOf(items);
}
```
