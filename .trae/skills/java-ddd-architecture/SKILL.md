---
name: "java-ddd-architecture"
description: "Domain-Driven Design architecture and package structure guide. Invoke when designing domain models, organizing packages, or implementing DDD patterns."
---

# Domain-Driven Design Architecture and Package Structure

This skill provides comprehensive guidelines for implementing Domain-Driven Design (DDD) in Java Spring Boot applications with proper package structure and architectural patterns.

## Package Structure

### Recommended Package Organization

Follow a modular, domain-centric package structure:

```
com.example.order-service/
├── order/                          # Order Bounded Context
│   ├── domain/                     # Domain Layer
│   │   ├── model/                  # Domain Models
│   │   │   ├── Order.java          # Aggregate Root
│   │   │   ├── OrderId.java        # Value Object
│   │   │   ├── OrderItem.java      # Entity
│   │   │   ├── OrderStatus.java    # Enum
│   │   │   └── Money.java          # Value Object
│   │   ├── event/                  # Domain Events
│   │   │   ├── OrderCreatedEvent.java
│   │   │   ├── OrderConfirmedEvent.java
│   │   │   └── OrderShippedEvent.java
│   │   ├── service/                # Domain Services
│   │   │   ├── OrderValidationService.java
│   │   │   └── PricingService.java
│   │   └── repository/             # Repository Interfaces
│   │       └── OrderRepository.java
│   ├── application/                # Application Layer
│   │   ├── command/                # Commands
│   │   │   ├── CreateOrderCommand.java
│   │   │   └── ConfirmOrderCommand.java
│   │   ├── query/                  # Queries
│   │   │   ├── FindOrderQuery.java
│   │   │   └── OrderQueryService.java
│   │   ├── service/                # Application Services
│   │   │   └── OrderApplicationService.java
│   │   └── dto/                    # Data Transfer Objects
│   │       ├── OrderDTO.java
│   │       └── OrderItemDTO.java
│   ├── infrastructure/             # Infrastructure Layer
│   │   ├── persistence/            # Persistence Implementation
│   │   │   ├── JpaOrderRepository.java
│   │   │   └── OrderEntity.java
│   │   ├── messaging/              # Messaging Implementation
│   │   │   └── OrderEventPublisher.java
│   │   ├── client/                 # External Service Clients
│   │   │   └── InventoryServiceClient.java
│   │   └── config/                 # Configuration
│   │       └── OrderConfiguration.java
│   └── interfaces/                 # Interface Layer (API)
│       ├── rest/                   # REST Controllers
│       │   ├── OrderController.java
│       │   └── OrderDTOAssembler.java
│       └── facade/                 # Facade for External Access
│           └── OrderServiceFacade.java
├── shared/                         # Shared Kernel
│   ├── domain/                     # Shared Domain Concepts
│   │   ├── AggregateRoot.java
│   │   ├── DomainEvent.java
│   │   └── Identity.java
│   ├── infrastructure/             # Shared Infrastructure
│   │   ├── event/                  # Event Publishing
│   │   └── persistence/            # Base Repositories
│   └── kernel/                     # Common Utilities
│       ├── Money.java
│       └── Quantity.java
└── Application.java                # Application Entry Point
```

## Layered Architecture

### Domain Layer
The heart of the application containing business logic and rules.

```java
package com.example.order.domain.model;

import org.axonframework.modelling.command.AggregateIdentifier;
import org.axonframework.modelling.command.AggregateLifecycle;
import org.axonframework.spring.stereotype.Aggregate;

@Aggregate
public class Order {
    
    @AggregateIdentifier
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private LocalDateTime createdAt;
    
    protected Order() {
        // Required by Axon
    }
    
    @CommandHandler
    public Order(CreateOrderCommand command) {
        validateCommand(command);
        
        apply(new OrderCreatedEvent(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems(),
            LocalDateTime.now()
        ));
    }
    
    @CommandHandler
    public void handle(ConfirmOrderCommand command) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Only created orders can be confirmed");
        }
        
        apply(new OrderConfirmedEvent(this.id, LocalDateTime.now()));
    }
    
    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.customerId = event.getCustomerId();
        this.items = event.getItems();
        this.status = OrderStatus.CREATED;
        this.totalAmount = calculateTotal();
        this.createdAt = event.getCreatedAt();
    }
    
    @EventSourcingHandler
    public void on(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
    }
    
    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getPrice)
            .reduce(Money.ZERO, Money::add);
    }
    
    private void validateCommand(CreateOrderCommand command) {
        if (command.getItems() == null || command.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
    }
    
    private void apply(Object event) {
        AggregateLifecycle.apply(event);
    }
}
```

### Application Layer
Orchestrates use cases and coordinates domain objects.

```java
package com.example.order.application.service;

import org.axonframework.commandhandling.gateway.CommandGateway;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderApplicationService {
    
    private final CommandGateway commandGateway;
    private final OrderQueryService queryService;
    
    public OrderApplicationService(CommandGateway commandGateway,
                                    OrderQueryService queryService) {
        this.commandGateway = commandGateway;
        this.queryService = queryService;
    }
    
    public OrderId createOrder(CreateOrderRequest request) {
        OrderId orderId = OrderId.generate();
        
        CreateOrderCommand command = new CreateOrderCommand(
            orderId,
            request.getCustomerId(),
            request.getItems()
        );
        
        commandGateway.sendAndWait(command);
        return orderId;
    }
    
    @Transactional
    public void confirmOrder(OrderId orderId) {
        Order order = queryService.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        if (!order.canBeConfirmed()) {
            throw new OrderCannotBeConfirmedException(orderId);
        }
        
        commandGateway.sendAndWait(new ConfirmOrderCommand(orderId));
    }
}
```

### Infrastructure Layer
Provides technical capabilities and external integrations.

```java
package com.example.order.infrastructure.persistence;

import org.springframework.stereotype.Component;

@Component
public class JpaOrderRepository implements OrderRepository {
    
    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;
    
    public JpaOrderRepository(OrderJpaRepository jpaRepository,
                              OrderMapper mapper) {
        this.jpaRepository = jpaRepository;
        this.mapper = mapper;
    }
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        OrderEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @Override
    public Optional<Order> findById(OrderId orderId) {
        return jpaRepository.findById(orderId.getValue())
            .map(mapper::toDomain);
    }
}
```

### Interface Layer
Exposes application capabilities to external world.

```java
package com.example.order.interfaces.rest;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderApplicationService orderService;
    private final OrderDTOAssembler assembler;
    
    public OrderController(OrderApplicationService orderService,
                           OrderDTOAssembler assembler) {
        this.orderService = orderService;
        this.assembler = assembler;
    }
    
    @PostMapping
    public ResponseEntity<OrderDTO> createOrder(@RequestBody CreateOrderRequest request) {
        OrderId orderId = orderService.createOrder(request);
        OrderDTO order = orderService.findOrderById(orderId);
        return ResponseEntity.ok(order);
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable String orderId) {
        OrderDTO order = orderService.findOrderById(OrderId.of(orderId));
        return ResponseEntity.ok(order);
    }
}
```

## Domain Model Patterns

### Aggregate Root
```java
@Aggregate
public class Order {
    
    @AggregateIdentifier
    private OrderId id;
    
    private List<OrderItem> items;
    private OrderStatus status;
    
    public void addItem(Product product, Quantity quantity) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Cannot add items to confirmed order");
        }
        
        OrderItem item = new OrderItem(
            OrderItemId.generate(),
            product,
            quantity
        );
        
        this.items.add(item);
        apply(new OrderItemAddedEvent(this.id, item));
    }
    
    public void removeItem(OrderItemId itemId) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Cannot remove items from confirmed order");
        }
        
        boolean removed = this.items.removeIf(item -> item.getId().equals(itemId));
        if (!removed) {
            throw new OrderItemNotFoundException(itemId);
        }
        
        apply(new OrderItemRemovedEvent(this.id, itemId));
    }
}
```

### Entity
```java
public class OrderItem {
    
    private OrderItemId id;
    private Product product;
    private Quantity quantity;
    private Money price;
    
    public OrderItem(OrderItemId id, Product product, Quantity quantity) {
        this.id = Objects.requireNonNull(id);
        this.product = Objects.requireNonNull(product);
        this.quantity = Objects.requireNonNull(quantity);
        this.price = product.getPrice().multiply(quantity);
    }
    
    public void increaseQuantity(Quantity additional) {
        this.quantity = this.quantity.add(additional);
        this.price = this.product.getPrice().multiply(this.quantity);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        OrderItem that = (OrderItem) o;
        return Objects.equals(id, that.id);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

### Value Object
```java
public class Money implements ValueObject {
    
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
        
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Money cannot be negative");
        }
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add money with different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public Money multiply(Quantity quantity) {
        return new Money(
            this.amount.multiply(BigDecimal.valueOf(quantity.getValue())),
            this.currency
        );
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.equals(money.amount) && currency.equals(money.currency);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

### Domain Service
```java
@Service
public class PricingService {
    
    private final DiscountPolicy discountPolicy;
    private final TaxCalculator taxCalculator;
    
    public PricingService(DiscountPolicy discountPolicy, TaxCalculator taxCalculator) {
        this.discountPolicy = discountPolicy;
        this.taxCalculator = taxCalculator;
    }
    
    public Money calculateFinalPrice(Order order, Customer customer) {
        Money basePrice = order.calculateTotal();
        Money discount = discountPolicy.calculateDiscount(order, customer);
        Money discountedPrice = basePrice.subtract(discount);
        Money tax = taxCalculator.calculateTax(discountedPrice, customer.getCountry());
        
        return discountedPrice.add(tax);
    }
}
```

## Domain Events

### Event Definition
```java
package com.example.order.domain.event;

import java.time.LocalDateTime;

public class OrderCreatedEvent {
    
    private final OrderId orderId;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private final LocalDateTime createdAt;
    
    public OrderCreatedEvent(OrderId orderId, CustomerId customerId, 
                             List<OrderItem> items, LocalDateTime createdAt) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.createdAt = createdAt;
    }
    
    // Getters
}
```

### Event Handler
```java
@Component
public class OrderEventHandler {
    
    private final NotificationService notificationService;
    private final InventoryService inventoryService;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        notificationService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId());
        
        for (OrderItem item : event.getItems()) {
            inventoryService.reserveStock(item.getProductId(), item.getQuantity());
        }
    }
    
    @EventHandler
    public void on(OrderConfirmedEvent event) {
        notificationService.sendOrderShippedNotification(event.getOrderId());
    }
}
```

## Repository Pattern

### Repository Interface (Domain Layer)
```java
package com.example.order.domain.repository;

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId orderId);
    List<Order> findByCustomerId(CustomerId customerId);
    void delete(Order order);
}
```

### Repository Implementation (Infrastructure Layer)
```java
package com.example.order.infrastructure.persistence;

@Repository
public class OrderJpaRepository implements OrderRepository {
    
    private final SpringDataOrderRepository repository;
    
    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderMapper.toEntity(order);
        OrderEntity saved = repository.save(entity);
        return OrderMapper.toDomain(saved);
    }
    
    @Override
    public Optional<Order> findById(OrderId orderId) {
        return repository.findById(orderId.getValue())
            .map(OrderMapper::toDomain);
    }
}
```

## Bounded Contexts

### Context Mapping
```
┌─────────────────┐         ┌──────────────────┐
│  Order Context  │         │ Inventory Context│
│                 │         │                  │
│  - Order        │<───────>│  - Product       │
│  - OrderItem    │         │  - Stock         │
│  - Payment      │         │  - Reservation   │
└─────────────────┘         └──────────────────┘
         │                           │
         │                           │
         v                           v
┌─────────────────┐         ┌──────────────────┐
│Shipping Context │         │ Customer Context │
│                 │         │                  │
│  - Shipment     │         │  - Customer      │
│  - Delivery     │         │  - Address       │
│  - Tracking     │         │  - Preferences   │
└─────────────────┘         └──────────────────┘
```

### Anti-Corruption Layer
```java
@Service
public class InventoryServiceAdapter {
    
    private final InventoryServiceClient client;
    
    public Product getProduct(ProductId productId) {
        InventoryProductDTO dto = client.getProduct(productId.getValue());
        
        return new Product(
            ProductId.of(dto.getId()),
            dto.getName(),
            new Money(dto.getPrice(), Currency.getInstance(dto.getCurrency())),
            Quantity.of(dto.getAvailableStock())
        );
    }
}
```

## Best Practices

### 1. Keep Domain Model Pure
- Domain layer should not depend on infrastructure
- Use interfaces for repositories
- Avoid framework annotations in domain model (except Axon annotations)

### 2. Use Value Objects Extensively
- Replace primitive types with value objects
- Value objects should be immutable
- Use value objects for IDs, Money, Quantities, etc.

### 3. Protect Invariants
- Aggregate roots should protect their invariants
- Validate all changes through aggregate methods
- Throw meaningful domain exceptions

### 4. Domain Events for Side Effects
- Use events to communicate between aggregates
- Keep event handlers idempotent
- Design events to be meaningful to the domain

### 5. Test Domain Logic Thoroughly
- Write unit tests for domain logic
- Test aggregate invariants
- Test business rules and constraints

## References

- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
- [Implementing Domain-Driven Design by Vaughn Vernon](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [Axon Framework Reference Guide](https://docs.axoniq.io/reference-guide/)
