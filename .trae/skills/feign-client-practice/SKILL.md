---
name: "feign-client-practice"
description: "Feign Client best practices for microservices communication. Invoke when implementing service-to-service calls, configuring Feign clients, or handling external API integrations."
---

# Feign Client Best Practices

This skill provides comprehensive guidelines for implementing and managing Feign Clients in Spring Boot microservices architecture.

## Feign Client Configuration

### Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot2</artifactId>
    </dependency>
    
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-feign</artifactId>
    </dependency>
</dependencies>
```

### Enable Feign Clients

```java
@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

### Application Configuration

```yaml
# application.yml
spring:
  application:
    name: order-service
  cloud:
    loadbalancer:
      ribbon:
        enabled: false

feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
        loggerLevel: BASIC
      inventory-service:
        connectTimeout: 3000
        readTimeout: 10000
        loggerLevel: FULL
        
  circuitbreaker:
    enabled: true
    
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
    response:
      enabled: true

resilience4j:
  circuitbreaker:
    instances:
      inventory-service:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 3
        slidingWindowType: COUNT_BASED
        
  retry:
    instances:
      inventory-service:
        maxAttempts: 3
        waitDuration: 1000
        exponentialBackoffMultiplier: 2
        
  timelimiter:
    instances:
      inventory-service:
        timeoutDuration: 3s

logging:
  level:
    com.example.order.infrastructure.client: DEBUG
```

## Feign Client Definition

### Basic Feign Client

```java
package com.example.order.infrastructure.client;

@FeignClient(
    name = "inventory-service",
    url = "${feign.client.inventory-service.url}",
    configuration = InventoryServiceConfiguration.class,
    fallbackFactory = InventoryServiceFallbackFactory.class
)
public interface InventoryServiceClient {
    
    @GetMapping("/api/inventory/products/{productId}")
    ProductResponse getProduct(@PathVariable("productId") String productId);
    
    @GetMapping("/api/inventory/products")
    List<ProductResponse> getProducts(@RequestParam("category") String category);
    
    @PostMapping("/api/inventory/reserve")
    ReservationResponse reserveStock(@RequestBody ReserveStockRequest request);
    
    @PostMapping("/api/inventory/release")
    void releaseStock(@RequestBody ReleaseStockRequest request);
    
    @GetMapping("/api/inventory/products/{productId}/availability")
    AvailabilityResponse checkAvailability(
        @PathVariable("productId") String productId,
        @RequestParam("quantity") Integer quantity
    );
}
```

### Feign Client with Custom Configuration

```java
@Configuration
public class InventoryServiceConfiguration {
    
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
    
    @Bean
    public RequestInterceptor requestInterceptor() {
        return template -> {
            template.header("X-Service-Name", "order-service");
            template.header("X-Request-ID", UUID.randomUUID().toString());
        };
    }
    
    @Bean
    public ErrorDecoder errorDecoder() {
        return new InventoryServiceErrorDecoder();
    }
    
    @Bean
    public Retryer retryer() {
        return new Retryer.Default(
            100,   // initial interval
            1000,  // max interval
            3      // max attempts
        );
    }
    
    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            5000,  // connect timeout
            10000  // read timeout
        );
    }
}
```

## Error Handling

### Custom Error Decoder

```java
public class InventoryServiceErrorDecoder implements ErrorDecoder {
    
    private static final Logger logger = LoggerFactory.getLogger(InventoryServiceErrorDecoder.class);
    
    @Override
    public Exception decode(String methodKey, Response response) {
        int status = response.status();
        
        if (status >= 400 && status <= 499) {
            return handleClientError(response, status);
        }
        
        if (status >= 500) {
            return handleServerError(response, status);
        }
        
        return new Default().decode(methodKey, response);
    }
    
    private Exception handleClientError(Response response, int status) {
        String responseBody = extractResponseBody(response);
        
        switch (status) {
            case 400:
                return new BadRequestException("Invalid request: " + responseBody);
            case 401:
                return new UnauthorizedException("Unauthorized access");
            case 403:
                return new ForbiddenException("Access forbidden");
            case 404:
                return new ProductNotFoundException("Resource not found: " + responseBody);
            case 409:
                return new ConflictException("Conflict: " + responseBody);
            case 429:
                return new RateLimitExceededException("Rate limit exceeded");
            default:
                return new InventoryServiceClientException(
                    "Client error: " + status + ", body: " + responseBody
                );
        }
    }
    
    private Exception handleServerError(Response response, int status) {
        String responseBody = extractResponseBody(response);
        logger.error("Server error from inventory service: status={}, body={}", 
            status, responseBody);
        
        return new InventoryServiceException(
            "Server error: " + status + ", body: " + responseBody
        );
    }
    
    private String extractResponseBody(Response response) {
        try {
            if (response.body() != null) {
                return Util.toString(response.body().asReader(StandardCharsets.UTF_8));
            }
        } catch (IOException e) {
            logger.error("Error reading response body", e);
        }
        return "";
    }
}
```

### Fallback Implementation

```java
@Component
public class InventoryServiceFallbackFactory implements FallbackFactory<InventoryServiceClient> {
    
    private static final Logger logger = LoggerFactory.getLogger(InventoryServiceFallbackFactory.class);
    
    @Override
    public InventoryServiceClient create(Throwable cause) {
        return new InventoryServiceClient() {
            
            @Override
            public ProductResponse getProduct(String productId) {
                logger.error("Fallback triggered for getProduct, productId: {}", productId, cause);
                throw new InventoryServiceUnavailableException(
                    "Inventory service is currently unavailable", cause
                );
            }
            
            @Override
            public List<ProductResponse> getProducts(String category) {
                logger.error("Fallback triggered for getProducts, category: {}", category, cause);
                return Collections.emptyList();
            }
            
            @Override
            public ReservationResponse reserveStock(ReserveStockRequest request) {
                logger.error("Fallback triggered for reserveStock", cause);
                throw new InventoryServiceUnavailableException(
                    "Cannot reserve stock - service unavailable", cause
                );
            }
            
            @Override
            public void releaseStock(ReleaseStockRequest request) {
                logger.error("Fallback triggered for releaseStock", cause);
            }
            
            @Override
            public AvailabilityResponse checkAvailability(String productId, Integer quantity) {
                logger.error("Fallback triggered for checkAvailability", cause);
                return new AvailabilityResponse(false, 0);
            }
        };
    }
}
```

## Circuit Breaker Pattern

### Circuit Breaker Configuration

```java
@Configuration
public class CircuitBreakerConfiguration {
    
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> globalCustomizer() {
        return factory -> {
            factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
                .circuitBreakerConfig(CircuitBreakerConfig.ofDefaults())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(3))
                    .build())
                .build());
        };
    }
    
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> inventoryServiceCustomizer() {
        return factory -> {
            factory.configure(builder -> builder
                .circuitBreakerConfig(CircuitBreakerConfig.custom()
                    .slidingWindowSize(10)
                    .failureRateThreshold(50)
                    .waitDurationInOpenState(Duration.ofSeconds(10))
                    .permittedNumberOfCallsInHalfOpenState(3)
                    .slidingWindowType(SlidingWindowType.COUNT_BASED)
                    .build())
                .timeLimiterConfig(TimeLimiterConfig.custom()
                    .timeoutDuration(Duration.ofSeconds(5))
                    .build()),
                "inventory-service"
            );
        };
    }
}
```

### Circuit Breaker with Fallback

```java
@Service
public class InventoryServiceAdapter {
    
    private final InventoryServiceClient inventoryClient;
    
    public InventoryServiceAdapter(InventoryServiceClient inventoryClient) {
        this.inventoryClient = inventoryClient;
    }
    
    @CircuitBreaker(
        name = "inventory-service",
        fallbackMethod = "getProductFallback"
    )
    @Retry(name = "inventory-service")
    @RateLimiter(name = "inventory-service")
    public Product getProduct(ProductId productId) {
        ProductResponse response = inventoryClient.getProduct(productId.getValue());
        return mapToProduct(response);
    }
    
    private Product getProductFallback(ProductId productId, Throwable t) {
        log.warn("Fallback triggered for getProduct, productId: {}", productId, t);
        
        return getFromCache(productId)
            .orElseThrow(() -> new InventoryServiceUnavailableException(
                "Unable to retrieve product information", t
            ));
    }
    
    private Optional<Product> getFromCache(ProductId productId) {
        // Return cached product if available
        return Optional.empty();
    }
    
    private Product mapToProduct(ProductResponse response) {
        return new Product(
            ProductId.of(response.getProductId()),
            response.getName(),
            response.getDescription(),
            new Money(response.getPrice(), Currency.getInstance(response.getCurrency())),
            Quantity.of(response.getAvailableStock())
        );
    }
}
```

## Request/Response Handling

### Request Interceptor

```java
@Component
public class FeignRequestInterceptor implements RequestInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(FeignRequestInterceptor.class);
    
    @Override
    public void apply(RequestTemplate template) {
        addCorrelationId(template);
        addAuthentication(template);
        addServiceHeaders(template);
        logRequest(template);
    }
    
    private void addCorrelationId(RequestTemplate template) {
        String correlationId = MDC.get("correlationId");
        if (correlationId != null) {
            template.header("X-Correlation-ID", correlationId);
        } else {
            template.header("X-Correlation-ID", UUID.randomUUID().toString());
        }
    }
    
    private void addAuthentication(RequestTemplate template) {
        String token = SecurityContextHolder.getContext()
            .getAuthentication()
            .getCredentials()
            .toString();
        
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
    }
    
    private void addServiceHeaders(RequestTemplate template) {
        template.header("X-Service-Name", "order-service");
        template.header("X-Service-Version", "1.0.0");
    }
    
    private void logRequest(RequestTemplate template) {
        logger.debug("Feign request: {} {}",
            template.method(),
            template.url()
        );
    }
}
```

### Response Interceptor

```java
@Component
public class FeignResponseInterceptor implements ResponseInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(FeignResponseInterceptor.class);
    
    @Override
    public Response intercept(InvocationContext context, Chain chain) throws IOException {
        long startTime = System.currentTimeMillis();
        
        try {
            Response response = chain.next(context);
            
            logResponse(response, startTime);
            extractCorrelationId(response);
            
            return response;
            
        } catch (Exception e) {
            logError(e, startTime);
            throw e;
        }
    }
    
    private void logResponse(Response response, long startTime) {
        long duration = System.currentTimeMillis() - startTime;
        
        logger.info("Feign response: status={}, duration={}ms, url={}",
            response.status(),
            duration,
            response.request().url()
        );
    }
    
    private void extractCorrelationId(Response response) {
        Collection<String> correlationIds = response.headers().get("X-Correlation-ID");
        if (correlationIds != null && !correlationIds.isEmpty()) {
            String correlationId = correlationIds.iterator().next();
            MDC.put("correlationId", correlationId);
        }
    }
    
    private void logError(Exception e, long startTime) {
        long duration = System.currentTimeMillis() - startTime;
        logger.error("Feign request failed after {}ms", duration, e);
    }
}
```

## DTO Definitions

### Request DTOs

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ReserveStockRequest {
    
    private String orderId;
    private List<StockItem> items;
    
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class StockItem {
        private String productId;
        private Integer quantity;
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class ReleaseStockRequest {
    
    private String orderId;
    private String reason;
}
```

### Response DTOs

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ProductResponse {
    
    private String productId;
    private String name;
    private String description;
    private BigDecimal price;
    private String currency;
    private Integer availableStock;
    private String category;
    private Instant lastUpdated;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class ReservationResponse {
    
    private String reservationId;
    private String orderId;
    private ReservationStatus status;
    private Instant reservedAt;
    
    public enum ReservationStatus {
        RESERVED,
        FAILED,
        PARTIAL
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class AvailabilityResponse {
    
    private Boolean available;
    private Integer availableQuantity;
}
```

## Testing Feign Clients

### Unit Testing with Mock

```java
@ExtendWith(MockitoExtension.class)
class InventoryServiceAdapterTest {
    
    @Mock
    private InventoryServiceClient inventoryClient;
    
    @InjectMocks
    private InventoryServiceAdapter inventoryAdapter;
    
    @Test
    @DisplayName("Should get product successfully")
    void shouldGetProductSuccessfully() {
        ProductId productId = ProductId.generate();
        ProductResponse response = new ProductResponse(
            productId.getValue(),
            "Product Name",
            "Description",
            BigDecimal.valueOf(100.00),
            "USD",
            10,
            "ELECTRONICS",
            Instant.now()
        );
        
        when(inventoryClient.getProduct(productId.getValue())).thenReturn(response);
        
        Product product = inventoryAdapter.getProduct(productId);
        
        assertNotNull(product);
        assertEquals(productId, product.getId());
        assertEquals("Product Name", product.getName());
        assertEquals(Quantity.of(10), product.getAvailableStock());
    }
    
    @Test
    @DisplayName("Should throw exception when product not found")
    void shouldThrowExceptionWhenProductNotFound() {
        ProductId productId = ProductId.generate();
        
        when(inventoryClient.getProduct(productId.getValue()))
            .thenThrow(new ProductNotFoundException("Product not found"));
        
        assertThrows(ProductNotFoundException.class, () -> {
            inventoryAdapter.getProduct(productId);
        });
    }
}
```

### Integration Testing with WireMock

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class InventoryServiceClientIntegrationTest {
    
    @Autowired
    private InventoryServiceClient inventoryClient;
    
    @Value("${wiremock.server.port}")
    private int wireMockPort;
    
    @BeforeEach
    void setUp() {
        stubFor(get(urlPathEqualTo("/api/inventory/products/PROD-123"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"productId\":\"PROD-123\",\"name\":\"Test Product\",\"price\":100.00,\"currency\":\"USD\",\"availableStock\":10}")));
    }
    
    @Test
    @DisplayName("Should get product from inventory service")
    void shouldGetProductFromInventoryService() {
        ProductResponse response = inventoryClient.getProduct("PROD-123");
        
        assertNotNull(response);
        assertEquals("PROD-123", response.getProductId());
        assertEquals("Test Product", response.getName());
        assertEquals(BigDecimal.valueOf(100.00), response.getPrice());
    }
    
    @Test
    @DisplayName("Should handle 404 error")
    void shouldHandle404Error() {
        stubFor(get(urlPathEqualTo("/api/inventory/products/INVALID"))
            .willReturn(aResponse()
                .withStatus(404)));
        
        assertThrows(ProductNotFoundException.class, () -> {
            inventoryClient.getProduct("INVALID");
        });
    }
    
    @Test
    @DisplayName("Should handle server error")
    void shouldHandleServerError() {
        stubFor(get(urlPathEqualTo("/api/inventory/products/SERVER_ERROR"))
            .willReturn(aResponse()
                .withStatus(500)));
        
        assertThrows(InventoryServiceException.class, () -> {
            inventoryClient.getProduct("SERVER_ERROR");
        });
    }
}
```

## Best Practices

### 1. Client Design
- Use meaningful names for Feign clients
- Separate clients by service/domain
- Use configuration classes for custom settings
- Define clear request/response DTOs

### 2. Error Handling
- Implement custom error decoders
- Use fallback mechanisms for resilience
- Log errors appropriately
- Provide meaningful error messages

### 3. Resilience
- Configure circuit breakers appropriately
- Implement retry logic with exponential backoff
- Set reasonable timeouts
- Use rate limiting for protection

### 4. Security
- Use authentication interceptors
- Never hardcode credentials
- Validate and sanitize inputs
- Use HTTPS in production

### 5. Monitoring
- Log requests and responses
- Track metrics and performance
- Monitor circuit breaker state
- Set up alerts for failures

### 6. Testing
- Mock Feign clients in unit tests
- Use WireMock for integration tests
- Test error scenarios
- Test circuit breaker behavior

## Common Patterns

### Service Adapter Pattern

```java
@Service
public class InventoryServiceAdapter {
    
    private final InventoryServiceClient client;
    private final InventoryCache cache;
    
    public Product getProduct(ProductId productId) {
        try {
            ProductResponse response = client.getProduct(productId.getValue());
            Product product = mapToProduct(response);
            cache.put(productId, product);
            return product;
        } catch (InventoryServiceException e) {
            return cache.get(productId)
                .orElseThrow(() -> e);
        }
    }
}
```

### Bulkhead Pattern

```java
@Service
public class InventoryServiceAdapter {
    
    private final InventoryServiceClient client;
    
    @Bulkhead(name = "inventory-service", type = Bulkhead.Type.SEMAPHORE)
    public List<Product> getProducts(List<ProductId> productIds) {
        return productIds.parallelStream()
            .map(this::getProduct)
            .collect(Collectors.toList());
    }
}
```

## References

- [Spring Cloud OpenFeign Documentation](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/)
- [Resilience4j Documentation](https://resilience4j.readme.io/docs)
- [Feign GitHub Repository](https://github.com/OpenFeign/feign)
- [Microservices Patterns](https://microservices.io/patterns/index.html)
