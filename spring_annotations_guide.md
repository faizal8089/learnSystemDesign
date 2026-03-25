# 🌱 Spring Boot Annotations — The Complete Interview Guide

> **ELI10 Style:** Every annotation is explained like you're 10 years old, followed by real code you'd actually write at work.

---

## 📦 TABLE OF CONTENTS

1. [Core / IoC Annotations](#1-core--ioc-annotations)
2. [Web / REST Annotations](#2-web--rest-annotations)
3. [Data / JPA Annotations](#3-data--jpa-annotations)
4. [Security Annotations](#4-security-annotations)
5. [Configuration Annotations](#5-configuration-annotations)
6. [Testing Annotations](#6-testing-annotations)
7. [Validation Annotations](#7-validation-annotations)
8. [Bonus: Interview Cheat Sheet](#8-bonus-interview-cheat-sheet)

---

## 1. Core / IoC Annotations

---

### `@SpringBootApplication`

**ELI10:** This is the "START" button of your entire app. You put it on one class and Spring wakes up, looks around, and figures out everything else on its own.

**What it does internally:** It's actually 3 annotations combined:
- `@Configuration` — this class has config
- `@EnableAutoConfiguration` — auto-setup everything
- `@ComponentScan` — scan this package for classes

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

**Interview Q:** *"What is @SpringBootApplication and what annotations does it combine?"*

---

### `@Component`

**ELI10:** You're telling Spring "Hey, remember this class. I might need it later." Spring puts it in its bag (the IoC container) so it can hand it to you when you ask.

```java
@Component
public class EmailValidator {
    public boolean isValid(String email) {
        return email.contains("@");
    }
}
```

**Key point:** `@Service`, `@Repository`, `@Controller` are all just specialized versions of `@Component`. They all do the same thing but tell you (and Spring) *what role* the class plays.

---

### `@Service`

**ELI10:** Same as `@Component`, but you put this on classes that do the *business logic* — like calculating a bill or sending an OTP. It's a label that says "this is where the real work happens."

```java
@Service
public class OrderService {
    public double calculateTotal(List<Item> items) {
        return items.stream()
                    .mapToDouble(Item::getPrice)
                    .sum();
    }
}
```

---

### `@Repository`

**ELI10:** You put this on classes that talk to the database. Spring also gives these classes a special power: it automatically converts database errors into Spring's own error types so your app handles them nicely.

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

**Interview Q:** *"What extra benefit does @Repository give over @Component?"*  
**Answer:** Exception translation — converts SQL/JPA exceptions to Spring's `DataAccessException`.

---

### `@Controller`

**ELI10:** This is the front door of your app. When a request comes in from the browser or Postman, the `@Controller` class is the first one to answer the door.

```java
@Controller
public class HomeController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("name", "Faizal");
        return "home"; // returns home.html (Thymeleaf)
    }
}
```

---

### `@RestController`

**ELI10:** Same as `@Controller`, but this one always speaks in JSON (or plain text), not HTML pages. Every method automatically sends back data, not a web page. Used for building APIs.

**What it does internally:** It's `@Controller` + `@ResponseBody` combined.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return new User(id, "Faizal", "faizal@example.com");
    }
}
```

**Interview Q:** *"Difference between @Controller and @RestController?"*  
**Answer:** `@RestController` adds `@ResponseBody` to every method, so responses are serialized as JSON instead of resolving a view template.

---

### `@Autowired`

**ELI10:** You're telling Spring: "I need this object. Please create it and give it to me — I don't want to do `new SomeClass()` myself." Spring finds the right object from its bag and hands it over.

```java
@Service
public class OrderService {

    @Autowired  // Spring injects UserRepository automatically
    private UserRepository userRepository;

    public User findUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

> ⚠️ **Interview Tip:** In modern Spring Boot, prefer **constructor injection** over field injection. It's easier to test and makes dependencies obvious.

```java
// ✅ Preferred: Constructor Injection
@Service
public class OrderService {

    private final UserRepository userRepository;

    public OrderService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

---

### `@Qualifier`

**ELI10:** Imagine Spring has TWO objects of the same type in its bag. If you just say `@Autowired`, Spring gets confused. `@Qualifier` is like saying "Give me *specifically* the red one, not the blue one."

```java
@Component("smsSender")
public class SmsSender implements NotificationSender { ... }

@Component("emailSender")
public class EmailSender implements NotificationSender { ... }

@Service
public class AlertService {

    @Autowired
    @Qualifier("emailSender")  // be specific!
    private NotificationSender sender;
}
```

---

### `@Primary`

**ELI10:** If Spring finds multiple beans of the same type and you don't use `@Qualifier`, it would crash. `@Primary` says "when in doubt, pick me as the default."

```java
@Component
@Primary  // this one wins by default
public class EmailSender implements NotificationSender { ... }
```

---

### `@Bean`

**ELI10:** `@Component` works on *your own* classes. But what if you want to put a third-party class (like a class from a library) into Spring's bag? That's where `@Bean` helps — you write a method that creates and returns the object, and Spring stores it.

```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        return mapper;
    }
}
```

**Interview Q:** *"Difference between @Component and @Bean?"*  
**Answer:** `@Component` is class-level (auto-detected), `@Bean` is method-level inside a `@Configuration` class (manual, used for third-party classes).

---

### `@Scope`

**ELI10:** By default, Spring creates only ONE copy of each object (like a shared pencil in the classroom). `@Scope` lets you change that — you can make Spring create a fresh copy every time someone asks.

```java
@Component
@Scope("prototype")  // new object every time
public class ReportGenerator { ... }

// Default is "singleton" — one shared instance
```

| Scope | Meaning |
|---|---|
| `singleton` | One instance for whole app (default) |
| `prototype` | New instance every time |
| `request` | One per HTTP request (web apps) |
| `session` | One per user session (web apps) |

---

### `@Value`

**ELI10:** You don't want to hardcode things like passwords, API keys, or URLs in your code. `@Value` lets you pull those values from your `application.properties` file into your code automatically.

```properties
# application.properties
app.name=MyShop
app.max-retries=3
```

```java
@Component
public class AppSettings {

    @Value("${app.name}")
    private String appName;

    @Value("${app.max-retries}")
    private int maxRetries;
}
```

---

## 2. Web / REST Annotations

---

### `@RequestMapping`

**ELI10:** This is the address of your class or method. It tells Spring "requests that come to this URL should be handled by me."

```java
@RestController
@RequestMapping("/api/products")  // all methods in this class start with /api/products
public class ProductController {

    @RequestMapping(method = RequestMethod.GET)
    public List<Product> getAll() { ... }
}
```

---

### `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`

**ELI10:** These are shortcuts. Instead of writing `@RequestMapping(method = RequestMethod.GET)` every time, you just write `@GetMapping`. Each one maps to a specific HTTP action.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping              // GET /api/users
    public List<User> getAll() { ... }

    @GetMapping("/{id}")     // GET /api/users/5
    public User getById(@PathVariable Long id) { ... }

    @PostMapping             // POST /api/users
    public User create(@RequestBody User user) { ... }

    @PutMapping("/{id}")     // PUT /api/users/5
    public User update(@PathVariable Long id, @RequestBody User user) { ... }

    @DeleteMapping("/{id}")  // DELETE /api/users/5
    public void delete(@PathVariable Long id) { ... }

    @PatchMapping("/{id}")   // PATCH /api/users/5 (partial update)
    public User patch(@PathVariable Long id, @RequestBody Map<String, Object> updates) { ... }
}
```

---

### `@PathVariable`

**ELI10:** When the URL has something like `/users/5`, the `5` is a variable part. `@PathVariable` pulls that `5` out of the URL and gives it to your method.

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
// GET /users/42  →  id = 42
```

---

### `@RequestParam`

**ELI10:** When the URL has `?name=faizal&page=2` after it, those are query parameters. `@RequestParam` pulls those out for you.

```java
@GetMapping("/search")
public List<User> search(
    @RequestParam String name,
    @RequestParam(defaultValue = "0") int page
) {
    return userService.search(name, page);
}
// GET /search?name=faizal&page=1
```

---

### `@RequestBody`

**ELI10:** When someone sends data TO your API (like filling a form), the data comes in the body of the request as JSON. `@RequestBody` takes that JSON and automatically converts it into a Java object for you.

```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // user is automatically filled with JSON data
    return userService.save(user);
}

// JSON sent: { "name": "Faizal", "email": "faizal@example.com" }
```

---

### `@ResponseBody`

**ELI10:** When your method returns a Java object, Spring needs to know "should I show an HTML page or send back JSON?" `@ResponseBody` tells it to convert the Java object to JSON and send it back. (Note: `@RestController` already includes this — you rarely need to use it manually.)

```java
@Controller
public class ApiController {

    @GetMapping("/data")
    @ResponseBody  // convert return value to JSON
    public Map<String, String> getData() {
        return Map.of("status", "ok");
    }
}
```

---

### `@ResponseStatus`

**ELI10:** By default, successful responses send HTTP 200. This annotation lets you change what status code gets sent back — like 201 (Created) for a new resource.

```java
@PostMapping("/users")
@ResponseStatus(HttpStatus.CREATED)  // sends 201 instead of 200
public User createUser(@RequestBody User user) {
    return userService.save(user);
}
```

---

### `@ExceptionHandler`

**ELI10:** When something goes wrong (like a user not found), instead of your app crashing with a scary error, this annotation catches the error in that controller and gives back a clean, friendly response.

```java
@RestController
public class UserController {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(UserNotFoundException ex) {
        return Map.of("error", ex.getMessage());
    }
}
```

---

### `@ControllerAdvice` / `@RestControllerAdvice`

**ELI10:** `@ExceptionHandler` only works inside one controller. `@ControllerAdvice` is like a global safety net — it catches errors from **all** controllers in the whole app.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneric(Exception ex) {
        return new ErrorResponse("SERVER_ERROR", "Something went wrong");
    }
}
```

**Interview Q:** *"How do you handle exceptions globally in Spring Boot?"*  
**Answer:** Using `@RestControllerAdvice` with `@ExceptionHandler` methods.

---

### `@CrossOrigin`

**ELI10:** Browsers have a rule — a webpage from `abc.com` can't call an API at `xyz.com` by default (security rule). `@CrossOrigin` tells your API "it's okay, let requests from these websites through."

```java
@CrossOrigin(origins = "http://localhost:3000")
@RestController
@RequestMapping("/api")
public class ProductController { ... }
```

---

## 3. Data / JPA Annotations

---

### `@Entity`

**ELI10:** This tells Spring (and JPA/Hibernate) "this Java class represents a table in the database." Each object of this class = one row in the table.

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String name;

    @Column(unique = true)
    private String email;

    // getters, setters...
}
```

---

### `@Id` and `@GeneratedValue`

**ELI10:** `@Id` marks the primary key (the unique ID for each row). `@GeneratedValue` tells the database to auto-generate this ID — you don't need to set it yourself.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)  // auto-increment
private Long id;
```

| Strategy | Meaning |
|---|---|
| `IDENTITY` | DB auto-increments (MySQL, PostgreSQL) |
| `SEQUENCE` | Uses a DB sequence (PostgreSQL) |
| `AUTO` | Spring picks the best strategy |
| `UUID` | Generates a UUID string |

---

### `@Column`

**ELI10:** By default, JPA guesses the column name from your field name. `@Column` lets you override that — set the exact column name, if it can be null, max length, etc.

```java
@Column(name = "email_address", nullable = false, unique = true, length = 100)
private String email;
```

---

### `@OneToMany`, `@ManyToOne`, `@OneToOne`, `@ManyToMany`

**ELI10:** These define how two tables are related to each other. Like: one `User` can have many `Orders` — that's `@OneToMany`.

```java
// One User has many Orders
@Entity
public class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}

// Each Order belongs to one User
@Entity
public class Order {
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

---

### `@Transactional`

**ELI10:** When you're doing multiple database operations (save, update, delete), and one of them fails, you want ALL of them to be undone — not just stop halfway. `@Transactional` wraps everything in a "all or nothing" deal.

```java
@Service
public class TransferService {

    @Transactional  // if anything fails, both operations are rolled back
    public void transfer(Long fromId, Long toId, double amount) {
        accountService.debit(fromId, amount);   // subtract from account A
        accountService.credit(toId, amount);    // add to account B
    }
}
```

**Interview Q:** *"What happens if an exception is thrown inside a @Transactional method?"*  
**Answer:** Spring rolls back the transaction for unchecked exceptions (RuntimeException) by default. You can configure it with `rollbackFor`.

---

## 4. Security Annotations

---

### `@EnableWebSecurity`

**ELI10:** This is the switch that turns on Spring Security for your app.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }
}
```

---

### `@PreAuthorize`

**ELI10:** This is a bouncer at the door of your method. Before your method runs, Spring checks "does this person have permission?" If not, they're blocked.

```java
@RestController
public class AdminController {

    @GetMapping("/admin/data")
    @PreAuthorize("hasRole('ADMIN')")  // only admins can call this
    public String adminData() {
        return "Secret admin data";
    }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public void deleteUser(@PathVariable Long id) { ... }
}
```

> Enable it with `@EnableMethodSecurity` on your config class.

---

### `@Secured`

**ELI10:** Similar to `@PreAuthorize` but simpler — just list the roles allowed.

```java
@Secured("ROLE_ADMIN")
public void deleteUser(Long id) { ... }
```

---

## 5. Configuration Annotations

---

### `@Configuration`

**ELI10:** This marks a class as a "settings file" for Spring. Inside it, you define `@Bean` methods. Spring reads this class at startup like a recipe book.

```java
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

### `@EnableAutoConfiguration`

**ELI10:** This tells Spring "figure out what I need based on what's in my project." If you have H2 in your dependencies, Spring auto-configures a database. If you have Spring Web, it auto-configures a web server. Magic!

> Usually you don't write this directly — `@SpringBootApplication` already includes it.

---

### `@ConditionalOnProperty`

**ELI10:** "Only create this bean IF a specific property is set in my config file." Great for feature flags.

```java
@Bean
@ConditionalOnProperty(name = "feature.notifications.enabled", havingValue = "true")
public NotificationService notificationService() {
    return new NotificationService();
}
```

```properties
# application.properties
feature.notifications.enabled=true
```

---

### `@Profile`

**ELI10:** You have different settings for `dev`, `test`, and `prod` environments. `@Profile` lets you say "use THIS bean only in dev mode."

```java
@Bean
@Profile("dev")
public DataSource devDataSource() {
    return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
}

@Bean
@Profile("prod")
public DataSource prodDataSource() {
    // real production DB
}
```

```properties
# application.properties
spring.profiles.active=dev
```

---

### `@ConfigurationProperties`

**ELI10:** Instead of using `@Value` for every single property, this lets you map a whole *group* of properties to a Java class at once. Much cleaner.

```properties
# application.properties
app.mail.host=smtp.gmail.com
app.mail.port=587
app.mail.username=hello@example.com
```

```java
@Configuration
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    private String username;
    // getters + setters
}
```

---

### `@EnableScheduling` + `@Scheduled`

**ELI10:** Want to run a task automatically every 5 minutes? Like a robot that does something on a timer? `@Scheduled` is that robot.

```java
@Configuration
@EnableScheduling
public class ScheduleConfig { }

@Component
public class ReportScheduler {

    @Scheduled(cron = "0 0 8 * * MON-FRI")  // 8 AM every weekday
    public void sendDailyReport() {
        System.out.println("Sending report...");
    }

    @Scheduled(fixedDelay = 5000)  // every 5 seconds
    public void healthCheck() {
        System.out.println("App is healthy!");
    }
}
```

---

### `@Async`

**ELI10:** When a task takes a long time (like sending an email), you don't want the user to wait. `@Async` runs that method in the background on a different thread, so the user gets their response immediately.

```java
@Configuration
@EnableAsync
public class AsyncConfig { }

@Service
public class EmailService {

    @Async  // runs in background thread
    public void sendWelcomeEmail(String to) {
        // this could take 2-3 seconds
        emailClient.send(to, "Welcome!");
    }
}
```

---

## 6. Testing Annotations

---

### `@SpringBootTest`

**ELI10:** Starts the full Spring application during the test — like pressing the START button just for testing purposes.

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Test
    void shouldCreateUser() {
        User user = userService.create(new User("Faizal", "faizal@example.com"));
        assertNotNull(user.getId());
    }
}
```

---

### `@WebMvcTest`

**ELI10:** Only loads the web layer (controllers) for testing — not the database or service layer. Use it to test API endpoints specifically. Much faster than `@SpringBootTest`.

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "Faizal"));

        mockMvc.perform(get("/api/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Faizal"));
    }
}
```

---

### `@MockBean`

**ELI10:** In tests, you don't want to actually call the real database or send real emails. `@MockBean` creates a fake (mock) version of a bean that you control completely.

```java
@MockBean
private UserRepository userRepository;

// Now you control what it returns:
when(userRepository.findById(1L)).thenReturn(Optional.of(new User()));
```

---

### `@DataJpaTest`

**ELI10:** Only loads the database layer for testing. Automatically uses an in-memory H2 database instead of your real one.

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldFindByEmail() {
        userRepository.save(new User("Faizal", "faizal@example.com"));
        Optional<User> found = userRepository.findByEmail("faizal@example.com");
        assertTrue(found.isPresent());
    }
}
```

---

## 7. Validation Annotations

> Add `spring-boot-starter-validation` to your `pom.xml`.

### Common Validation Annotations

**ELI10:** These annotations check if the data someone sends to your API is correct — before your code even runs.

```java
public class UserRequest {

    @NotNull(message = "Name cannot be null")
    @NotBlank(message = "Name cannot be empty")
    @Size(min = 2, max = 50, message = "Name must be 2-50 characters")
    private String name;

    @Email(message = "Must be a valid email")
    @NotBlank
    private String email;

    @Min(value = 18, message = "Must be at least 18")
    @Max(value = 120, message = "Age seems too high")
    private int age;

    @Pattern(regexp = "^\\+?[0-9]{10,13}$", message = "Invalid phone number")
    private String phone;

    @NotNull
    @Future(message = "Date must be in the future")
    private LocalDate appointmentDate;
}
```

### `@Valid` — trigger the validation

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@Valid @RequestBody UserRequest request) {
    // if validation fails, Spring throws MethodArgumentNotValidException automatically
    return ResponseEntity.ok(userService.create(request));
}
```

| Annotation | What it checks |
|---|---|
| `@NotNull` | Field is not null |
| `@NotBlank` | String is not null AND not empty/whitespace |
| `@NotEmpty` | Collection/string is not null AND not empty |
| `@Size` | String/collection has length within range |
| `@Min` / `@Max` | Number is above/below limit |
| `@Email` | Valid email format |
| `@Pattern` | Matches a regex pattern |
| `@Positive` | Number > 0 |
| `@Past` / `@Future` | Date is before/after now |

---

## 8. Bonus: Interview Cheat Sheet

### Most Asked Interview Questions

| Question | Short Answer |
|---|---|
| `@Component` vs `@Bean` | `@Component` = class-level auto-scan; `@Bean` = method-level manual registration |
| `@Controller` vs `@RestController` | `@RestController` = `@Controller` + `@ResponseBody` on every method |
| `@PathVariable` vs `@RequestParam` | `@PathVariable` from URL path (`/users/5`); `@RequestParam` from query string (`?page=1`) |
| `@Autowired` vs Constructor Injection | Constructor injection is preferred — easier to test, explicit dependencies |
| What does `@Transactional` do? | Wraps operations in a DB transaction — rolls back all on failure |
| `@SpringBootApplication` combines what? | `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| How to handle global exceptions? | `@RestControllerAdvice` + `@ExceptionHandler` |
| `@Primary` vs `@Qualifier` | `@Primary` sets default bean; `@Qualifier` picks a specific one by name |
| `@Service` vs `@Repository` | Both are `@Component`; `@Repository` adds exception translation |
| How to validate request body? | `@Valid` on `@RequestBody` + JSR-380 annotations on the DTO |

---

### Annotation Quick Reference Card

```
LAYER         ANNOTATION              PURPOSE
──────────────────────────────────────────────────────────────
Bootstrap     @SpringBootApplication  Start the app
IoC           @Component              Register as a bean
              @Service                Business logic bean
              @Repository             Data access bean
              @Controller             Web MVC controller
              @RestController         REST API controller
              @Autowired              Inject a dependency
              @Qualifier              Pick specific bean
              @Primary                Default bean
              @Bean                   Manual bean method
              @Scope                  Bean lifecycle

Config        @Configuration          Config class
              @Value                  Inject a property value
              @ConfigurationProperties Map property group to class
              @Profile                Environment-specific bean
              @Conditional*           Conditional bean creation

Web           @RequestMapping         Map URL to class/method
              @GetMapping             HTTP GET
              @PostMapping            HTTP POST
              @PutMapping             HTTP PUT
              @DeleteMapping          HTTP DELETE
              @PatchMapping           HTTP PATCH
              @PathVariable           Extract from URL /path/{var}
              @RequestParam           Extract from ?key=value
              @RequestBody            Parse JSON body to object
              @ResponseBody           Serialize return to JSON
              @ResponseStatus         Set HTTP status code
              @ExceptionHandler       Handle specific exceptions
              @RestControllerAdvice   Global exception handler
              @CrossOrigin            Allow CORS requests

Data/JPA      @Entity                 Map class to DB table
              @Table                  Set table name
              @Id                     Primary key
              @GeneratedValue         Auto-generate PK
              @Column                 Map field to column
              @OneToMany              One-to-many relationship
              @ManyToOne              Many-to-one relationship
              @Transactional          DB transaction wrapper

Scheduling    @EnableScheduling       Turn on scheduling
              @Scheduled              Run on timer/cron

Async         @EnableAsync            Turn on async
              @Async                  Run method in background

Security      @EnableWebSecurity      Turn on Spring Security
              @PreAuthorize           Method-level access control

Testing       @SpringBootTest         Full app context test
              @WebMvcTest             Web layer only test
              @DataJpaTest            DB layer only test
              @MockBean               Fake/mock a bean

Validation    @Valid                  Trigger validation
              @NotNull / @NotBlank    Null/empty checks
              @Email / @Pattern       Format checks
              @Min / @Max / @Size     Range checks
```

---

*Happy coding, Faizal! 🚀 — Spring Boot annotations are 80% of your interview.*
