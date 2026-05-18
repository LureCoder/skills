---
name: "java-logging-docs"
description: "Java logging and documentation best practices. Invoke when implementing logging, writing API documentation, or creating code documentation."
---

# Java Logging and Documentation Best Practices

This skill provides comprehensive guidelines for implementing effective logging and documentation strategies in Java Spring Boot applications.

## Logging Best Practices

### Logger Configuration

#### Logback Configuration (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_PATH" value="logs"/>
    <property name="APP_NAME" value="order-service"/>
    
    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- File Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>30</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
    </appender>
    
    <!-- Error File Appender -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>90</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- JSON Appender for ELK Stack -->
    <appender name="JSON" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}-json.log</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app_name":"${APP_NAME}"}</customFields>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}-json.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- Logger Levels -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.axonframework" level="INFO"/>
    <logger name="org.springframework" level="INFO"/>
    <logger name="org.hibernate" level="WARN"/>
    
    <!-- Root Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="ERROR_FILE"/>
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

### Structured Logging

```java
package com.example.order.infrastructure.logging;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component
public class OrderLogger {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderLogger.class);
    
    public void logOrderCreated(OrderId orderId, CustomerId customerId) {
        logger.info("Order created - orderId: {}, customerId: {}, timestamp: {}",
            orderId.getValue(),
            customerId.getValue(),
            Instant.now()
        );
    }
    
    public void logOrderConfirmed(OrderId orderId) {
        logger.info("Order confirmed - orderId: {}", orderId.getValue());
    }
    
    public void logOrderFailed(OrderId orderId, String reason, Exception exception) {
        logger.error("Order processing failed - orderId: {}, reason: {}",
            orderId.getValue(),
            reason,
            exception
        );
    }
    
    public void logPerformance(String operation, long durationMs) {
        logger.info("Performance metric - operation: {}, duration: {}ms",
            operation,
            durationMs
        );
    }
}
```

### MDC (Mapped Diagnostic Context)

```java
@Component
public class OrderContextInterceptor implements HandlerInterceptor {
    
    private static final String ORDER_ID = "orderId";
    private static final String CUSTOMER_ID = "customerId";
    private static final String CORRELATION_ID = "correlationId";
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                             HttpServletResponse response, 
                             Object handler) {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }
        
        MDC.put(CORRELATION_ID, correlationId);
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                                 HttpServletResponse response, 
                                 Object handler, 
                                 Exception ex) {
        MDC.remove(ORDER_ID);
        MDC.remove(CUSTOMER_ID);
        MDC.remove(CORRELATION_ID);
    }
}

@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(CreateOrderCommand command) {
        MDC.put("orderId", command.getOrderId().getValue());
        MDC.put("customerId", command.getCustomerId().getValue());
        
        try {
            logger.info("Creating order");
            
            Order order = new Order(command);
            
            logger.info("Order created successfully");
            return order;
            
        } catch (Exception e) {
            logger.error("Failed to create order", e);
            throw e;
        } finally {
            MDC.remove("orderId");
            MDC.remove("customerId");
        }
    }
}
```

### Logging Levels

```java
public class LoggingExamples {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingExamples.class);
    
    public void demonstrateLoggingLevels() {
        // TRACE - Very detailed information
        logger.trace("Entering method processOrder with parameters: {}", parameters);
        
        // DEBUG - Detailed information for debugging
        logger.debug("Processing order item: productId={}, quantity={}", 
            productId, quantity);
        
        // INFO - Important business events
        logger.info("Order {} created successfully for customer {}", 
            orderId, customerId);
        
        // WARN - Potentially harmful situations
        logger.warn("Order {} has been pending for {} hours", 
            orderId, pendingHours);
        
        // ERROR - Error events that might still allow the application to continue
        logger.error("Failed to process payment for order {}", orderId, exception);
    }
}
```

### Exception Logging

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    private static final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleOrderNotFound(OrderNotFoundException ex) {
        logger.warn("Order not found: orderId={}", ex.getOrderId());
        
        ErrorResponse error = new ErrorResponse(
            "ORDER_NOT_FOUND",
            "Order not found: " + ex.getOrderId(),
            Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(PaymentFailedException.class)
    public ResponseEntity<ErrorResponse> handlePaymentFailed(PaymentFailedException ex) {
        logger.error("Payment failed for order: orderId={}, reason={}",
            ex.getOrderId(),
            ex.getMessage(),
            ex
        );
        
        ErrorResponse error = new ErrorResponse(
            "PAYMENT_FAILED",
            "Payment processing failed",
            Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception ex) {
        logger.error("Unexpected error occurred", ex);
        
        ErrorResponse error = new ErrorResponse(
            "INTERNAL_ERROR",
            "An unexpected error occurred",
            Instant.now()
        );
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## API Documentation

### OpenAPI/Swagger Configuration

```java
@Configuration
public class OpenApiConfiguration {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Order Service API")
                .version("1.0.0")
                .description("API for managing orders in e-commerce platform")
                .contact(new Contact()
                    .name("API Support")
                    .email("api-support@example.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .externalDocs(new ExternalDocumentation()
                .description("Order Service Documentation")
                .url("https://docs.example.com/order-service"));
    }
    
    @Bean
    public GroupedOpenApi orderApi() {
        return GroupedOpenApi.builder()
            .group("orders")
            .pathsToMatch("/api/orders/**")
            .build();
    }
    
    @Bean
    public GroupedOpenApi paymentApi() {
        return GroupedOpenApi.builder()
            .group("payments")
            .pathsToMatch("/api/payments/**")
            .build();
    }
}
```

### Controller Documentation

```java
@RestController
@RequestMapping("/api/orders")
@Tag(name = "Order Management", description = "APIs for managing orders")
public class OrderController {
    
    @Operation(
        summary = "Create a new order",
        description = "Creates a new order for the specified customer with the given items",
        responses = {
            @ApiResponse(
                responseCode = "201",
                description = "Order created successfully",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = OrderDTO.class)
                )
            ),
            @ApiResponse(
                responseCode = "400",
                description = "Invalid request data",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = ErrorResponse.class)
                )
            ),
            @ApiResponse(
                responseCode = "404",
                description = "Customer not found",
                content = @Content(
                    mediaType = "application/json",
                    schema = @Schema(implementation = ErrorResponse.class)
                )
            )
        }
    )
    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(
        @Parameter(description = "Order creation request", required = true)
        @Valid @RequestBody CreateOrderRequest request
    ) {
        OrderDTO order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    @Operation(
        summary = "Get order by ID",
        description = "Retrieves order details for the specified order ID"
    )
    @ApiResponses(value = {
        @ApiResponse(
            responseCode = "200",
            description = "Order found",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = OrderDTO.class)
            )
        ),
        @ApiResponse(
            responseCode = "404",
            description = "Order not found",
            content = @Content(
                mediaType = "application/json",
                schema = @Schema(implementation = ErrorResponse.class)
            )
        )
    })
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDTO> getOrder(
        @Parameter(description = "Order ID", required = true, example = "ORD-123456")
        @PathVariable String orderId
    ) {
        OrderDTO order = orderService.findById(OrderId.of(orderId));
        return ResponseEntity.ok(order);
    }
    
    @Operation(
        summary = "Update order status",
        description = "Updates the status of an existing order"
    )
    @PatchMapping("/{orderId}/status")
    public ResponseEntity<OrderDTO> updateOrderStatus(
        @Parameter(description = "Order ID", required = true)
        @PathVariable String orderId,
        
        @Parameter(description = "New order status", required = true)
        @Valid @RequestBody UpdateOrderStatusRequest request
    ) {
        OrderDTO order = orderService.updateStatus(
            OrderId.of(orderId),
            request.getStatus()
        );
        return ResponseEntity.ok(order);
    }
}
```

### DTO Documentation

```java
@Schema(description = "Request for creating a new order")
public class CreateOrderRequest {
    
    @Schema(
        description = "Customer ID",
        example = "CUST-123456",
        required = true
    )
    @NotNull(message = "Customer ID is required")
    private String customerId;
    
    @Schema(
        description = "List of order items",
        required = true,
        minProperties = 1
    )
    @NotNull(message = "Order items are required")
    @Size(min = 1, message = "Order must have at least one item")
    private List<OrderItemRequest> items;
    
    @Schema(
        description = "Shipping address",
        required = true
    )
    @NotNull(message = "Shipping address is required")
    private ShippingAddressRequest shippingAddress;
    
    @Schema(
        description = "Payment method",
        example = "CREDIT_CARD",
        allowableValues = {"CREDIT_CARD", "DEBIT_CARD", "PAYPAL"}
    )
    private String paymentMethod;
    
    @Schema(
        description = "Coupon code for discount",
        example = "SAVE10"
    )
    private String couponCode;
}

@Schema(description = "Order item in the request")
public class OrderItemRequest {
    
    @Schema(
        description = "Product ID",
        example = "PROD-789",
        required = true
    )
    @NotNull(message = "Product ID is required")
    private String productId;
    
    @Schema(
        description = "Quantity",
        example = "2",
        minimum = "1",
        required = true
    )
    @NotNull(message = "Quantity is required")
    @Min(value = 1, message = "Quantity must be at least 1")
    private Integer quantity;
}

@Schema(description = "Order response DTO")
public class OrderDTO {
    
    @Schema(
        description = "Order ID",
        example = "ORD-123456"
    )
    private String orderId;
    
    @Schema(
        description = "Customer ID",
        example = "CUST-123456"
    )
    private String customerId;
    
    @Schema(
        description = "Order status",
        example = "CONFIRMED",
        allowableValues = {"CREATED", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"}
    )
    private String status;
    
    @Schema(
        description = "Total order amount",
        example = "299.99"
    )
    private BigDecimal totalAmount;
    
    @Schema(
        description = "Order creation timestamp",
        example = "2024-01-15T10:30:00Z"
    )
    private Instant createdAt;
    
    @Schema(description = "Order items")
    private List<OrderItemDTO> items;
}
```

## Code Documentation

### Javadoc Standards

```java
/**
 * Service for managing orders in the e-commerce system.
 * 
 * <p>This service provides functionality for creating, updating, and querying orders.
 * It follows the CQRS pattern using Axon Framework for command and query separation.</p>
 * 
 * <h2>Key Features:</h2>
 * <ul>
 *   <li>Order creation and validation</li>
 *   <li>Order status management</li>
 *   <li>Order querying and filtering</li>
 *   <li>Integration with payment and inventory services</li>
 * </ul>
 * 
 * <h2>Usage Example:</h2>
 * <pre>{@code
 * OrderApplicationService orderService = // obtain instance
 * 
 * CreateOrderRequest request = new CreateOrderRequest(
 *     customerId,
 *     Arrays.asList(
 *         new OrderItemRequest(productId, quantity)
 *     )
 * );
 * 
 * OrderId orderId = orderService.createOrder(request);
 * }</pre>
 * 
 * @author Development Team
 * @version 1.0.0
 * @since 2024-01-01
 * @see Order
 * @see OrderRepository
 * @see CreateOrderCommand
 */
@Service
public class OrderApplicationService {
    
    /**
     * Creates a new order for the specified customer.
     * 
     * <p>This method performs the following steps:</p>
     * <ol>
     *   <li>Validates the order request</li>
     *   <li>Creates a new order aggregate</li>
     *   <li>Persists the order to the event store</li>
     *   <li>Triggers order creation event handlers</li>
     * </ol>
     * 
     * @param request the order creation request containing customer ID and items
     * @return the generated order ID for the newly created order
     * @throws IllegalArgumentException if the request is null or contains invalid data
     * @throws CustomerNotFoundException if the specified customer does not exist
     * @throws ProductNotFoundException if any product in the order does not exist
     * @throws InsufficientStockException if any product has insufficient stock
     */
    public OrderId createOrder(CreateOrderRequest request) {
        // Implementation
    }
    
    /**
     * Finds an order by its unique identifier.
     * 
     * @param orderId the order ID to search for
     * @return the order DTO if found
     * @throws OrderNotFoundException if no order exists with the given ID
     */
    public OrderDTO findById(OrderId orderId) {
        // Implementation
    }
    
    /**
     * Updates the status of an existing order.
     * 
     * <p>Valid state transitions:</p>
     * <ul>
     *   <li>CREATED → CONFIRMED</li>
     *   <li>CREATED → CANCELLED</li>
     *   <li>CONFIRMED → SHIPPED</li>
     *   <li>SHIPPED → DELIVERED</li>
     * </ul>
     * 
     * @param orderId the order ID to update
     * @param status the new status
     * @return the updated order DTO
     * @throws OrderNotFoundException if no order exists with the given ID
     * @throws IllegalOrderStateException if the status transition is not allowed
     */
    public OrderDTO updateStatus(OrderId orderId, OrderStatus status) {
        // Implementation
    }
}
```

### README Template

```markdown
# Order Service

## Overview

Order Service is a microservice responsible for managing orders in the e-commerce platform. 
It implements Domain-Driven Design (DDD) and CQRS patterns using Axon Framework.

## Architecture

### Technology Stack
- Java 17
- Spring Boot 3.2
- Axon Framework 4.9
- PostgreSQL 15
- Redis 7

### Key Components
- **Order Aggregate**: Manages order lifecycle and business rules
- **Order Projection**: Provides read-optimized order views
- **Order Saga**: Orchestrates order processing workflow
- **Event Handlers**: Handle side effects and notifications

## Getting Started

### Prerequisites
- JDK 17+
- Maven 3.8+
- Docker and Docker Compose
- PostgreSQL 15+
- Redis 7+

### Installation

1. Clone the repository:
```bash
git clone https://github.com/example/order-service.git
cd order-service
```

2. Start infrastructure:
```bash
docker-compose up -d
```

3. Build and run:
```bash
mvn clean install
mvn spring-boot:run
```

### Configuration

Application configuration is managed through `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orderdb
    username: order_user
    password: order_pass
    
axon:
  axonserver:
    servers: localhost:8124
```

## API Documentation

Access Swagger UI at: http://localhost:8080/swagger-ui.html

### Key Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/orders | Create a new order |
| GET | /api/orders/{id} | Get order by ID |
| PATCH | /api/orders/{id}/status | Update order status |
| GET | /api/orders | List orders with filters |

## Development

### Running Tests
```bash
mvn test
```

### Code Style
This project follows Google Java Style Guide. Run formatter:
```bash
mvn spotless:apply
```

### Building Docker Image
```bash
docker build -t order-service:latest .
```

## Monitoring

### Health Checks
- Application: http://localhost:8080/actuator/health
- Database: http://localhost:8080/actuator/health/db
- Redis: http://localhost:8080/actuator/health/redis

### Metrics
Access Prometheus metrics at: http://localhost:8080/actuator/prometheus

### Logging
- Application logs: `/var/log/order-service/application.log`
- Error logs: `/var/log/order-service/error.log`
- JSON logs: `/var/log/order-service/json.log`

## Troubleshooting

### Common Issues

1. **Order not found**
   - Check if order ID is correct
   - Verify order exists in database
   - Check event store for order events

2. **Payment failed**
   - Verify payment service is running
   - Check payment credentials
   - Review payment logs

3. **Saga stuck**
   - Check saga state in saga store
   - Review saga event handlers
   - Check for deadlocks

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details.
```

## Best Practices

### 1. Logging
- Use appropriate log levels
- Include context in log messages
- Use structured logging for machine parsing
- Never log sensitive information (passwords, tokens, PII)
- Use MDC for request tracing

### 2. Documentation
- Keep documentation up-to-date
- Use meaningful descriptions in API docs
- Include examples in documentation
- Document error responses
- Version your API documentation

### 3. Code Comments
- Comment "why", not "what"
- Use Javadoc for public APIs
- Keep comments concise and relevant
- Update comments when code changes

### 4. Monitoring
- Implement health checks
- Expose metrics
- Set up alerting
- Monitor log aggregation

## References

- [SLF4J Documentation](https://www.slf4j.org/manual.html)
- [Logback Documentation](https://logback.qos.ch/documentation.html)
- [OpenAPI Specification](https://swagger.io/specification/)
- [SpringDoc OpenAPI](https://springdoc.org/)
