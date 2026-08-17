# 🍃 Spring Boot

> Spring is an inversion-of-control container with a very large ecosystem bolted onto it. Boot is auto-configuration on top. Understand the container and the proxy mechanism, and everything else follows.

---

## 📑 Table of Contents

1. [Mental Model](#1-mental-model)
2. [The IoC Container](#2-ioc-container)
3. [Bean Lifecycle and Scopes](#3-bean-lifecycle)
4. [Auto-Configuration](#4-auto-configuration)
5. [AOP and Proxies](#5-aop-and-proxies)
6. [Web Layer](#6-web-layer)
7. [Data Access](#7-data-access)
8. [Transactions](#8-transactions)
9. [Validation and Error Handling](#9-validation-errors)
10. [Spring Security](#10-security)
11. [Configuration and Profiles](#11-configuration)
12. [Testing](#12-testing)
13. [Reactive Spring](#13-reactive)
14. [Observability](#14-observability)
15. [Production](#15-production)
16. [Interview Section](#16-interview-section)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. Mental Model

```
   ┌────────────────────────────────────────────────────────────────┐
   │                      SPRING BOOT                               │
   │                                                                │
   │  ┌──────────────────────────────────────────────────────────┐  │
   │  │  AUTO-CONFIGURATION                                      │  │
   │  │  "There's a DataSource on the classpath and no bean      │  │
   │  │   defined? I'll configure one from your properties."     │  │
   │  └────────────────────────┬─────────────────────────────────┘  │
   │  ┌────────────────────────▼─────────────────────────────────┐  │
   │  │  IoC CONTAINER (ApplicationContext)                      │  │
   │  │   • Creates objects (beans)                              │  │
   │  │   • Wires their dependencies                             │  │
   │  │   • Manages their lifecycle                              │  │
   │  │   • WRAPS THEM IN PROXIES for @Transactional, @Cacheable,│  │
   │  │     @Async, @PreAuthorize ...    ⭐ the key mechanism      │  │
   │  └────────────────────────┬─────────────────────────────────┘  │
   │  ┌────────────────────────▼─────────────────────────────────┐  │
   │  │  STARTERS  web · data-jpa · security · validation · ...  │  │
   │  └──────────────────────────────────────────────────────────┘  │
   └────────────────────────────────────────────────────────────────┘
```

🧠 **The one thing to internalize: Spring gives you a proxy, not your object.** When you annotate a method with `@Transactional`, Spring wraps your bean in a proxy that opens a transaction, calls your method, and commits. Almost every confusing Spring bug traces back to this — most famously, self-invocation bypassing the proxy entirely.

---

## 2. IoC Container

### 2.1 Dependency injection styles

```java
// ✅ CONSTRUCTOR INJECTION — the only one you should use
@Service
public class OrderService {
    private final OrderRepository repo;      // final = guaranteed initialized
    private final PaymentClient payment;

    // @Autowired is optional with a single constructor
    public OrderService(OrderRepository repo, PaymentClient payment) {
        this.repo = repo;
        this.payment = payment;
    }
}

// Or with Lombok
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repo;
    private final PaymentClient payment;
}

// ❌ FIELD INJECTION — avoid
@Autowired private OrderRepository repo;
```

**Why constructor injection wins:**

| Reason | Detail |
|---|---|
| Immutability | Fields can be `final` |
| Fail fast | Missing dependency = context startup failure, not a runtime NPE |
| Testable | `new OrderService(mockRepo, mockClient)` — no reflection, no Spring |
| Honest | A constructor with 9 parameters *tells you* the class does too much |
| No circular deps | Spring detects the cycle at startup instead of hiding it |

### 2.2 Stereotypes

```java
@Component      // generic managed bean
@Service        // business logic (semantic marker)
@Repository     // data access — ⭐ also translates JDBC/JPA exceptions
                //   into Spring's DataAccessException hierarchy
@Controller     // MVC controller (returns view names)
@RestController // @Controller + @ResponseBody
@Configuration  // declares @Bean methods
```

```java
@Configuration
public class AppConfig {
    @Bean
    public RestClient orderClient(RestClient.Builder builder) {
        return builder
            .baseUrl("https://orders.internal")
            .requestFactory(clientHttpRequestFactory())
            .build();
    }

    @Bean
    @ConditionalOnMissingBean          // only if nobody else defined one
    public Clock clock() { return Clock.systemUTC(); }
}
```

### 2.3 Resolving ambiguity

```java
// Multiple implementations of one interface
@Service @Primary
public class SmtpMailer implements Mailer {}

@Service @Qualifier("ses")
public class SesMailer implements Mailer {}

// Injection
public NotificationService(@Qualifier("ses") Mailer mailer) {}

// Inject ALL implementations
public Dispatcher(List<Mailer> mailers) {}          // ordered by @Order
public Dispatcher(Map<String, Mailer> byName) {}    // key = bean name
```

---

## 3. Bean Lifecycle

```
   ┌──────────────────────────────────────────────────────────┐
   │ 1. Instantiate (constructor)                             │
   │ 2. Populate properties (setter/field injection)          │
   │ 3. *Aware callbacks (BeanNameAware, ApplicationContextAware)│
   │ 4. BeanPostProcessor.postProcessBeforeInitialization      │
   │ 5. @PostConstruct                                        │
   │ 6. InitializingBean.afterPropertiesSet()                 │
   │ 7. Custom init-method                                    │
   │ 8. BeanPostProcessor.postProcessAfterInitialization      │
   │    ⭐ THIS IS WHERE PROXIES ARE CREATED (AOP)             │
   │ ─────────── bean is ready for use ───────────            │
   │ 9. @PreDestroy                                           │
   │10. DisposableBean.destroy()                              │
   │11. Custom destroy-method                                 │
   └──────────────────────────────────────────────────────────┘
```

### Scopes

| Scope | Instances | Use |
|---|---|---|
| `singleton` (default) | One per container | Stateless services ⭐ |
| `prototype` | New on every injection/lookup | Stateful helpers |
| `request` | One per HTTP request | Request-scoped context |
| `session` | One per HTTP session | User session state |
| `application` | One per ServletContext | |

⚠️ **Singletons must be stateless.** A mutable field on a `@Service` is shared by every concurrent request — a classic and dangerous race condition.

⚠️ **Injecting a prototype into a singleton gives you one instance forever**, because injection happens once at startup. Use `ObjectProvider<T>` or `@Lookup` when you genuinely need a fresh one per call.

---

## 4. Auto-Configuration

```
   @SpringBootApplication
      = @Configuration
      + @ComponentScan          (scan this package and below)
      + @EnableAutoConfiguration

   How auto-config works:
   1. Read META-INF/spring/…AutoConfiguration.imports from every jar
   2. For each candidate, evaluate its @Conditional annotations
   3. Apply the ones whose conditions pass

   Conditions:
     @ConditionalOnClass         — the class is on the classpath
     @ConditionalOnMissingBean   — ⭐ you haven't defined one yourself
     @ConditionalOnProperty      — a property has a given value
     @ConditionalOnWebApplication
```

`@ConditionalOnMissingBean` is the mechanism that makes Boot feel magical yet overridable: define your own `DataSource` bean and Boot silently backs off.

```bash
# See exactly what was and wasn't applied and why
java -jar app.jar --debug
# or
management.endpoints.web.exposure.include=conditions
GET /actuator/conditions
```

---

## 5. AOP and Proxies

### 5.1 How proxying works

```
   Caller ──▶ PROXY ──▶ your bean
                │
                ├─ before: start transaction / check cache / authorize
                ├─ invoke the real method
                └─ after: commit or rollback / store in cache

   TWO PROXY TYPES
   • JDK dynamic proxy — bean implements an interface; proxy implements it too
   • CGLIB subclass    — no interface; proxy is a generated SUBCLASS
                         ⚠️ so the method cannot be final or private,
                            and the class cannot be final
```

### 5.2 The self-invocation trap

**This is the single most common Spring bug.**

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order o) {
        repo.save(o);
        sendEmail(o);            // internal call
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendEmail(Order o) {     // ❌ NOT a new transaction!
        ...                              //    It's a plain method call on `this`,
    }                                    //    which bypasses the proxy entirely.
}
```

```
   Caller ──▶ PROXY ──▶ createOrder()  ← proxy intercepted ✅
                            │
                            └─ this.sendEmail()  ← direct call on the target
                                                   NO PROXY, NO NEW TRANSACTION ❌
```

**Fixes:**

```java
// 1. ⭐ Best: move it to another bean
@Service
class OrderService {
    private final EmailService email;      // separate bean → real proxy
    @Transactional public void createOrder(Order o) { repo.save(o); email.send(o); }
}

// 2. Self-injection
@Service
class OrderService {
    @Autowired @Lazy private OrderService self;
    @Transactional public void createOrder(Order o) { self.sendEmail(o); }
}

// 3. AopContext (requires exposeProxy = true)
((OrderService) AopContext.currentProxy()).sendEmail(o);
```

The same trap applies to `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable` — every proxy-based annotation.

### 5.3 Custom aspects

```java
@Aspect @Component @Slf4j
public class TimingAspect {

    @Around("@annotation(Timed) || within(@org.springframework.stereotype.Service *)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            if (ms > 500) {
                log.warn("Slow: {} took {}ms", pjp.getSignature().toShortString(), ms);
            }
        }
    }
}
```

---

## 6. Web Layer

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService service;

    @GetMapping("/{id}")
    public OrderResponse get(@PathVariable UUID id) {
        return service.findById(id).map(OrderResponse::from)
            .orElseThrow(() -> new NotFoundException("Order", id));
    }

    @GetMapping
    public PageResponse<OrderResponse> list(
            @RequestParam(required = false) OrderStatus status,
            @RequestParam(defaultValue = "20") @Max(100) int limit,
            @RequestParam(required = false) String cursor,
            @AuthenticationPrincipal UserPrincipal principal) {
        return service.list(principal.id(), status, limit, cursor);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<OrderResponse> create(
            @Valid @RequestBody CreateOrderRequest req,
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            UriComponentsBuilder uri) {
        Order order = service.create(req, idempotencyKey);
        return ResponseEntity
            .created(uri.path("/api/v1/orders/{id}").buildAndExpand(order.getId()).toUri())
            .body(OrderResponse.from(order));
    }

    @PatchMapping("/{id}")
    public OrderResponse patch(@PathVariable UUID id, @Valid @RequestBody UpdateOrderRequest r) {
        return OrderResponse.from(service.update(id, r));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable UUID id) { service.delete(id); }
}
```

### DTOs as records

```java
public record CreateOrderRequest(
    @NotNull UUID customerId,
    @NotEmpty @Size(max = 100) List<@Valid OrderLineRequest> lines,
    @Size(max = 500) String notes
) {}

public record OrderResponse(UUID id, UUID customerId, long totalCents,
                            OrderStatus status, Instant createdAt) {
    public static OrderResponse from(Order o) {
        return new OrderResponse(o.getId(), o.getCustomerId(),
                                 o.getTotalCents(), o.getStatus(), o.getCreatedAt());
    }
}
```

🏭 **Never expose JPA entities directly from controllers.** Doing so leaks your schema into your API contract, triggers lazy-loading exceptions during serialization, and risks exposing internal fields. Records make DTOs nearly free.

---

## 7. Data Access

### 7.1 Entities

```java
@Entity
@Table(name = "orders", indexes = @Index(columnList = "customer_id, created_at"))
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)   // JPA requires it; hide it
public class Order {

    @Id @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, updatable = false)
    private UUID customerId;

    @Enumerated(EnumType.STRING)     // ⭐ NEVER ORDINAL — reordering the enum
    @Column(nullable = false)        //    silently corrupts existing rows
    private OrderStatus status;

    @Column(nullable = false)
    private long totalCents;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();

    @Version                          // ⭐ optimistic locking
    private long version;

    @CreationTimestamp @Column(updatable = false)
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;

    // Bidirectional helper — keeps both sides consistent
    public void addLine(OrderLine line) { lines.add(line); line.setOrder(this); }
}
```

### 7.2 Repositories

```java
public interface OrderRepository extends JpaRepository<Order, UUID> {

    // Derived query — Spring generates the implementation from the method name
    List<Order> findByCustomerIdAndStatusOrderByCreatedAtDesc(UUID customerId, OrderStatus s);

    // ⭐ Fixes N+1 by fetching the association in one query
    @Query("SELECT o FROM Order o LEFT JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") UUID id);

    // Or declaratively
    @EntityGraph(attributePaths = {"lines", "customer"})
    Optional<Order> findWithGraphById(UUID id);

    // Projection — selects ONLY these columns
    interface OrderSummary { UUID getId(); long getTotalCents(); OrderStatus getStatus(); }
    List<OrderSummary> findSummaryByCustomerId(UUID customerId);

    // Pessimistic lock
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT o FROM Order o WHERE o.id = :id")
    Optional<Order> findByIdForUpdate(@Param("id") UUID id);

    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("UPDATE Order o SET o.status = :s WHERE o.id = :id AND o.status = :expected")
    int updateStatusIfCurrent(UUID id, OrderStatus s, OrderStatus expected);
}
```

### 7.3 The N+1 problem

```java
// ❌ 1 query for orders + N queries for lines
List<Order> orders = repo.findAll();
orders.forEach(o -> o.getLines().size());

// ✅ Fixes, in order of preference:
// 1. JOIN FETCH / @EntityGraph                    → 1 query
// 2. @BatchSize(size = 50) on the collection      → 1 + N/50 queries
// 3. Projection — don't load the association at all
```

```yaml
# ⭐ Make N+1 impossible to ship unnoticed
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100
logging.level.org.hibernate.SQL: DEBUG          # dev only
```

Better still: assert query counts in integration tests with Hibernate's `Statistics` or a library like `datasource-proxy`.

⚠️ **`FetchType.EAGER` is a trap.** It looks like a fix but loads the association on *every* query touching that entity, including ones that don't need it, and it can't be disabled per query. Always use `LAZY` and fetch explicitly.

---

## 8. Transactions

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // default
    isolation = Isolation.READ_COMMITTED,
    timeout = 10,
    readOnly = false,
    rollbackFor = Exception.class            // ⭐ see the gotcha below
)
public void transfer(UUID from, UUID to, long cents) { ... }
```

### 8.1 The default rollback rule

```java
// ⚠️ By default Spring rolls back on RuntimeException and Error ONLY.
//    A CHECKED exception COMMITS the transaction.

@Transactional
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("failed");    // ❌ entity IS COMMITTED
}

@Transactional(rollbackFor = Exception.class)   // ✅ rolls back on anything
```

This surprises nearly everyone the first time.

### 8.2 Propagation

| Value | Behavior |
|---|---|
| `REQUIRED` | Join the existing transaction, or start one (default) |
| `REQUIRES_NEW` | **Suspend** the current one, start an independent one |
| `SUPPORTS` | Join if one exists, otherwise run non-transactionally |
| `NOT_SUPPORTED` | Suspend any current transaction |
| `MANDATORY` | Throw if there's no existing transaction |
| `NEVER` | Throw if there *is* one |
| `NESTED` | Savepoint within the current transaction |

```java
// Classic use: audit logging that must survive a rollback of the main work
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String event) { auditRepo.save(new Audit(event)); }
```

⚠️ `REQUIRES_NEW` holds **two** connections simultaneously. With a pool of 10 and heavy use, you can deadlock the pool — every thread holding one connection while waiting for a second.

### 8.3 Optimistic vs pessimistic locking

```java
// OPTIMISTIC — @Version field; throws OptimisticLockException on conflict
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2))
@Transactional
public void updateOrder(UUID id, Consumer<Order> mutator) {
    Order o = repo.findById(id).orElseThrow();
    mutator.accept(o);
}

// PESSIMISTIC — a database row lock; blocks other writers
@Transactional
public void decrementStock(UUID productId, int qty) {
    Product p = repo.findByIdForUpdate(productId).orElseThrow();
    if (p.getStock() < qty) throw new InsufficientStockException();
    p.setStock(p.getStock() - qty);
}
```

### 8.4 Transaction gotchas

```
   ⚠️ Self-invocation → no proxy → NO TRANSACTION (see §5.2)
   ⚠️ private/final methods can't be proxied by CGLIB
   ⚠️ Checked exceptions commit by default
   ⚠️ @Transactional on a controller = a transaction spanning HTTP + view rendering
   ⚠️ Long transactions block DB vacuum and exhaust the pool
   ⚠️ Calling an external HTTP API inside a transaction holds a connection
      for the duration of that call ← very common production problem
   ⚠️ readOnly=true is a HINT — it sets the JDBC flag and skips dirty checking,
      but doesn't guarantee the database rejects writes
```

🏭 **Keep transactions short and free of network calls.** Load data, mutate, commit. Do the HTTP call before or after, never inside.

---

## 9. Validation and Errors

```java
public record CreateUserRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 12, max = 128) String password,
    @NotNull @Min(13) @Max(120) Integer age,
    @Pattern(regexp = "^[a-z0-9_]{3,32}$") String username,
    @Valid AddressRequest address                       // ⭐ cascade validation
) {}
```

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_ENTITY);
        pd.setTitle("Validation failed");
        pd.setType(URI.create("https://api.example.com/errors/validation"));
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .map(e -> Map.of("field", e.getField(),
                             "code", e.getCode(),
                             "message", e.getDefaultMessage()))
            .toList());
        return pd;
    }

    @ExceptionHandler(NotFoundException.class)
    public ProblemDetail onNotFound(NotFoundException ex) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Not Found");
        return pd;
    }

    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ProblemDetail onConflict(Exception ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT,
            "The resource was modified concurrently. Retry with fresh data.");
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail onUnhandled(Exception ex, HttpServletRequest req) {
        String id = MDC.get("requestId");
        log.error("Unhandled error requestId={}", id, ex);
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred.");     // ⭐ never leak the stack trace
        pd.setProperty("requestId", id);
        return pd;
    }
}
```

`ProblemDetail` (Spring 6) implements RFC 9457 natively — see [API Design §7](api-design.md#7-error-handling).

---

## 10. Security

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                        // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())    // stateless JWT API; KEEP for cookie sessions
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**", "/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .anyRequest().authenticated())          // ⭐ deny by default
            .oauth2ResourceServer(o -> o.jwt(jwt -> jwt
                .jwtAuthenticationConverter(converter())))
            .exceptionHandling(e -> e
                .authenticationEntryPoint(problemEntryPoint())   // 401
                .accessDeniedHandler(problemAccessDenied()))     // 403
            .headers(h -> h
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .contentSecurityPolicy(c -> c.policyDirectives("default-src 'self'")))
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);       // or Argon2PasswordEncoder
    }
}
```

```java
// Method-level authorization
@PreAuthorize("hasRole('ADMIN')")
@PreAuthorize("hasAuthority('order:write')")
@PreAuthorize("#userId == authentication.principal.id")        // ⭐ ownership check
@PostAuthorize("returnObject.ownerId == authentication.principal.id")
@PreAuthorize("@orderSecurity.canAccess(#orderId, authentication)")   // custom bean
```

⚠️ **The filter chain order matters and is easy to get wrong.** Rules are evaluated top to bottom; the first match wins. Putting `.anyRequest().authenticated()` early makes everything after it dead code.

⚠️ **Disable CSRF only for stateless token APIs.** If you use cookie-based sessions, CSRF protection is required.

---

## 11. Configuration

```yaml
# application.yml
spring:
  application.name: order-service
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20          # ⭐ (cores × 2) + spindles, NOT "as many as possible"
      minimum-idle: 5
      connection-timeout: 3000       # fail fast rather than pile up
      max-lifetime: 1800000
      leak-detection-threshold: 60000
  jpa:
    hibernate.ddl-auto: validate     # ⭐ NEVER `update` or `create` in production
    open-in-view: false              # ⭐⭐ ALWAYS false — see below
    properties:
      hibernate.jdbc.batch_size: 50
      hibernate.order_inserts: true
      hibernate.order_updates: true
  flyway:
    enabled: true
    locations: classpath:db/migration

management:
  endpoints.web.exposure.include: health,metrics,prometheus,info
  endpoint.health.show-details: when-authorized
  metrics.tags.application: ${spring.application.name}

server:
  shutdown: graceful
  tomcat.threads.max: 200
  compression.enabled: true

logging.level.root: INFO

---
spring.config.activate.on-profile: dev
spring.jpa.show-sql: true
logging.level.org.hibernate.SQL: DEBUG
```

⚠️ **`spring.jpa.open-in-view` defaults to `true` and should always be `false`.** When true, the Hibernate session stays open for the entire HTTP request including view rendering, so a lazy association accessed during JSON serialization silently issues a query — from the presentation layer, holding a database connection for the whole request. It hides N+1 problems and exhausts the connection pool under load.

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String apiKey,
    @NotNull Duration timeout,
    @Min(1) @Max(10) int maxRetries
) {}
```

Validated at startup — a bad config fails the boot, not a 3 a.m. request.

---

## 12. Testing

```java
// ── Unit: no Spring at all, fastest ──────────────────────
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock PaymentClient payment;
    @InjectMocks OrderService service;

    @Test
    void createsOrderAndCharges() {
        when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));
        service.create(request);
        verify(payment).charge(eq(4500L), any());
    }
}

// ── Slice: only the web layer ────────────────────────────
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;              // Spring Boot 3.4+

    @Test
    void returns404WhenMissing() throws Exception {
        when(service.findById(any())).thenReturn(Optional.empty());
        mvc.perform(get("/api/v1/orders/{id}", UUID.randomUUID()))
           .andExpect(status().isNotFound())
           .andExpect(jsonPath("$.title").value("Not Found"));
    }
}

// ── Slice: only JPA, with a real database via Testcontainers ──
@DataJpaTest
@AutoConfigureTestDatabase(replace = NONE)
@Testcontainers
class OrderRepositoryTest {
    @Container @ServiceConnection                   // ⭐ auto-wires the datasource
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void fetchesLinesInOneQuery() { ... }
}

// ── Full integration ─────────────────────────────────────
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderIntegrationTest {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:16");

    @Autowired TestRestTemplate rest;

    @Test
    void createOrderEndToEnd() {
        var res = rest.postForEntity("/api/v1/orders", request, OrderResponse.class);
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

```
   TEST PYRAMID
             ▲  few, slow, high confidence
        ┌────┴────┐
        │   E2E   │   @SpringBootTest + Testcontainers
       ┌┴─────────┴┐
       │  SLICES   │  @WebMvcTest, @DataJpaTest — fast, focused
      ┌┴───────────┴┐
      │    UNIT     │  plain JUnit + Mockito, milliseconds
      └─────────────┘  many, fast
```

🏭 **Testcontainers over H2.** An in-memory database with different SQL dialect, different constraint behavior, and no real locking gives you passing tests and broken production. A real Postgres in Docker costs seconds and tests the truth.

---

## 13. Reactive

```java
@RestController
public class ReactiveController {

    @GetMapping("/orders/{id}")
    public Mono<OrderResponse> get(@PathVariable UUID id) {
        return repo.findById(id).map(OrderResponse::from)
                   .switchIfEmpty(Mono.error(new NotFoundException()));
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Event> stream() {
        return eventService.stream().delayElements(Duration.ofMillis(100));
    }

    // Combine independent calls
    public Mono<Dashboard> dashboard(UUID userId) {
        return Mono.zip(
            userClient.get(userId),
            orderClient.recent(userId),
            recsClient.forUser(userId)
        ).map(t -> new Dashboard(t.getT1(), t.getT2(), t.getT3()));
    }
}
```

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Model | Thread per request | Event loop, non-blocking |
| Threads for 10k concurrent | 10k (or a queue) | A handful |
| Blocking code | Fine | **Catastrophic** |
| Debugging | Normal stack traces | Painful |
| Ecosystem | Complete | Fewer non-blocking drivers |
| Learning curve | Low | High |

🏭 **With Java 21 virtual threads, WebFlux's main advantage largely evaporates.** Setting `spring.threads.virtual.enabled=true` gives blocking-style MVC code the scalability of an event loop, without the reactive programming model. For new services, that's usually the better trade — reserve WebFlux for genuine streaming and backpressure requirements.

---

## 14. Observability

```java
@Observed(name = "order.create")           // creates a metric AND a trace span
public Order create(CreateOrderRequest r) { ... }

// Custom metrics
private final Counter created = Counter.builder("orders.created")
    .tag("service", "order").register(meterRegistry);

// Structured logging with correlation
MDC.put("requestId", requestId);
MDC.put("userId", userId);
log.info("Order created orderId={} totalCents={}", order.getId(), order.getTotalCents());
```

```yaml
management:
  tracing.sampling.probability: 0.1        # 10% in production
  otlp.tracing.endpoint: http://collector:4318/v1/traces
  endpoints.web.exposure.include: health,metrics,prometheus
  endpoint.health:
    probes.enabled: true                    # /health/liveness and /health/readiness
    group.readiness.include: db,redis
```

---

## 15. Production

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew build.gradle settings.gradle ./
COPY gradle ./gradle
RUN ./gradlew dependencies --no-daemon
COPY src ./src
RUN ./gradlew bootJar --no-daemon
RUN java -Djarmode=layertools -jar build/libs/*.jar extract

FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
# Layered copy — dependencies change rarely, so Docker caches them
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
USER app
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC \
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp"
ENTRYPOINT ["sh","-c","java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

```
   ⭐ In containers use -XX:MaxRAMPercentage, NOT -Xmx.
      The JVM then respects the cgroup limit automatically,
      so the same image works with any memory allocation.
```

**Startup time:** Spring Boot's cold start hurts in serverless contexts. Options: `spring.main.lazy-initialization=true` (fast start, deferred cost), Class Data Sharing (`-XX:SharedArchiveFile`), or GraalVM native images for sub-100ms startup at the cost of build complexity and reflection configuration.

---

## 16. Interview Section

<details>
<summary><b>Q1. What is dependency injection and why constructor injection?</b></summary>

Dependency injection means an object receives its collaborators rather than constructing them. The container owns object creation and wiring, which is the "inversion of control."

Constructor injection is the right form for several concrete reasons. Fields can be `final`, so the object is immutable and safely publishable across threads. A missing dependency fails at context startup rather than as a null pointer at runtime. The class is testable with plain `new` — no reflection, no Spring context. And it's honest: a constructor with nine parameters visibly tells you the class does too much, whereas nine `@Autowired` fields hide it.

Field injection also makes circular dependencies silently work, which sounds convenient but means a design problem goes undetected. Constructor injection surfaces the cycle at startup.
</details>

<details>
<summary><b>Q2. Why does `@Transactional` sometimes not work?</b></summary>

Almost always self-invocation. Spring implements `@Transactional` with a proxy that wraps your bean. When an external caller invokes a method, it goes through the proxy, which starts the transaction. But when a method inside the bean calls another method on `this`, that's a direct call on the target object — the proxy is never involved, so the annotation does nothing.

The fix is to move the annotated method to a different bean, so the call goes through a real proxy. Self-injection with `@Lazy` also works, and `AopContext.currentProxy()` if the proxy is exposed, but extracting a collaborator is cleaner and usually reveals a better design.

Secondary causes: the method is `private` or `final`, so CGLIB can't override it; the exception thrown was checked, and Spring's default only rolls back on `RuntimeException` and `Error`; or the class isn't a Spring bean at all.

The same trap applies to every proxy-based annotation — `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`.
</details>

<details>
<summary><b>Q3. Explain the N+1 problem in JPA and how to fix it.</b></summary>

You load N entities with one query, then access a lazy association on each, producing N additional queries. Loading 100 orders and touching their lines becomes 101 queries.

Fixes in order of preference. `JOIN FETCH` or `@EntityGraph` loads the association in the same query — one query total, and it's explicit at the call site so different use cases can fetch differently. `@BatchSize` on the collection turns N queries into N divided by the batch size, which is a good default safety net. And projections avoid loading the association at all when you only need a few columns.

What I'd avoid is `FetchType.EAGER`. It looks like a fix but loads the association on every query touching that entity, including ones that don't need it, and you can't turn it off per query.

The important part is detection, since N+1 is invisible with test data and fatal with production data. I'd set `open-in-view: false` so lazy loading outside a transaction fails loudly, log queries slower than a threshold, and assert query counts in integration tests.
</details>

<details>
<summary><b>Q4. What is `open-in-view` and why turn it off?</b></summary>

It keeps the Hibernate session open for the entire HTTP request, including view rendering and JSON serialization. It defaults to true, which is one of Spring Boot's more questionable defaults.

Two problems. It hides N+1: a lazy association accessed during serialization silently issues a query from the presentation layer, so the code looks clean while generating hundreds of queries. And it holds a database connection for the whole request duration rather than just the transactional portion, so under load the connection pool exhausts and everything queues.

Turning it off makes lazy access outside a transaction throw `LazyInitializationException`. That's uncomfortable initially but correct — it forces you to decide explicitly what data each endpoint needs and fetch it inside the service layer, which is where that decision belongs.
</details>

<details>
<summary><b>Q5. How does Spring Boot auto-configuration work?</b></summary>

`@EnableAutoConfiguration` reads a list of configuration classes from `META-INF/spring/…AutoConfiguration.imports` in every jar on the classpath. Each is guarded by conditions, and only those whose conditions pass get applied.

The conditions are things like `@ConditionalOnClass` — is this library present? — and `@ConditionalOnProperty`. The one that makes it feel magical yet controllable is `@ConditionalOnMissingBean`: Boot only configures a `DataSource` if you haven't defined one. So overriding anything is just defining your own bean; Boot silently backs off.

When something unexpected happens, `--debug` or the `/actuator/conditions` endpoint prints exactly which auto-configurations were applied, which were skipped, and the specific condition that decided each. That report answers most "why is Spring doing this" questions in seconds.
</details>

<details>
<summary><b>Q6. Bean scopes — and what goes wrong?</b></summary>

Singleton is the default: one instance per container, shared by all requests. Prototype creates a new instance per injection or lookup. There are also web scopes — request, session, application.

Two things go wrong. First, singletons with mutable state: a field on a `@Service` is shared by every concurrent request, so it's a race condition. Services must be stateless, with all per-request data passed as parameters.

Second, injecting a prototype into a singleton. Injection happens once at startup, so the singleton holds one prototype instance forever — the scope silently does nothing. If you genuinely need a fresh instance per call, you need `ObjectProvider`, `@Lookup`, or a scoped proxy.
</details>

<details>
<summary><b>Q7. How do you handle concurrent updates to the same entity?</b></summary>

Optimistic locking by default: a `@Version` field that Hibernate includes in the WHERE clause and increments on update. If another transaction modified the row, zero rows match and you get `OptimisticLockException`, which you catch and retry with fresh data. It costs nothing when there's no contention, which is the common case.

Pessimistic locking when contention is genuinely high and retries would thrash — `SELECT FOR UPDATE` via `@Lock(PESSIMISTIC_WRITE)`. Inventory decrements are the classic case: many concurrent buyers for the same item, where optimistic retry loops would waste more than the lock costs.

The tradeoff is that pessimistic locks serialize access and risk deadlocks, so I'd keep the locked section as short as possible and always acquire multiple locks in a consistent order.

There's a third option worth mentioning: a conditional update statement — `UPDATE ... WHERE status = :expected` — which pushes the check into the database atomically and avoids loading the entity at all.
</details>

<details>
<summary><b>Q8. Spring MVC vs WebFlux — and where do virtual threads fit?</b></summary>

MVC is thread-per-request: each request occupies a platform thread, which blocks on I/O. Simple to write and debug, but 10,000 concurrent requests means 10,000 threads, which is roughly 10 GB of stacks and heavy context switching.

WebFlux is an event loop with a handful of threads, so it scales to high concurrency with little memory. The costs are real: a single blocking call poisons the loop, stack traces become nearly useless, and you need non-blocking drivers for everything including your database.

Virtual threads change the calculus significantly. With `spring.threads.virtual.enabled=true` on Java 21, MVC code written in ordinary blocking style gets scheduled onto a small pool of carrier threads and unmounts on blocking I/O. You get most of WebFlux's scalability with none of its programming model.

So for new services I'd default to MVC with virtual threads, and reserve WebFlux for genuine streaming with backpressure requirements — where the reactive model earns its complexity rather than just being used for concurrency.
</details>

<details>
<summary><b>Q9. How do you test a Spring Boot application?</b></summary>

Three layers. Unit tests with plain JUnit and Mockito, no Spring context at all — that's where business logic lives and it should run in milliseconds. Slice tests like `@WebMvcTest` and `@DataJpaTest`, which start only the relevant part of the context so they stay fast. And a small number of full `@SpringBootTest` integration tests for critical paths.

The most important choice is Testcontainers over H2 for anything touching the database. H2 has a different SQL dialect, different constraint behavior, and no real locking — so it produces passing tests and broken production. A real Postgres in Docker costs a few seconds of startup and tests the actual truth. `@ServiceConnection` wires the datasource automatically now, so the setup is a couple of lines.

I'd also check things people forget: that error responses have the right shape, that authorization actually blocks cross-tenant access, and that query counts haven't regressed — N+1 problems don't fail tests unless you assert on them.
</details>

<details>
<summary><b>Q10. How do you size the connection pool and what goes wrong?</b></summary>

The counterintuitive answer is that smaller is usually better. HikariCP's own guidance is roughly cores times two plus effective spindles — so often 10 to 20, not 100. Beyond that the database becomes context-switch bound and total throughput *drops*.

What goes wrong: pool exhaustion, where every connection is held and requests queue until they time out. The usual causes are transactions held open across a network call to an external API, `open-in-view` holding a connection for the entire request including rendering, `REQUIRES_NEW` propagation where one thread holds two connections simultaneously, and connection leaks from code that doesn't close properly.

The settings I'd always configure: a short `connection-timeout` so requests fail fast rather than piling up, `leak-detection-threshold` to log stack traces of connections held too long, and `max-lifetime` below any proxy or database idle timeout so you don't hand out dead connections.

And the multiplication people miss: pool size is per instance. Twenty replicas with a pool of twenty is four hundred connections, which may exceed the database's limit entirely.
</details>

---

## 17. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                       SPRING BOOT — ONE PAGE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ SPRING GIVES YOU A PROXY, NOT YOUR OBJECT                          ║
║   self-invocation (this.method()) BYPASSES the proxy                 ║
║   → @Transactional/@Cacheable/@Async silently do NOTHING             ║
║   fix: move the method to another bean                               ║
╠══════════════════════════════════════════════════════════════════════╣
║ DI: constructor injection only (final fields, fail fast, testable)   ║
║ SCOPES: singleton default → MUST be stateless                        ║
║   prototype into singleton = injected ONCE (use ObjectProvider)      ║
╠══════════════════════════════════════════════════════════════════════╣
║ @Transactional                                                       ║
║   rolls back on RuntimeException/Error ONLY                          ║
║   → checked exception COMMITS unless rollbackFor=Exception.class     ║
║   REQUIRES_NEW holds TWO connections → pool deadlock risk            ║
║   NEVER make network calls inside a transaction                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ JPA                                                                  ║
║   spring.jpa.open-in-view: FALSE  ⭐⭐ (default true is wrong)         ║
║   ddl-auto: validate (never update/create in prod)                   ║
║   @Enumerated(STRING) never ORDINAL                                  ║
║   N+1 → JOIN FETCH / @EntityGraph / @BatchSize; never EAGER          ║
║   @Version for optimistic lock + @Retryable                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ POOL: size ≈ (cores × 2) + spindles — SMALLER is usually faster      ║
║   per instance! 20 replicas × 20 = 400 DB connections                ║
║   set connection-timeout + leak-detection-threshold                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ WEB: never expose entities — use record DTOs                         ║
║   @RestControllerAdvice + ProblemDetail (RFC 9457)                   ║
║   never leak stack traces; include a requestId                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ SECURITY: anyRequest().authenticated() LAST (first match wins)       ║
║   @PreAuthorize("#userId == authentication.principal.id") for BOLA   ║
║   keep CSRF for cookie sessions; disable only for stateless tokens   ║
╠══════════════════════════════════════════════════════════════════════╣
║ TEST: Testcontainers NOT H2 · unit > slice > integration             ║
║ PROD: -XX:MaxRAMPercentage (not -Xmx) · graceful shutdown ·          ║
║   Java 21 virtual threads instead of WebFlux for most services       ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Java](../01-languages/java.md) · [API Design](api-design.md) · [Databases](databases.md) · [Observability](../06-cloud-devops/observability-sre.md)
