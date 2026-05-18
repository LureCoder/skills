---
name: "java-code-style"
description: "Java code style and best practices guide. Invoke when writing Java code, reviewing code quality, or enforcing coding standards."
---

# Java Code Style and Best Practices

This skill provides comprehensive guidelines for writing clean, maintainable, and professional Java code in Spring Boot applications.

## Naming Conventions

### Class Names
- Use PascalCase for class names
- Classes should be nouns representing their purpose
- Avoid generic names like `Manager`, `Util`, `Helper`

```java
// Good
public class OrderAggregate { }
public class PaymentService { }
public class CustomerRepository { }

// Bad
public class OrderManager { }
public class PaymentUtil { }
public class Helper { }
```

### Method Names
- Use camelCase for method names
- Methods should start with a verb
- Be descriptive and specific

```java
// Good
public void processPayment(PaymentCommand command) { }
public Optional<Order> findOrderById(OrderId orderId) { }
public boolean isPaymentCompleted() { }

// Bad
public void payment() { }
public Order order() { }
public boolean check() { }
```

### Variable Names
- Use camelCase for variables
- Use meaningful names that reveal intent
- Avoid single-letter names except for loop counters

```java
// Good
List<OrderItem> orderItems = order.getItems();
Money totalAmount = calculateTotal();
int retryCount = 0;

// Bad
List<OrderItem> list = order.getItems();
Money m = calculateTotal();
int x = 0;
```

### Constants
- Use SCREAMING_SNAKE_CASE for constants
- Use `static final` for class-level constants
- Group related constants in dedicated classes

```java
// Good
public static final String DEFAULT_CURRENCY = "USD";
public static final int MAX_RETRY_ATTEMPTS = 3;
public static final long CACHE_EXPIRATION_SECONDS = 3600L;

// Better - dedicated constant class
public final class OrderStatus {
    public static final String CREATED = "CREATED";
    public static final String CONFIRMED = "CONFIRMED";
    public static final String SHIPPED = "SHIPPED";
    public static final String DELIVERED = "DELIVERED";
    
    private OrderStatus() { }
}
```

## Avoid Hardcoding

### Use Configuration Properties
Instead of hardcoding values, use Spring's `@ConfigurationProperties`:

```java
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties {
    private int timeoutSeconds = 30;
    private int maxRetryAttempts = 3;
    private String defaultCurrency = "USD";
    
    // Getters and setters
}
```

application.yml:
```yaml
app:
  payment:
    timeout-seconds: 30
    max-retry-attempts: 3
    default-currency: USD
```

### Use Enums for Fixed Values
```java
public enum OrderStatus {
    CREATED,
    CONFIRMED,
    SHIPPED,
    DELIVERED,
    CANCELLED
}

public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    PAYPAL,
    BANK_TRANSFER
}
```

### Use Message Keys for User Messages
```java
@Service
public class NotificationService {
    
    @Value("${notification.order.created.message}")
    private String orderCreatedMessage;
    
    public void notifyOrderCreated(OrderId orderId) {
        String message = String.format(orderCreatedMessage, orderId.getValue());
        sendNotification(message);
    }
}
```

## Code Readability

### Method Length
- Keep methods small and focused (max 20-30 lines)
- Each method should do one thing well
- Extract complex logic into separate methods

```java
// Good - small, focused method
public Order createOrder(CreateOrderCommand command) {
    validateCommand(command);
    Order order = buildOrder(command);
    order.validate();
    return orderRepository.save(order);
}

// Bad - long, complex method
public Order createOrder(CreateOrderCommand command) {
    // 100 lines of complex logic...
}
```

### Parameter Count
- Limit parameters to 3-4 maximum
- Use Parameter Object pattern for more parameters

```java
// Good - use parameter object
public class CreateOrderCommand {
    private CustomerId customerId;
    private List<OrderItem> items;
    private ShippingAddress shippingAddress;
    private PaymentMethod paymentMethod;
}

public Order createOrder(CreateOrderCommand command) { }

// Bad - too many parameters
public Order createOrder(CustomerId customerId, 
                         List<OrderItem> items,
                         ShippingAddress address,
                         PaymentMethod method,
                         String couponCode,
                         boolean giftWrap) { }
```

### Use Meaningful Expressions
```java
// Good - clear intent
if (order.isEligibleForDiscount()) {
    order.applyDiscount(discountRate);
}

// Bad - unclear logic
if (order.getTotalAmount() > 100 && order.getItemCount() > 5 
    && customer.getMembershipLevel() > 2) {
    order.setTotalAmount(order.getTotalAmount() * 0.9);
}
```

## Code Formatting

### Indentation and Braces
- Use 4 spaces for indentation (or 2 spaces per team convention)
- Always use braces, even for single-line statements
- Opening brace on same line, closing brace on new line

```java
// Good
if (order.isValid()) {
    processOrder(order);
} else {
    log.warn("Invalid order: {}", order.getId());
}

// Bad
if (order.isValid())
    processOrder(order);
```

### Blank Lines
- Use blank lines to separate logical sections
- One blank line between methods
- Two blank lines between classes/interfaces

```java
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    
    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
    
    public Order createOrder(CreateOrderCommand command) {
        // Implementation
    }
    
    public Optional<Order> findOrderById(OrderId orderId) {
        // Implementation
    }
}
```

### Import Organization
- Organize imports alphabetically
- Separate groups with blank lines:
  1. Java standard library
  2. Third-party libraries
  3. Framework imports (Spring, Axon)
  4. Project-specific imports

```java
import java.util.List;
import java.util.Optional;

import com.fasterxml.jackson.databind.ObjectMapper;

import org.axonframework.commandhandling.CommandHandler;
import org.springframework.stereotype.Service;

import com.example.order.domain.Order;
import com.example.order.domain.OrderId;
```

## Comments and Documentation

### When to Comment
- Comment "why", not "what"
- Use Javadoc for public APIs
- Avoid redundant comments

```java
// Good - explains why
// Using pessimistic locking to prevent concurrent order modifications
@Transactional
public Order updateOrder(OrderId orderId, UpdateOrderCommand command) {
    // Implementation
}

// Bad - explains what (obvious from code)
// Get order by ID
Order order = orderRepository.findById(orderId);
```

### Javadoc Standards
```java
/**
 * Creates a new order for the specified customer.
 *
 * @param command the create order command containing order details
 * @return the created order with generated order ID
 * @throws IllegalArgumentException if command is null or invalid
 * @throws CustomerNotFoundException if customer does not exist
 */
public Order createOrder(CreateOrderCommand command) {
    // Implementation
}
```

## Code Smells to Avoid

### Magic Numbers
```java
// Bad
if (order.getTotalAmount() > 1000) {
    order.applyDiscount(0.1);
}

// Good
private static final BigDecimal VIP_THRESHOLD = BigDecimal.valueOf(1000);
private static final BigDecimal VIP_DISCOUNT_RATE = BigDecimal.valueOf(0.1);

if (order.getTotalAmount().compareTo(VIP_THRESHOLD) > 0) {
    order.applyDiscount(VIP_DISCOUNT_RATE);
}
```

### Deep Nesting
```java
// Bad - deep nesting
public void processOrder(Order order) {
    if (order != null) {
        if (order.isValid()) {
            if (order.hasItems()) {
                if (customer.isActive()) {
                    // Process order
                }
            }
        }
    }
}

// Good - early returns
public void processOrder(Order order) {
    if (order == null) {
        return;
    }
    
    if (!order.isValid()) {
        log.warn("Invalid order: {}", order.getId());
        return;
    }
    
    if (!order.hasItems()) {
        log.warn("Order has no items: {}", order.getId());
        return;
    }
    
    if (!customer.isActive()) {
        log.warn("Customer is not active: {}", customer.getId());
        return;
    }
    
    // Process order
}
```

### God Classes
- Avoid classes with too many responsibilities
- Follow Single Responsibility Principle
- Keep classes focused and cohesive

## Best Practices Summary

1. **Use meaningful names** - Code should read like well-written prose
2. **Avoid hardcoding** - Use configuration, constants, and enums
3. **Keep methods small** - Each method should do one thing well
4. **Limit parameters** - Use parameter objects for complex operations
5. **Comment wisely** - Explain why, not what
6. **Format consistently** - Follow team conventions
7. **Avoid code smells** - Refactor when you see issues
8. **Write self-documenting code** - Clear names and structure reduce need for comments

## Tools and Automation

### Recommended Tools
- **Checkstyle**: Enforce coding standards
- **SpotBugs**: Find bugs and code smells
- **PMD**: Detect suboptimal code
- **SonarQube**: Comprehensive code quality analysis

### IDE Configuration
- Configure auto-formatting on save
- Enable import optimization
- Set up code inspection rules
- Use code templates for common patterns

## References

- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Effective Java by Joshua Bloch](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Clean Code by Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)
