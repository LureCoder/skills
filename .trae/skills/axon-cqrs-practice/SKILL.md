---
name: "axon-cqrs-practice"
description: "Axon Framework CQRS and Event Sourcing best practices. Invoke when implementing CQRS patterns, event sourcing, sagas, or Axon-specific features."
---

# Axon Framework CQRS Best Practices

This skill provides comprehensive guidelines for implementing CQRS (Command Query Responsibility Segregation) and Event Sourcing using Axon Framework in Spring Boot applications.

## CQRS Fundamentals

### Command Side (Write Model)

Commands represent intentions to change state. They are imperative and expressed in present tense.

```java
package com.example.order.application.command;

import org.axonframework.modelling.command.TargetAggregateIdentifier;

public class CreateOrderCommand {
    
    @TargetAggregateIdentifier
    private final OrderId orderId;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    
    public CreateOrderCommand(OrderId orderId, CustomerId customerId, List<OrderItem> items) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
    }
    
    // Getters
}
```

### Query Side (Read Model)

Queries request information and do not change state. They are expressed as questions.

```java
package com.example.order.application.query;

public class FindOrderByIdQuery {
    
    private final OrderId orderId;
    
    public FindOrderByIdQuery(OrderId orderId) {
        this.orderId = orderId;
    }
    
    // Getters
}
```

## Aggregate Design

### Aggregate Root with Event Sourcing

```java
package com.example.order.domain.model;

import org.axonframework.commandhandling.CommandHandler;
import org.axonframework.eventsourcing.EventSourcingHandler;
import org.axonframework.modelling.command.AggregateIdentifier;
import org.axonframework.modelling.command.AggregateLifecycle;
import org.axonframework.spring.stereotype.Aggregate;

import static org.axonframework.modelling.command.AggregateLifecycle.apply;

@Aggregate
public class Order {
    
    @AggregateIdentifier
    private OrderId id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    protected Order() {
        // Required by Axon Framework
    }
    
    @CommandHandler
    public Order(CreateOrderCommand command) {
        validateCreateCommand(command);
        
        apply(new OrderCreatedEvent(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems(),
            LocalDateTime.now()
        ));
    }
    
    @CommandHandler
    public void handle(AddOrderItemCommand command) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Cannot add items to a confirmed order");
        }
        
        if (command.getQuantity().getValue() <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        
        apply(new OrderItemAddedEvent(
            this.id,
            command.getProduct(),
            command.getQuantity(),
            LocalDateTime.now()
        ));
    }
    
    @CommandHandler
    public void handle(ConfirmOrderCommand command) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("Only created orders can be confirmed");
        }
        
        if (this.items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm an empty order");
        }
        
        apply(new OrderConfirmedEvent(this.id, LocalDateTime.now()));
    }
    
    @CommandHandler
    public void handle(CancelOrderCommand command) {
        if (this.status == OrderStatus.SHIPPED || this.status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel shipped or delivered orders");
        }
        
        apply(new OrderCancelledEvent(this.id, command.getReason(), LocalDateTime.now()));
    }
    
    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.customerId = event.getCustomerId();
        this.items = new ArrayList<>(event.getItems());
        this.status = OrderStatus.CREATED;
        this.totalAmount = calculateTotal();
        this.createdAt = event.getCreatedAt();
        this.updatedAt = event.getCreatedAt();
    }
    
    @EventSourcingHandler
    public void on(OrderItemAddedEvent event) {
        OrderItem newItem = new OrderItem(
            OrderItemId.generate(),
            event.getProduct(),
            event.getQuantity()
        );
        this.items.add(newItem);
        this.totalAmount = calculateTotal();
        this.updatedAt = event.getAddedAt();
    }
    
    @EventSourcingHandler
    public void on(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
        this.updatedAt = event.getConfirmedAt();
    }
    
    @EventSourcingHandler
    public void on(OrderCancelledEvent event) {
        this.status = OrderStatus.CANCELLED;
        this.updatedAt = event.getCancelledAt();
    }
    
    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getPrice)
            .reduce(Money.ZERO, Money::add);
    }
    
    private void validateCreateCommand(CreateOrderCommand command) {
        if (command.getOrderId() == null) {
            throw new IllegalArgumentException("Order ID cannot be null");
        }
        if (command.getCustomerId() == null) {
            throw new IllegalArgumentException("Customer ID cannot be null");
        }
        if (command.getItems() == null || command.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }
    }
}
```

### Aggregate Configuration

```java
@Configuration
public class AxonConfiguration {
    
    @Bean
    public AggregateFactory<Order> orderAggregateFactory() {
        return new GenericAggregateFactory<>(Order.class);
    }
    
    @Bean
    public EventSourcingRepository<Order> orderRepository(
            EventStore eventStore,
            SnapshotTriggerDefinition snapshotTriggerDefinition) {
        return EventSourcingRepository.builder(Order.class)
            .eventStore(eventStore)
            .snapshotTriggerDefinition(snapshotTriggerDefinition)
            .build();
    }
    
    @Bean
    public SnapshotTriggerDefinition orderSnapshotTriggerDefinition(
            EventStore eventStore) {
        return new EventCountSnapshotTriggerDefinition(
            eventStore,
            100 // Snapshot every 100 events
        );
    }
}
```

## Command Handling

### Command Gateway

```java
@Service
public class OrderCommandService {
    
    private final CommandGateway commandGateway;
    
    public OrderCommandService(CommandGateway commandGateway) {
        this.commandGateway = commandGateway;
    }
    
    public CompletableFuture<OrderId> createOrder(CreateOrderRequest request) {
        OrderId orderId = OrderId.generate();
        
        CreateOrderCommand command = new CreateOrderCommand(
            orderId,
            request.getCustomerId(),
            request.getItems()
        );
        
        return commandGateway.send(command)
            .thenApply(result -> orderId);
    }
    
    public CompletableFuture<Void> confirmOrder(OrderId orderId) {
        return commandGateway.send(new ConfirmOrderCommand(orderId));
    }
    
    public CompletableFuture<Void> cancelOrder(OrderId orderId, String reason) {
        return commandGateway.send(new CancelOrderCommand(orderId, reason));
    }
}
```

### Command Dispatching

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderCommandService commandService;
    
    @PostMapping
    public CompletableFuture<ResponseEntity<OrderId>> createOrder(
            @RequestBody CreateOrderRequest request) {
        return commandService.createOrder(request)
            .thenApply(orderId -> ResponseEntity.ok(orderId));
    }
    
    @PostMapping("/{orderId}/confirm")
    public CompletableFuture<ResponseEntity<Void>> confirmOrder(
            @PathVariable String orderId) {
        return commandService.confirmOrder(OrderId.of(orderId))
            .thenApply(result -> ResponseEntity.ok().build());
    }
}
```

## Query Handling

### Query Model (Projection)

```java
package com.example.order.infrastructure.projection;

import org.axonframework.eventhandling.EventHandler;
import org.axonframework.queryhandling.QueryHandler;
import org.springframework.stereotype.Component;

@Component
public class OrderProjection {
    
    private final OrderReadRepository repository;
    
    public OrderProjection(OrderReadRepository repository) {
        this.repository = repository;
    }
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderSummary summary = new OrderSummary(
            event.getOrderId().getValue(),
            event.getCustomerId().getValue(),
            event.getItems().size(),
            calculateTotal(event.getItems()),
            "CREATED",
            event.getCreatedAt()
        );
        
        repository.save(summary);
    }
    
    @EventHandler
    public void on(OrderConfirmedEvent event) {
        repository.findById(event.getOrderId().getValue())
            .ifPresent(summary -> {
                summary.setStatus("CONFIRMED");
                summary.setUpdatedAt(event.getConfirmedAt());
                repository.save(summary);
            });
    }
    
    @QueryHandler
    public OrderSummary handle(FindOrderByIdQuery query) {
        return repository.findById(query.getOrderId().getValue())
            .orElseThrow(() -> new OrderNotFoundException(query.getOrderId()));
    }
    
    @QueryHandler
    public List<OrderSummary> handle(FindOrdersByCustomerQuery query) {
        return repository.findByCustomerId(query.getCustomerId().getValue());
    }
}
```

### Query Service

```java
@Service
public class OrderQueryService {
    
    private final QueryGateway queryGateway;
    
    public OrderQueryService(QueryGateway queryGateway) {
        this.queryGateway = queryGateway;
    }
    
    public CompletableFuture<OrderSummary> findById(OrderId orderId) {
        return queryGateway.query(
            new FindOrderByIdQuery(orderId),
            ResponseTypes.instanceOf(OrderSummary.class)
        );
    }
    
    public CompletableFuture<List<OrderSummary>> findByCustomer(CustomerId customerId) {
        return queryGateway.query(
            new FindOrdersByCustomerQuery(customerId),
            ResponseTypes.multipleInstancesOf(OrderSummary.class)
        );
    }
    
    public Flux<OrderSummary> subscribeToOrderUpdates(OrderId orderId) {
        return queryGateway.subscriptionQuery(
            new FindOrderByIdQuery(orderId),
            ResponseTypes.instanceOf(OrderSummary.class),
            ResponseTypes.instanceOf(OrderSummary.class)
        ).updates();
    }
}
```

## Event Sourcing

### Event Store Configuration

```java
@Configuration
public class EventStoreConfiguration {
    
    @Bean
    public EventStore eventStore(EventStorageEngine storageEngine) {
        return AxonServerEventStore.builder()
            .storageEngine(storageEngine)
            .build();
    }
    
    @Bean
    public EventStorageEngine eventStorageEngine(DataSource dataSource) {
        return JdbcEventStorageEngine.builder()
            .dataSource(dataSource)
            .transactionManager(DataSourceTransactionManager::new)
            .build();
    }
}
```

### Snapshot Configuration

```java
@Configuration
public class SnapshotConfiguration {
    
    @Bean
    public Snapshotter snapshotter(EventStore eventStore) {
        return AggregateSnapshotter.builder()
            .eventStore(eventStore)
            .aggregateFactories(Collections.singletonList(
                new GenericAggregateFactory<>(Order.class)
            ))
            .build();
    }
    
    @Bean
    public SnapshotTriggerDefinition orderSnapshotTriggerDefinition(
            Snapshotter snapshotter) {
        return new EventCountSnapshotTriggerDefinition(
            snapshotter,
            100
        );
    }
}
```

## Saga Pattern

### Saga Implementation

```java
@Saga
public class OrderManagementSaga {
    
    @Autowired
    private transient CommandGateway commandGateway;
    
    private OrderId orderId;
    private PaymentId paymentId;
    private ShipmentId shipmentId;
    private boolean paymentCompleted = false;
    private boolean shipmentCompleted = false;
    
    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        
        String paymentId = UUID.randomUUID().toString();
        this.paymentId = new PaymentId(paymentId);
        
        commandGateway.send(new CreatePaymentCommand(
            this.paymentId,
            event.getOrderId(),
            event.getCustomerId(),
            calculateTotalAmount(event.getItems())
        ));
    }
    
    @SagaEventHandler(associationProperty = "paymentId")
    public void handle(PaymentCompletedEvent event) {
        this.paymentCompleted = true;
        
        if (paymentCompleted && !shipmentCompleted) {
            String shipmentId = UUID.randomUUID().toString();
            this.shipmentId = new ShipmentId(shipmentId);
            
            commandGateway.send(new CreateShipmentCommand(
                this.shipmentId,
                this.orderId,
                getShippingAddress()
            ));
        }
    }
    
    @SagaEventHandler(associationProperty = "shipmentId")
    public void handle(ShipmentCompletedEvent event) {
        this.shipmentCompleted = true;
        
        if (paymentCompleted && shipmentCompleted) {
            commandGateway.send(new CompleteOrderCommand(this.orderId));
            SagaLifecycle.end();
        }
    }
    
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCancelledEvent event) {
        if (paymentId != null) {
            commandGateway.send(new CancelPaymentCommand(paymentId));
        }
        if (shipmentId != null) {
            commandGateway.send(new CancelShipmentCommand(shipmentId));
        }
        SagaLifecycle.end();
    }
    
    @EndSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void handle(OrderCompletedEvent event) {
        // Saga ends
    }
}
```

### Saga Configuration

```java
@Configuration
public class SagaConfiguration {
    
    @Bean
    public SagaStore<Object> sagaStore(DataSource dataSource) {
        return JdbcSagaStore.builder()
            .dataSource(dataSource)
            .sqlSchema(new HsqlSagaStoreSchema())
            .build();
    }
    
    @Bean
    public SagaManager<OrderManagementSaga> orderManagementSagaManager(
            SagaStore<Object> sagaStore) {
        return AnnotatedSagaManager.builder()
            .sagaStore(sagaStore)
            .sagaType(OrderManagementSaga.class)
            .build();
    }
}
```

## Event Handling

### Event Handlers

```java
@Component
public class OrderEventHandler {
    
    private final NotificationService notificationService;
    private final InventoryService inventoryService;
    private final OrderReadRepository readRepository;
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        notificationService.sendOrderCreatedNotification(
            event.getCustomerId(),
            event.getOrderId()
        );
        
        for (OrderItem item : event.getItems()) {
            inventoryService.reserveStock(
                item.getProductId(),
                item.getQuantity()
            );
        }
    }
    
    @EventHandler
    public void on(OrderConfirmedEvent event) {
        notificationService.sendOrderConfirmedNotification(event.getOrderId());
        
        readRepository.findById(event.getOrderId().getValue())
            .ifPresent(order -> {
                order.setStatus("CONFIRMED");
                readRepository.save(order);
            });
    }
    
    @EventHandler
    public void on(OrderCancelledEvent event) {
        notificationService.sendOrderCancelledNotification(event.getOrderId());
        
        readRepository.findById(event.getOrderId().getValue())
            .ifPresent(order -> {
                for (OrderItem item : order.getItems()) {
                    inventoryService.releaseStock(
                        item.getProductId(),
                        item.getQuantity()
                    );
                }
            });
    }
}
```

### Event Processing Configuration

```java
@Configuration
public class EventProcessingConfiguration {
    
    @Bean
    public EventProcessingConfigurer eventProcessingConfigurer(
            EventProcessingConfigurer configurer) {
        
        configurer.registerEventHandlerConfiguration("order-group",
            SubscribingEventProcessorConfiguration.forName("order-group")
                .andBatchSize(10)
        );
        
        configurer.registerEventHandlerConfiguration("notification-group",
            TrackingEventProcessorConfiguration.forName("notification-group")
                .andInitialSegmentsCount(4)
                .andBatchSize(100)
        );
        
        return configurer;
    }
    
    @Bean
    public SequencingPolicy<EventMessage<?>> orderSequencingPolicy() {
        return event -> {
            if (event.getPayload() instanceof OrderEvent) {
                return ((OrderEvent) event.getPayload()).getOrderId();
            }
            return event.getIdentifier();
        };
    }
}
```

## Best Practices

### 1. Command Design
- Commands should be immutable
- Use meaningful command names (CreateOrder, not OrderCreate)
- Include all necessary data in the command
- Validate commands in the command handler

### 2. Event Design
- Events should be immutable
- Use past tense for event names (OrderCreated, not CreateOrder)
- Include all relevant data for event handlers
- Events should be serializable

### 3. Aggregate Design
- Keep aggregates small and focused
- Protect invariants within aggregate boundaries
- Use events to communicate between aggregates
- Implement proper validation in command handlers

### 4. Saga Design
- Keep sagas simple and focused
- Handle compensating actions for failures
- Use meaningful association properties
- End sagas when the process completes

### 5. Performance Considerations
- Use snapshots for large event streams
- Configure appropriate batch sizes for event processors
- Use tracking event processors for scalability
- Monitor event store performance

### 6. Testing
- Test aggregate behavior with commands and events
- Test saga orchestration
- Test event handlers independently
- Use Axon Test Fixtures

## Testing with Axon

### Aggregate Test Fixture

```java
public class OrderTest {
    
    private AggregateTestFixture<Order> fixture;
    
    @BeforeEach
    void setUp() {
        fixture = new AggregateTestFixture<>(Order.class);
    }
    
    @Test
    void shouldCreateOrder() {
        OrderId orderId = OrderId.generate();
        CustomerId customerId = CustomerId.generate();
        List<OrderItem> items = Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(2))
        );
        
        fixture.givenNoPriorActivity()
            .when(new CreateOrderCommand(orderId, customerId, items))
            .expectEvents(new OrderCreatedEvent(orderId, customerId, items, any(LocalDateTime.class)))
            .expectSuccessfulHandlerExecution();
    }
    
    @Test
    void shouldNotConfirmEmptyOrder() {
        OrderId orderId = OrderId.generate();
        
        fixture.given(new OrderCreatedEvent(orderId, CustomerId.generate(), Collections.emptyList(), LocalDateTime.now()))
            .when(new ConfirmOrderCommand(orderId))
            .expectException(IllegalStateException.class)
            .expectExceptionMessage("Cannot confirm an empty order");
    }
}
```

### Saga Test Fixture

```java
public class OrderManagementSagaTest {
    
    private SagaTestFixture<OrderManagementSaga> fixture;
    
    @BeforeEach
    void setUp() {
        fixture = new SagaTestFixture<>(OrderManagementSaga.class);
    }
    
    @Test
    void shouldCreatePaymentOnOrderCreated() {
        OrderId orderId = OrderId.generate();
        CustomerId customerId = CustomerId.generate();
        
        fixture.givenNoPriorActivity()
            .whenPublishingA(new OrderCreatedEvent(orderId, customerId, Arrays.asList(
                new OrderItem(ProductId.generate(), Quantity.of(1))
            )))
            .expectActiveSagas(1)
            .expectDispatchedCommandsMatching(
                payloadsMatching(exactMatch(any(CreatePaymentCommand.class)))
            );
    }
}
```

## References

- [Axon Framework Reference Guide](https://docs.axoniq.io/reference-guide/)
- [Axon Best Practices](https://docs.axoniq.io/reference-guide/axon-framework/platform-integration/best-practices)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
