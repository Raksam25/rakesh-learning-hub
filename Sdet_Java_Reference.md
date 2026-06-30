# SDET Java Reference Guide
> Covers Core Java, Design Patterns, Concurrency, Java 8+, and Spring Boot — filtered for SDET / Senior SDET / Lead SDET roles.

---

## Table of Contents
1. [OOP for Test Automation](#1-oop-for-test-automation)
2. [Static, ThreadLocal & Parallel Test Safety](#2-static-threadlocal--parallel-test-safety)
3. [Exception Handling in Test Code](#3-exception-handling-in-test-code)
4. [Collections for Test Data Management](#4-collections-for-test-data-management)
5. [Generics & Type-Safe Test Utilities](#5-generics--type-safe-test-utilities)
6. [Enums for Test Configuration](#6-enums-for-test-configuration)
7. [String Handling & Assertions](#7-string-handling--assertions)
8. [Concurrency for Parallel Test Execution](#8-concurrency-for-parallel-test-execution)
9. [Java 8+ for Clean Test Code](#9-java-8-for-clean-test-code)
10. [Design Patterns for Test Automation](#10-design-patterns-for-test-automation)
11. [Spring Boot for API & Integration Testing](#11-spring-boot-for-api--integration-testing)
12. [Test Architect — Additional Topics to Cover](#12-test-architect--additional-topics-to-cover)

---

## 1. OOP for Test Automation

### 1.1 Encapsulation — Page Object Model Foundation
Encapsulation is the core principle behind POM. WebElement locators are private; actions are exposed as public methods.

```java
public class LoginPage {
    private WebDriver driver;

    // Private — internal implementation detail
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton   = By.id("loginBtn");

    public LoginPage(WebDriver driver) { this.driver = driver; }

    // Public — contract exposed to test
    public HomePage login(String user, String pass) {
        driver.findElement(usernameField).sendKeys(user);
        driver.findElement(passwordField).sendKeys(pass);
        driver.findElement(loginButton).click();
        return new HomePage(driver);
    }
}
```

**Why it matters in SDET:** If the locator changes, you update ONE place. Tests never break due to locator changes — they break only when actual behavior changes.

---

### 1.2 Abstraction — BasePage & BaseTest
Abstraction hides the "how" and exposes the "what". `BasePage` hides waits and driver calls; test classes only see business actions.

```java
// Abstract BasePage — hides WebDriver complexity
public abstract class BasePage {
    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait   = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    protected void click(By locator) {
        wait.until(ExpectedConditions.elementToBeClickable(locator)).click();
    }

    protected void type(By locator, String text) {
        WebElement el = wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
        el.clear();
        el.sendKeys(text);
    }

    protected String getText(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator)).getText();
    }
}

// LoginPage only describes WHAT the page does
public class LoginPage extends BasePage {
    private By username = By.id("username");
    private By password = By.id("password");
    private By submit   = By.cssSelector("button[type='submit']");

    public LoginPage(WebDriver driver) { super(driver); }

    public DashboardPage loginAs(String user, String pass) {
        type(username, user);
        type(password, pass);
        click(submit);
        return new DashboardPage(driver);
    }
}
```

---

### 1.3 Inheritance — BaseTest
```java
public class BaseTest {
    protected WebDriver driver;

    @BeforeMethod
    public void setUp() {
        driver = BrowserFactory.createDriver(BrowserType.CHROME);
        DriverManager.setDriver(driver);
    }

    @AfterMethod
    public void tearDown() {
        DriverManager.unload();
    }
}

public class LoginTest extends BaseTest {
    @Test
    public void validLoginTest() {
        new LoginPage(DriverManager.getDriver())
            .loginAs("admin", "secret123")
            .verifyWelcomeMessage("Welcome, Admin");
    }
}
```

---

### 1.4 Polymorphism — Strategy for Multiple Environments/Browsers
Runtime polymorphism lets tests run against different implementations (browsers, environments, auth types) without changing test code.

```java
interface Browser {
    WebDriver create();
}

class ChromeBrowser implements Browser {
    public WebDriver create() {
        ChromeOptions opts = new ChromeOptions();
        opts.addArguments("--headless");
        return new ChromeDriver(opts);
    }
}

class FirefoxBrowser implements Browser {
    public WebDriver create() { return new FirefoxDriver(); }
}

// Polymorphic usage — test doesn't know which browser
Browser browser = BrowserType.valueOf(System.getProperty("browser", "CHROME")).create();
WebDriver driver = browser.create();
```

---

### 1.5 IS-A vs HAS-A in Test Frameworks

| Relationship | Usage in Testing |
|---|---|
| IS-A (extends) | `LoginTest extends BaseTest` — inherits setup/teardown |
| HAS-A (composition) | `LoginPage` HAS-A `WebDriver` — page uses driver, doesn't extend it |

**Rule:** Use `extends` for test classes sharing lifecycle. Use composition for Page Objects using WebDriver — never extend WebDriver.

---

### 1.6 Interfaces for Loose Coupling

```java
// Test depends on interface, not concrete class
public interface UserRepository {
    User findById(long id);
    User save(User user);
}

// Real implementation uses DB
public class JpaUserRepository implements UserRepository { ... }

// Test implementation uses in-memory map
public class InMemoryUserRepository implements UserRepository {
    private Map<Long, User> store = new HashMap<>();
    public User findById(long id) { return store.get(id); }
    public User save(User user)   { store.put(user.getId(), user); return user; }
}
```

**SDET benefit:** Tests remain fast and isolated; no DB required for unit-level tests.

---

## 2. Static, ThreadLocal & Parallel Test Safety

### 2.1 Static Members in Test Frameworks
Static members belong to the class — shared across all objects. In parallel tests, sharing a static `WebDriver` causes race conditions.

```java
// WRONG — static driver shared across threads
public class BaseTest {
    private static WebDriver driver;  // shared → tests interfere!
}

// CORRECT — ThreadLocal isolates driver per thread
public class DriverManager {
    private static final ThreadLocal<WebDriver> driverHolder = new ThreadLocal<>();

    public static void setDriver(WebDriver driver) { driverHolder.set(driver); }
    public static WebDriver getDriver()            { return driverHolder.get(); }
    public static void unload() {
        if (driverHolder.get() != null) {
            driverHolder.get().quit();
            driverHolder.remove();         // prevent memory leak
        }
    }
}
```

**Key rules:**
- `ThreadLocal.remove()` must always be called in `@AfterMethod` — failure causes memory leaks, especially in thread pool environments.
- Static constants (`static final`) are safe. Static mutable state (`static int count`) is not safe without synchronization.

---

### 2.2 ThreadLocal for Parallel Execution
```java
// Full parallel-safe base test
public class BaseTest {

    @BeforeMethod
    @Parameters("browser")
    public void setUp(@Optional("CHROME") String browser) {
        WebDriver driver = BrowserType.valueOf(browser.toUpperCase()).create();
        DriverManager.setDriver(driver);
        DriverManager.getDriver().get(ConfigLoader.get("base.url"));
    }

    @AfterMethod(alwaysRun = true)   // alwaysRun ensures cleanup even on failure
    public void tearDown() {
        DriverManager.unload();
    }
}

// testng.xml for parallel execution
// <suite name="Suite" parallel="methods" thread-count="5">
```

---

## 3. Exception Handling in Test Code

### 3.1 Checked vs Unchecked in Testing Context
| Exception Type | When to throw in tests | Example |
|---|---|---|
| Unchecked (RuntimeException) | Most test utilities and helpers | `ElementNotFoundException` |
| Checked | I/O, config loading, DB operations | `IOException` when reading test data file |

```java
// Custom test exceptions
public class TestConfigException extends RuntimeException {
    public TestConfigException(String property) {
        super("Missing required test property: " + property);
    }
}

public class ElementNotFoundException extends RuntimeException {
    public ElementNotFoundException(String locator) {
        super("Element not found after timeout: " + locator);
    }
}
```

---

### 3.2 Exception Handling for Test Stability
```java
// Retry-safe action with exception chaining
public void clickWithRetry(By locator, int maxAttempts) {
    for (int i = 1; i <= maxAttempts; i++) {
        try {
            driver.findElement(locator).click();
            return;
        } catch (StaleElementReferenceException | ElementClickInterceptedException e) {
            if (i == maxAttempts) {
                throw new RuntimeException("Click failed after " + maxAttempts + " attempts on " + locator, e);
            }
            sleep(500);
        }
    }
}

// Config loader with meaningful errors
public class ConfigLoader {
    private static Properties props = new Properties();

    static {
        try (InputStream is = ConfigLoader.class.getResourceAsStream("/config.properties")) {
            if (is == null) throw new TestConfigException("config.properties not found in classpath");
            props.load(is);
        } catch (IOException e) {
            throw new TestConfigException("Failed to load config: " + e.getMessage());
        }
    }

    public static String get(String key) {
        String value = props.getProperty(key);
        if (value == null) throw new TestConfigException(key);
        return value;
    }
}
```

---

### 3.3 try-with-resources for Test Data Files
```java
// Read test data from CSV/Excel safely
public List<String[]> readTestData(String fileName) {
    List<String[]> data = new ArrayList<>();
    try (BufferedReader br = new BufferedReader(
            new FileReader("src/test/resources/testdata/" + fileName))) {
        String line;
        br.readLine(); // skip header
        while ((line = br.readLine()) != null) {
            data.add(line.split(","));
        }
    } catch (IOException e) {
        throw new TestConfigException("Cannot read test data: " + fileName);
    }
    return data;
}
```

---

### 3.4 finally in Test Teardown
```java
@Test
public void criticalFlowTest() {
    WebDriver driver = BrowserFactory.createDriver();
    try {
        new LoginPage(driver).loginAs("admin", "pass");
        // test steps
    } catch (Exception e) {
        captureScreenshot(driver, "criticalFlowTest_FAILED");
        throw e;   // re-throw so TestNG marks test as FAILED
    } finally {
        driver.quit();  // always quit, even on failure
    }
}
```

---

## 4. Collections for Test Data Management

### 4.1 List — Test Step Sequences & Multiple Values
```java
// Collect all validation errors
List<String> validationErrors = new ArrayList<>();
if (actualTitle.isEmpty())         validationErrors.add("Title is empty");
if (actualPrice.equals("0.00"))    validationErrors.add("Price is zero");
if (actualStatus.equals("DRAFT"))  validationErrors.add("Status is DRAFT");

// Soft assert — fail with all errors at once
Assert.assertTrue(validationErrors.isEmpty(),
    "Validation failed:\n" + String.join("\n", validationErrors));

// Data-driven test data
List<String[]> loginData = Arrays.asList(
    new String[]{"admin@test.com",    "Admin@123",  "Dashboard"},
    new String[]{"user@test.com",     "User@123",   "Home"},
    new String[]{"invalid@test.com",  "wrong",      null}
);
```

---

### 4.2 Map — Test Configuration & Response Validation
```java
// Environment-specific config
Map<String, String> envConfig = new HashMap<>();
envConfig.put("QA",   "https://qa.example.com");
envConfig.put("UAT",  "https://uat.example.com");
envConfig.put("PROD", "https://example.com");

String baseUrl = envConfig.getOrDefault(System.getenv("ENV"), envConfig.get("QA"));

// API response validation
Map<String, Object> responseBody = objectMapper.readValue(response.getBody(), Map.class);
assertEquals(responseBody.get("status"), "SUCCESS");
assertEquals(responseBody.get("userId"), expectedUserId);

// Test data parameterization
Map<String, String> formData = new LinkedHashMap<>(); // insertion order matters for forms
formData.put("firstName", "John");
formData.put("lastName",  "Doe");
formData.put("email",     "john@test.com");
formData.forEach((field, value) -> page.fillField(field, value));
```

---

### 4.3 Set — Unique Validation
```java
// Check no duplicate product names on a listing page
List<String> productNames = page.getAllProductNames();
Set<String> unique = new HashSet<>(productNames);
assertEquals(unique.size(), productNames.size(), "Duplicate products found on page");

// Verify exact items present (order-independent)
Set<String> expectedMenuItems = new HashSet<>(Arrays.asList("Home", "Products", "Cart", "Profile"));
Set<String> actualMenuItems   = new HashSet<>(page.getMenuItemTexts());
assertEquals(actualMenuItems, expectedMenuItems);
```

---

### 4.4 Collections Complexity Reference

| Collection | get | add | remove | Notes for Testing |
|---|---|---|---|---|
| ArrayList | O(1) | O(1) amortized | O(n) | Default choice for test data lists |
| LinkedList | O(n) | O(1) ends | O(1) iterator | Use when heavy insert/delete in middle |
| HashSet | O(1) | O(1) | O(1) | Deduplication, membership checks |
| TreeSet | O(log n) | O(log n) | O(log n) | Sorted output verification |
| HashMap | O(1) | O(1) | O(1) | Key-value test data, config maps |
| LinkedHashMap | O(1) | O(1) | O(1) | Preserve form field order |

---

### 4.5 Fail-fast vs Fail-safe — Safe Iteration in Tests
```java
// WRONG — ConcurrentModificationException if test modifies during iteration
for (String item : testSteps) {
    if (shouldSkip(item)) testSteps.remove(item);  // throws!
}

// CORRECT
Iterator<String> it = testSteps.iterator();
while (it.hasNext()) {
    if (shouldSkip(it.next())) it.remove();  // safe removal
}

// Java 8+ preferred
testSteps.removeIf(this::shouldSkip);

// For parallel test listeners (fail-safe)
List<String> failedTests = new CopyOnWriteArrayList<>(); // thread-safe, fail-safe
```

---

## 5. Generics & Type-Safe Test Utilities

```java
// Generic page factory — no casting needed
public class PageFactory {
    public static <T extends BasePage> T create(Class<T> pageClass, WebDriver driver) {
        try {
            return pageClass.getDeclaredConstructor(WebDriver.class).newInstance(driver);
        } catch (Exception e) {
            throw new RuntimeException("Cannot create page: " + pageClass.getSimpleName(), e);
        }
    }
}
// Usage
LoginPage login = PageFactory.create(LoginPage.class, driver);

// Generic test data builder
public class TestDataBuilder<T> {
    private T entity;
    public TestDataBuilder(Supplier<T> factory) { this.entity = factory.get(); }
    public TestDataBuilder<T> with(Consumer<T> setter) { setter.accept(entity); return this; }
    public T build() { return entity; }
}
// Usage
User user = new TestDataBuilder<>(User::new)
    .with(u -> u.setName("John"))
    .with(u -> u.setEmail("john@test.com"))
    .build();

// Generic API response wrapper
public class ApiResponse<T> {
    private int statusCode;
    private T body;
    private String errorMessage;

    public void assertSuccess() {
        assertTrue(statusCode >= 200 && statusCode < 300,
            "Expected success but got: " + statusCode + " - " + errorMessage);
    }
    public T getBody() { return body; }
}
ApiResponse<OrderDto> response = apiClient.post("/orders", payload, OrderDto.class);
response.assertSuccess();
```

---

## 6. Enums for Test Configuration

Enums replace magic strings and provide compile-time safety across the framework.

```java
// Browser configuration
public enum BrowserType {
    CHROME {
        public WebDriver create() {
            ChromeOptions opts = new ChromeOptions();
            opts.addArguments("--headless", "--no-sandbox");
            return new ChromeDriver(opts);
        }
    },
    FIREFOX {
        public WebDriver create() { return new FirefoxDriver(); }
    },
    EDGE {
        public WebDriver create() { return new EdgeDriver(); }
    };

    public abstract WebDriver create();
}

// Environment configuration
public enum Environment {
    QA  ("https://qa.example.com",   "qa_db",   "qa_user"),
    UAT ("https://uat.example.com",  "uat_db",  "uat_user"),
    PROD("https://example.com",      "prod_db", "prod_user");

    public final String baseUrl, dbName, dbUser;
    Environment(String baseUrl, String dbName, String dbUser) {
        this.baseUrl = baseUrl; this.dbName = dbName; this.dbUser = dbUser;
    }
}

// Test priority / severity
public enum TestPriority { P0_SMOKE, P1_CRITICAL, P2_REGRESSION, P3_EXPLORATORY }

// CI/CD driven selection
BrowserType browser = BrowserType.valueOf(
    System.getProperty("browser", "CHROME").toUpperCase());
Environment env = Environment.valueOf(
    System.getProperty("env", "QA").toUpperCase());

WebDriver driver   = browser.create();
String    baseUrl  = env.baseUrl;
```

---

## 7. String Handling & Assertions

### 7.1 Common String Operations in Test Code
```java
// == vs equals() — critical bug in test assertions
String expected = "Login Successful";
String actual   = page.getSuccessMessage();

// WRONG — may pass or fail randomly (reference comparison)
if (expected == actual) { ... }

// CORRECT — always use equals() for string comparison
assertTrue(actual.equals(expected));
assertEquals(actual, expected);   // TestNG / JUnit assertion

// Case-insensitive comparison
assertEquals(actual.toLowerCase(), expected.toLowerCase());
assertTrue(actual.equalsIgnoreCase(expected));

// Contains check (partial match)
assertTrue(actual.contains("Successful"));
assertTrue(actual.startsWith("Login"));

// Trim before asserting (hidden spaces cause flakiness)
assertEquals(actual.trim(), expected.trim());
```

---

### 7.2 StringBuilder for Dynamic Test Data
```java
// Build dynamic test report message
StringBuilder report = new StringBuilder();
report.append("Test: ").append(testName).append("\n");
report.append("Status: ").append(status).append("\n");
report.append("Duration: ").append(durationMs).append("ms\n");

// Build dynamic SQL for test data setup
StringBuilder query = new StringBuilder("SELECT * FROM orders WHERE 1=1");
if (userId   != null) query.append(" AND user_id = ").append(userId);
if (status   != null) query.append(" AND status = '").append(status).append("'");
if (fromDate != null) query.append(" AND created_at >= '").append(fromDate).append("'");
```

---

### 7.3 String.format / formatted() for Test Messages
```java
// Descriptive failure messages using formatted strings
String errorMsg = String.format(
    "Expected price [%s] but found [%s] for product [%s] on env [%s]",
    expectedPrice, actualPrice, productName, Environment.QA.baseUrl);
assertEquals(actualPrice, expectedPrice, errorMsg);

// Log entry generation
String logEntry = "%-20s | %-10s | %-8s | %s".formatted(
    testName, status, duration + "ms", timestamp);
```

---

## 8. Concurrency for Parallel Test Execution

### 8.1 Thread Lifecycle — Why It Matters for Tests
| State | Test Scenario |
|---|---|
| NEW | Thread created but `start()` not called |
| RUNNABLE | Test running in thread pool (TestNG parallel execution) |
| BLOCKED | Waiting for synchronized lock (e.g., shared DriverManager without ThreadLocal) |
| WAITING | `latch.await()` waiting for all setup tasks to finish |
| TIMED_WAITING | `Thread.sleep()`, `wait(timeout)` |
| TERMINATED | Test method finished (pass or fail) |

---

### 8.2 ThreadLocal — Core Pattern for Parallel Safety
```java
public class DriverManager {
    private static final ThreadLocal<WebDriver> driver      = new ThreadLocal<>();
    private static final ThreadLocal<String>    testName    = new ThreadLocal<>();
    private static final ThreadLocal<String>    browserName = new ThreadLocal<>();

    public static void setDriver(WebDriver d)    { driver.set(d); }
    public static WebDriver getDriver()           { return driver.get(); }
    public static void setTestName(String name)   { testName.set(name); }
    public static String getTestName()            { return testName.get(); }

    public static void unload() {
        if (driver.get() != null) {
            driver.get().quit();
        }
        driver.remove();      // prevent ThreadLocal memory leak
        testName.remove();
        browserName.remove();
    }
}
```

---

### 8.3 ExecutorService for Parallel API Tests
```java
// Run 10 API calls in parallel, collect results
ExecutorService pool = Executors.newFixedThreadPool(5);
List<Future<ApiResponse<OrderDto>>> futures = new ArrayList<>();

for (String orderId : orderIds) {
    futures.add(pool.submit(() -> apiClient.get("/orders/" + orderId, OrderDto.class)));
}

// Collect all results, fail if any had errors
List<String> errors = new ArrayList<>();
for (Future<ApiResponse<OrderDto>> f : futures) {
    try {
        ApiResponse<OrderDto> resp = f.get(10, TimeUnit.SECONDS);
        if (resp.getStatusCode() != 200) errors.add("Order " + resp.getBody().getId() + " returned " + resp.getStatusCode());
    } catch (TimeoutException e) {
        errors.add("API call timed out");
    }
}
pool.shutdown();

assertTrue(errors.isEmpty(), "API errors:\n" + String.join("\n", errors));
```

---

### 8.4 CompletableFuture for Async API Testing
```java
// Parallel calls with timeout — clean and non-blocking
CompletableFuture<User>   userFuture   = CompletableFuture.supplyAsync(() -> apiClient.getUser(userId));
CompletableFuture<Order[]> orderFuture = CompletableFuture.supplyAsync(() -> apiClient.getOrders(userId));

CompletableFuture.allOf(userFuture, orderFuture)
    .orTimeout(10, TimeUnit.SECONDS)
    .thenRun(() -> {
        User    user   = userFuture.join();
        Order[] orders = orderFuture.join();
        assertEquals(orders[0].getUserId(), user.getId());
    })
    .exceptionally(ex -> { fail("Async test failed: " + ex.getMessage()); return null; })
    .join();
```

---

### 8.5 Semaphore — Throttle Parallel Tests
```java
// Allow max 3 parallel driver sessions to avoid resource exhaustion
private static final Semaphore driverPermits = new Semaphore(3);

@BeforeMethod
public void setUp() throws InterruptedException {
    driverPermits.acquire();
    WebDriver driver = BrowserFactory.createDriver();
    DriverManager.setDriver(driver);
}

@AfterMethod(alwaysRun = true)
public void tearDown() {
    DriverManager.unload();
    driverPermits.release();
}
```

---

### 8.6 CountDownLatch — Wait for Parallel Setup
```java
// Seed test data in parallel before tests start
CountDownLatch setupLatch = new CountDownLatch(3);

executor.submit(() -> { seedUsers();    setupLatch.countDown(); });
executor.submit(() -> { seedProducts(); setupLatch.countDown(); });
executor.submit(() -> { seedOrders();   setupLatch.countDown(); });

boolean ready = setupLatch.await(30, TimeUnit.SECONDS);
assertTrue(ready, "Test data setup timed out after 30s");
// All data ready — tests can start
```

---

### 8.7 volatile for Test Status Flags
```java
// Shared flag across threads (visibility guaranteed)
private volatile boolean stopTestExecution = false;

// In a listener — set from any thread
public void onTestFailure(ITestResult result) {
    if (isCriticalTest(result)) stopTestExecution = true;
}

// In each test — check before executing
@BeforeMethod
public void checkGlobalStatus() {
    if (stopTestExecution) throw new SkipException("Skipped: critical test failed upstream");
}
```

---

### 8.8 synchronized for Shared Test Resources
```java
public class ScreenshotManager {
    private static int screenshotCount = 0;

    public static synchronized String nextName(String testName) {
        return testName + "_" + (++screenshotCount) + "_" + System.currentTimeMillis() + ".png";
    }
}

// Or use AtomicInteger (lock-free, preferred)
private static final AtomicInteger counter = new AtomicInteger(0);
String name = testName + "_" + counter.incrementAndGet() + ".png";
```

---

## 9. Java 8+ for Clean Test Code

### 9.1 Lambdas & Method References in Tests
```java
// Condition polling with lambda
waitUntil(() -> driver.findElements(By.cssSelector(".spinner")).isEmpty(), 10);

// Filter test data
List<User> activeAdmins = users.stream()
    .filter(u -> u.isActive() && u.hasRole("ADMIN"))
    .collect(Collectors.toList());

// Method reference for cleaner assertions
List<String> productNames = page.getProductElements().stream()
    .map(WebElement::getText)          // method reference
    .map(String::trim)
    .filter(Predicate.not(String::isEmpty))
    .collect(Collectors.toList());

assertTrue(productNames.contains("iPhone 15"));
```

---

### 9.2 Streams for Test Data Processing
```java
// Extract and assert on a specific field from API response list
List<OrderDto> orders = apiClient.getOrdersForUser(userId);

// Verify all orders belong to expected user
boolean allMatch = orders.stream()
    .allMatch(o -> o.getUserId().equals(userId));
assertTrue(allMatch, "Found orders belonging to other users");

// Find failed orders
List<String> failedOrderIds = orders.stream()
    .filter(o -> "FAILED".equals(o.getStatus()))
    .map(OrderDto::getId)
    .collect(Collectors.toList());
assertTrue(failedOrderIds.isEmpty(), "Failed orders: " + failedOrderIds);

// Group test results by status for report
Map<String, Long> resultSummary = testResults.stream()
    .collect(Collectors.groupingBy(TestResult::getStatus, Collectors.counting()));
// {PASSED=45, FAILED=3, SKIPPED=2}

// Sort products by price for validation
List<String> expectedOrder = products.stream()
    .sorted(Comparator.comparing(Product::getPrice))
    .map(Product::getName)
    .collect(Collectors.toList());
assertEquals(page.getProductNames(), expectedOrder, "Products not sorted by price");
```

---

### 9.3 Optional for Null-Safe Test Helpers
```java
// Avoid NPE in element text extraction
public Optional<String> getElementText(By locator) {
    try {
        return Optional.of(driver.findElement(locator).getText().trim());
    } catch (NoSuchElementException e) {
        return Optional.empty();
    }
}

// Usage — fail with clear message if element absent
String welcomeText = getElementText(By.id("welcome-msg"))
    .orElseThrow(() -> new AssertionError("Welcome message element not found"));
assertTrue(welcomeText.contains("Welcome"));

// OR use default
String title = getElementText(By.cssSelector("h1")).orElse("ELEMENT_MISSING");
assertNotEquals(title, "ELEMENT_MISSING", "Page title element missing");
```

---

### 9.4 Functional Interfaces for Test Utilities
```java
// Predicate — reusable test conditions
Predicate<Product> inStock       = p -> p.getQuantity() > 0;
Predicate<Product> affordable    = p -> p.getPrice() < 100.0;
Predicate<Product> highValue     = p -> p.getPrice() > 500.0 && inStock.test(p);
Predicate<Product> inStockAffordable = inStock.and(affordable);

// Function — test data transformations
Function<String, String> normalize = s -> s.trim().toLowerCase().replaceAll("\\s+", " ");
assertEquals(normalize.apply(actualTitle), normalize.apply(expectedTitle));

// Consumer — reusable action sequences (e.g., fill form fields)
Consumer<WebDriver> acceptCookies = d -> {
    try { d.findElement(By.id("cookie-accept")).click(); }
    catch (NoSuchElementException ignored) {}
};

// Supplier — lazy test data (only created if needed)
Supplier<User> testUser = () -> UserFactory.create()
    .withRole("ADMIN")
    .withEmail("admin_" + System.currentTimeMillis() + "@test.com")
    .build();
```

---

### 9.5 var for Cleaner Test Code (Java 10+)
```java
// Verbose
List<Map<String, Object>> responseList = objectMapper.readValue(body, List.class);

// With var — type inferred, still type-safe
var responseList = objectMapper.readValue(body, List.class);
var productPage  = new ProductPage(DriverManager.getDriver());
var apiResponse  = apiClient.post("/checkout", payload, OrderDto.class);
```

---

## 10. Design Patterns for Test Automation

### 10.1 Singleton — DriverManager (Thread-Safe per Thread)
**Problem:** Multiple instances of WebDriver per thread waste resources.
**Solution:** Singleton per thread using `ThreadLocal`.

```java
public class DriverManager {
    private static final ThreadLocal<WebDriver> DRIVER = new ThreadLocal<>();

    private DriverManager() {}  // prevent instantiation

    public static void setDriver(WebDriver driver) { DRIVER.set(driver); }
    public static WebDriver getDriver()            { return DRIVER.get(); }
    public static void unload() {
        Optional.ofNullable(DRIVER.get()).ifPresent(WebDriver::quit);
        DRIVER.remove();
    }
}
```

**When to use:** Any shared resource per test thread (WebDriver, DB connection, API client, auth token).

---

### 10.2 Factory + Enum — Browser/Driver Creation
**Problem:** Tests shouldn't know how drivers are created; environment-specific config should be centralized.

```java
public enum BrowserType {
    CHROME {
        public WebDriver create(boolean headless) {
            ChromeOptions opts = new ChromeOptions();
            if (headless) opts.addArguments("--headless=new", "--no-sandbox");
            return new ChromeDriver(opts);
        }
    },
    FIREFOX {
        public WebDriver create(boolean headless) {
            FirefoxOptions opts = new FirefoxOptions();
            if (headless) opts.addArguments("-headless");
            return new FirefoxDriver(opts);
        }
    },
    REMOTE {
        public WebDriver create(boolean headless) {
            DesiredCapabilities caps = new DesiredCapabilities();
            caps.setBrowserName(System.getProperty("remote.browser", "chrome"));
            try {
                return new RemoteWebDriver(new URL(ConfigLoader.get("grid.url")), caps);
            } catch (MalformedURLException e) {
                throw new TestConfigException("Invalid grid URL");
            }
        }
    };

    public abstract WebDriver create(boolean headless);
}

// BaseTest usage — driven by CI property
BrowserType type   = BrowserType.valueOf(System.getProperty("browser", "CHROME").toUpperCase());
boolean headless   = Boolean.parseBoolean(System.getProperty("headless", "false"));
WebDriver driver   = type.create(headless);
```

---

### 10.3 Builder — Test Data Objects
**Problem:** Creating complex test entities with many optional fields leads to multiple constructors or hard-to-read setup code.

```java
public class UserTestData {
    private final String email;
    private final String password;
    private final String role;
    private final String firstName;
    private final boolean active;

    private UserTestData(Builder b) {
        this.email     = b.email;
        this.password  = b.password;
        this.role      = b.role;
        this.firstName = b.firstName;
        this.active    = b.active;
    }

    public static class Builder {
        private String email     = "test_" + System.currentTimeMillis() + "@example.com";
        private String password  = "Test@123";
        private String role      = "USER";
        private String firstName = "Test";
        private boolean active   = true;

        public Builder email(String e)      { this.email = e;     return this; }
        public Builder password(String p)   { this.password = p;  return this; }
        public Builder role(String r)       { this.role = r;       return this; }
        public Builder admin()              { this.role = "ADMIN"; return this; }
        public Builder inactive()           { this.active = false; return this; }

        public UserTestData build()         { return new UserTestData(this); }
    }
}

// Test usage
UserTestData admin   = new UserTestData.Builder().admin().email("admin@test.com").build();
UserTestData expired = new UserTestData.Builder().inactive().build();
```

---

### 10.4 Strategy — Test Flows / Authentication
**Problem:** Different user types follow different login flows; test shouldn't know which strategy to use.

```java
public interface AuthStrategy {
    void authenticate(WebDriver driver, String username, String password);
}

public class StandardLoginStrategy implements AuthStrategy {
    public void authenticate(WebDriver driver, String username, String password) {
        new LoginPage(driver).loginAs(username, password);
    }
}

public class SSOLoginStrategy implements AuthStrategy {
    public void authenticate(WebDriver driver, String username, String password) {
        new SSOPage(driver).loginWithSSO(username, password);
    }
}

public class ApiTokenStrategy implements AuthStrategy {
    public void authenticate(WebDriver driver, String username, String password) {
        String token = apiClient.getToken(username, password);
        driver.manage().addCookie(new Cookie("auth_token", token));
        driver.navigate().refresh();
    }
}

// BaseTest — picks strategy from config
AuthStrategy auth = AuthStrategyFactory.get(ConfigLoader.get("auth.type"));
auth.authenticate(driver, testUser.getEmail(), testUser.getPassword());
```

---

### 10.5 Observer/Listener — Reporting & Screenshots
**Problem:** Screenshot capture on failure, Extent/Allure report updates should happen automatically without test code changes.

```java
// TestNG ITestListener implementation
public class TestListener implements ITestListener {

    @Override
    public void onTestFailure(ITestResult result) {
        // Auto screenshot on failure
        String screenshotPath = ScreenshotUtil.capture(
            DriverManager.getDriver(), result.getName());

        // Auto attach to report
        ExtentTestManager.getTest()
            .fail(result.getThrowable())
            .addScreenCaptureFromPath(screenshotPath);
    }

    @Override
    public void onTestStart(ITestResult result) {
        ExtentTestManager.startTest(result.getName(), result.getMethod().getDescription());
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        ExtentTestManager.getTest().pass("Test passed");
    }

    @Override
    public void onFinish(ITestContext context) {
        ExtentReportManager.flush();
    }
}

// Apply listener at suite level
// @Listeners(TestListener.class)  on BaseTest, OR in testng.xml:
// <listeners><listener class-name="com.framework.listeners.TestListener"/></listeners>
```

---

### 10.6 Template Method — BaseTest Lifecycle
**Problem:** Every test needs setup and teardown. Without a template, code is duplicated in each test class.

```java
public abstract class BaseTest {

    // Template — defines fixed skeleton of test lifecycle
    @BeforeSuite
    public final void globalSetup() {
        ExtentReportManager.init();
        ConfigLoader.load();
    }

    @BeforeMethod
    public final void setUp(Method method) {
        String browser = ConfigLoader.get("browser");
        WebDriver driver = BrowserType.valueOf(browser).create(isHeadless());
        DriverManager.setDriver(driver);
        DriverManager.getDriver().get(ConfigLoader.get("base.url"));
        ExtentTestManager.startTest(method.getName());
        doBeforeMethod(method);     // hook for subclass customization
    }

    @AfterMethod(alwaysRun = true)
    public final void tearDown(ITestResult result) {
        doAfterMethod(result);      // hook
        DriverManager.unload();
        ExtentTestManager.endTest(result);
    }

    // Hook methods — subclasses override selectively
    protected void doBeforeMethod(Method method) { }
    protected void doAfterMethod(ITestResult result) { }
    protected boolean isHeadless() {
        return Boolean.parseBoolean(ConfigLoader.get("headless"));
    }
}
```

---

### 10.7 Decorator — WebDriver Wrapper for Logging & Screenshots
**Problem:** Add logging, screenshots, and retry behavior to WebDriver actions without modifying every test or page class.

```java
public class LoggingWebDriver implements WebDriver {
    private final WebDriver wrapped;

    public LoggingWebDriver(WebDriver driver) { this.wrapped = driver; }

    @Override
    public void get(String url) {
        ExtentTestManager.getTest().info("Navigating to: " + url);
        wrapped.get(url);
    }

    @Override
    public WebElement findElement(By by) {
        try {
            return wrapped.findElement(by);
        } catch (NoSuchElementException e) {
            ScreenshotUtil.capture(wrapped, "element_not_found");
            throw new ElementNotFoundException("No element for: " + by);
        }
    }

    @Override  // delegate all other methods
    public String getTitle()         { return wrapped.getTitle(); }
    public String getCurrentUrl()    { return wrapped.getCurrentUrl(); }
    // ... implement all WebDriver methods
}

// Usage
WebDriver driver = new LoggingWebDriver(BrowserType.CHROME.create(false));
DriverManager.setDriver(driver);
```

---

### 10.8 Page Object Model (POM) Full Example
```java
// Page class hierarchy
BasePage
  └── LoginPage
  └── DashboardPage
  └── ProductPage
        └── ProductDetailPage

// Test
public class AddToCartTest extends BaseTest {

    @Test(description = "User can add in-stock product to cart")
    public void addProductToCartTest() {
        int cartCount = new LoginPage(DriverManager.getDriver())
            .loginAs("user@test.com", "Pass@123")
            .goToProducts()
            .searchFor("iPhone 15")
            .openFirstResult()
            .addToCart()
            .getCartItemCount();

        assertEquals(cartCount, 1, "Cart should have 1 item after adding product");
    }
}
```

---

## 11. Spring Boot for API & Integration Testing

### 11.1 @SpringBootTest — Integration Testing
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestPropertySource(locations = "classpath:test-application.properties")
public class OrderApiIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;  // auto-wired test HTTP client

    @Autowired
    private OrderRepository orderRepository;

    @BeforeEach
    public void setUp() {
        orderRepository.deleteAll();  // clean state before each test
    }

    @Test
    public void createOrder_returnsCreated() {
        OrderRequest request = new OrderRequest("PROD001", 2, "user123");

        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/orders", request, OrderResponse.class);

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody().getOrderId());
        assertEquals("PENDING", response.getBody().getStatus());
    }
}
```

---

### 11.2 @MockBean — Mocking Dependencies in Spring Tests
```java
@SpringBootTest
public class OrderServiceTest {

    @MockBean
    private PaymentService paymentService;  // replaces real PaymentService bean

    @MockBean
    private InventoryService inventoryService;

    @Autowired
    private OrderService orderService;      // the service under test

    @Test
    public void placeOrder_whenPaymentFails_throwsException() {
        when(inventoryService.isAvailable("PROD001", 1)).thenReturn(true);
        when(paymentService.charge(any())).thenThrow(new PaymentException("Card declined"));

        assertThrows(OrderException.class,
            () -> orderService.placeOrder(new OrderRequest("PROD001", 1, "user1")));

        verify(inventoryService, never()).reserve(any());  // reservation skipped on payment failure
    }
}
```

---

### 11.3 @DataJpaTest — Repository Testing
```java
@DataJpaTest   // loads only JPA layer — fast, no web layer
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)  // H2 in-memory
public class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Test
    public void findByEmail_returnsUser() {
        User saved = userRepository.save(new User("john@test.com", "John", "ACTIVE"));

        Optional<User> found = userRepository.findByEmail("john@test.com");

        assertTrue(found.isPresent());
        assertEquals("John", found.get().getName());
    }
}
```

---

### 11.4 @Transactional in Tests — Automatic Rollback
```java
@SpringBootTest
@Transactional  // each test method runs in a transaction that rolls back on completion
public class ProductServiceTest {

    @Autowired private ProductService productService;
    @Autowired private ProductRepository productRepository;

    @Test
    public void updatePrice_persistsChange() {
        Product p = productRepository.save(new Product("Widget", 10.00));

        productService.updatePrice(p.getId(), 15.00);

        Product updated = productRepository.findById(p.getId()).orElseThrow();
        assertEquals(15.00, updated.getPrice());
        // Transaction rolls back after test — DB left clean
    }
}
```

---

### 11.5 @WebMvcTest — Controller Layer Testing
```java
@WebMvcTest(OrderController.class)  // loads ONLY web layer
public class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @Test
    public void getOrder_returns200WithBody() throws Exception {
        OrderDto dto = new OrderDto("ORD001", "DELIVERED", 250.00);
        when(orderService.findById("ORD001")).thenReturn(dto);

        mockMvc.perform(get("/api/orders/ORD001")
                   .contentType(MediaType.APPLICATION_JSON))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.orderId").value("ORD001"))
               .andExpect(jsonPath("$.status").value("DELIVERED"))
               .andExpect(jsonPath("$.amount").value(250.00));
    }

    @Test
    public void createOrder_withInvalidBody_returns400() throws Exception {
        mockMvc.perform(post("/api/orders")
                   .contentType(MediaType.APPLICATION_JSON)
                   .content("{}"))  // missing required fields
               .andExpect(status().isBadRequest());
    }
}
```

---

### 11.6 REST API Structure — What SDETs Need to Know for API Testing
```java
// Understanding controller structure helps design API tests
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @GetMapping                                    // GET /api/v1/products
    public List<ProductDto> getAll() { ... }

    @GetMapping("/{id}")                           // GET /api/v1/products/123
    public ResponseEntity<ProductDto> getById(@PathVariable Long id) { ... }

    @PostMapping                                   // POST /api/v1/products
    public ResponseEntity<ProductDto> create(
        @Valid @RequestBody ProductRequest req) { ... }

    @PutMapping("/{id}")                           // PUT /api/v1/products/123
    public ProductDto update(@PathVariable Long id,
                             @RequestBody ProductRequest req) { ... }

    @DeleteMapping("/{id}")                        // DELETE /api/v1/products/123
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { ... }

    @GetMapping("/search")                         // GET /api/v1/products/search?category=electronics
    public Page<ProductDto> search(
        @RequestParam String category,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) { ... }
}
```

**HTTP status codes for test assertions:**

| Status | Meaning | When to Assert |
|---|---|---|
| 200 OK | Success (GET) | Successful retrieval |
| 201 Created | Resource created | POST creates entity |
| 204 No Content | Success, no body | DELETE, some PUT |
| 400 Bad Request | Invalid input | Missing fields, validation fail |
| 401 Unauthorized | Not authenticated | Missing/invalid auth token |
| 403 Forbidden | Authenticated but no permission | Wrong role access |
| 404 Not Found | Resource missing | Get non-existent ID |
| 409 Conflict | State conflict | Duplicate creation |
| 422 Unprocessable | Semantic validation fail | Business rule violation |
| 500 Server Error | Unexpected server error | Should never appear in happy path |

---

## 12. Test Architect — Additional Topics to Cover

> The following topics go beyond SDET scope and are relevant for **Lead SDET** to **Test Architect** level. Adding these to the reference document will make it a complete career progression guide.

---

### 12.1 Test Pyramid & Strategy
- **Unit Tests (70%):** Fast, isolated, developer-written. Mock all external dependencies.
- **Integration Tests (20%):** Verify components work together (DB + Service, Service + API). Use Testcontainers for real DB/Kafka in CI.
- **E2E Tests (10%):** Full user journey through UI or API. Slow, fragile — keep minimal, cover only critical flows.
- **Test Strategy Document:** Scope, risk assessment, entry/exit criteria, test types per layer, tooling decisions.

---

### 12.2 BDD with Cucumber / Gherkin
```gherkin
Feature: User Login
  Scenario Outline: Valid login redirects to dashboard
    Given I am on the login page
    When I login with "<username>" and "<password>"
    Then I should see the "<page>" page

    Examples:
      | username         | password  | page      |
      | admin@test.com   | Admin@123 | Dashboard |
      | user@test.com    | User@123  | Home      |
```
- Step definitions in Java using `@Given`, `@When`, `@Then`
- Page Object pattern integrates naturally with Cucumber steps
- Hooks: `@Before`, `@After` per scenario for setup/teardown
- Tags: `@smoke`, `@regression`, `@P0` for selective execution

---

### 12.3 API Testing — REST Assured
```java
given()
    .baseUri("https://api.example.com")
    .header("Authorization", "Bearer " + token)
    .contentType(ContentType.JSON)
    .body(requestPayload)
.when()
    .post("/orders")
.then()
    .statusCode(201)
    .body("status", equalTo("PENDING"))
    .body("orderId", notNullValue())
    .time(lessThan(2000L));  // response time assertion
```
- Schema validation with JSON Schema: `.body(matchesJsonSchemaInClasspath("order-schema.json"))`
- Contract testing with Pact: consumer defines contract, provider verifies it in CI
- API versioning awareness (v1 vs v2) in test design

---

### 12.4 Test Data Management
- **Strategy:** Test data should be independent per test (not shared state that causes order-dependency).
- **Approaches:**
  - DB fixtures / SQL scripts (`@Sql` annotation in Spring)
  - API-based setup (call API to create data → test → teardown via API)
  - Builder pattern for in-memory data
  - Testcontainers for isolated DB per test run
- **Data cleanup:** `@Transactional` rollback, `@AfterEach` deletion, dedicated cleanup service

---

### 12.5 CI/CD Integration
```yaml
# GitHub Actions example
jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-java@v3
        with: { java-version: '17' }
      - name: Run tests
        run: mvn test -Pbrowser=CHROME -Dheadless=true -Denv=QA -Dgroups=regression
      - name: Publish Allure Report
        uses: simple-irobot/allure-report-action@v1
```
- Maven profiles for environment-specific config
- Parallel execution configuration in TestNG XML / Surefire plugin
- Test result gates: fail pipeline if pass rate drops below threshold
- Flaky test detection: retry plugin + tagging flaky tests

---

### 12.6 Cloud & Grid Testing
- **Selenium Grid:** Hub + Node architecture; Docker-based grid (`docker-compose up selenium-hub`)
- **Cloud providers:** BrowserStack, Sauce Labs, LambdaTest — cross-browser/OS matrix
- **Playwright / Cypress** awareness (modern alternatives to Selenium)
- **Appium** for mobile testing (iOS/Android)

---

### 12.7 Performance & Load Testing Basics
- **JMeter:** Thread groups, samplers, assertions, listeners, distributed load
- **Gatling (Scala/Java DSL):** Code-based load tests, integrated with CI
- **Key metrics:** Response time (p50, p95, p99), throughput (TPS), error rate, concurrent users
- **Non-functional test types:** Load, Stress, Soak, Spike testing

---

### 12.8 Containerized Testing with Testcontainers
```java
@SpringBootTest
@Testcontainers
public class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    public void saveOrder_persistsToRealDB() { ... }
}
```
- Also supports Kafka, Redis, MongoDB, LocalStack (AWS) containers
- Each test run gets isolated, real infrastructure — no mocking needed

---

### 12.9 Observability & Logging in Tests
- Structured test logging: log each step with test name, thread ID, timestamp
- Allure reporting: `@Step`, `@Attachment`, `@Description`, severity levels
- MDC (Mapped Diagnostic Context) for thread-specific log context in parallel runs:
  ```java
  MDC.put("testName", result.getName());
  MDC.put("threadId", String.valueOf(Thread.currentThread().getId()));
  ```
- Correlation IDs: pass `X-Correlation-ID` header in API tests to trace requests in server logs

---

### 12.10 Test Code Quality (SOLID in Automation)
| SOLID Principle | Test Framework Application |
|---|---|
| **S** – Single Responsibility | Each Page Object handles ONE page; each utility class does ONE thing |
| **O** – Open/Closed | Add new browsers via BrowserType enum — no existing code changes |
| **L** – Liskov Substitution | Any BasePage subclass must be usable where BasePage is expected |
| **I** – Interface Segregation | Don't force all pages to implement methods they don't use |
| **D** – Dependency Inversion | Test depends on `AuthStrategy` interface, not `StandardLoginStrategy` |

**Common test code smells to avoid:**
- `Thread.sleep()` — replace with explicit waits
- Hard-coded test data in test methods — use builders or data providers
- Test depending on execution order — each test must be independent
- Assertions inside page objects — assertions belong in test classes
- `driver.findElement()` in test classes — always through page objects

---

### 12.11 Microservices Testing Strategy
- **Consumer-Driven Contract Testing (Pact):** Consumer publishes contract; provider CI verifies it — decoupled from E2E
- **Component Testing:** Test each microservice in isolation with mocked dependencies (WireMock)
- **End-to-End Testing:** Full journey across services — Docker Compose to spin up all services
- **Chaos Engineering basics:** Simulate service failures to verify resilience (Resilience4j tests)

---

### 12.12 Framework Design Principles (Lead SDET / Architect Level)
- **Modularity:** Core module (driver, config, reporting) separate from test modules (UI, API, DB)
- **Configuration hierarchy:** `default.properties` → `env-specific.properties` → system properties (CI) — later overrides earlier
- **Reporting strategy:** Screenshots + logs + videos on failure; Allure/ExtentReports dashboard integrated with CI
- **Retry mechanism:** TestNG `IRetryAnalyzer` for known flaky scenarios — max 1 retry, log each retry
- **Fail-fast vs Fail-safe:** For smoke tests, fail suite on first failure; for regression, collect all failures
- **Version control:** Test code in same repo as application code (shift-left) OR separate repo with agreed contract
- **Framework documentation:** Onboarding guide, architecture diagram, contribution standards

---

*Document covers SDET → Senior SDET scope. Section 12 covers Lead SDET → Test Architect scope.*
*Source: Recap.docx — Java Complete Reference Guide*
