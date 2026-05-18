---
name: "java-testing-strategy"
description: "Java testing strategies and best practices. Invoke when writing tests, setting up test infrastructure, or improving test coverage."
---

# Java Testing Strategy and Best Practices

This skill provides comprehensive guidelines for implementing effective testing strategies in Java Spring Boot applications with DDD, CQRS, and Axon Framework.

## Testing Pyramid

### Test Categories

```
        /\
       /  \      E2E Tests
      /----\     (Few, Slow, Expensive)
     /      \
    /--------\   Integration Tests
   /          \  (Some, Medium Speed)
  /            \
 /--------------\ Unit Tests
/                \ (Many, Fast, Cheap)
```

## Unit Testing

### Domain Model Testing

Test domain logic in isolation without dependencies.

```java
package com.example.order.domain.model;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import static org.junit.jupiter.api.Assertions.*;

class OrderTest {
    
    private OrderId orderId;
    private CustomerId customerId;
    private List<OrderItem> items;
    
    @BeforeEach
    void setUp() {
        orderId = OrderId.generate();
        customerId = CustomerId.generate();
        items = Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(2), Money.of("100.00"))
        );
    }
    
    @Test
    @DisplayName("Should create order with valid data")
    void shouldCreateOrderWithValidData() {
        Order order = new Order(orderId, customerId, items);
        
        assertNotNull(order.getId());
        assertEquals(customerId, order.getCustomerId());
        assertEquals(OrderStatus.CREATED, order.getStatus());
        assertFalse(order.getItems().isEmpty());
    }
    
    @Test
    @DisplayName("Should throw exception when creating order with null items")
    void shouldThrowExceptionWhenCreatingOrderWithNullItems() {
        assertThrows(IllegalArgumentException.class, () -> {
            new Order(orderId, customerId, null);
        });
    }
    
    @Test
    @DisplayName("Should throw exception when creating order with empty items")
    void shouldThrowExceptionWhenCreatingOrderWithEmptyItems() {
        assertThrows(IllegalArgumentException.class, () -> {
            new Order(orderId, customerId, Collections.emptyList());
        });
    }
    
    @Test
    @DisplayName("Should calculate total amount correctly")
    void shouldCalculateTotalAmountCorrectly() {
        List<OrderItem> items = Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(2), Money.of("100.00")),
            new OrderItem(ProductId.generate(), Quantity.of(3), Money.of("50.00"))
        );
        
        Order order = new Order(orderId, customerId, items);
        
        Money expectedTotal = Money.of("350.00"); // (2 * 100) + (3 * 50)
        assertEquals(expectedTotal, order.getTotalAmount());
    }
    
    @Test
    @DisplayName("Should confirm created order")
    void shouldConfirmCreatedOrder() {
        Order order = new Order(orderId, customerId, items);
        
        order.confirm();
        
        assertEquals(OrderStatus.CONFIRMED, order.getStatus());
    }
    
    @Test
    @DisplayName("Should not confirm already confirmed order")
    void shouldNotConfirmAlreadyConfirmedOrder() {
        Order order = new Order(orderId, customerId, items);
        order.confirm();
        
        assertThrows(IllegalStateException.class, order::confirm);
    }
    
    @ParameterizedTest
    @ValueSource(strings = {"SHIPPED", "DELIVERED", "CANCELLED"})
    @DisplayName("Should not confirm order in invalid status")
    void shouldNotConfirmOrderInInvalidStatus(String status) {
        Order order = new Order(orderId, customerId, items);
        order.setStatus(OrderStatus.valueOf(status));
        
        assertThrows(IllegalStateException.class, order::confirm);
    }
}
```

### Value Object Testing

```java
class MoneyTest {
    
    @Test
    @DisplayName("Should add money with same currency")
    void shouldAddMoneyWithSameCurrency() {
        Money money1 = Money.of("100.00", "USD");
        Money money2 = Money.of("50.00", "USD");
        
        Money result = money1.add(money2);
        
        assertEquals(Money.of("150.00", "USD"), result);
    }
    
    @Test
    @DisplayName("Should throw exception when adding money with different currencies")
    void shouldThrowExceptionWhenAddingMoneyWithDifferentCurrencies() {
        Money usd = Money.of("100.00", "USD");
        Money eur = Money.of("100.00", "EUR");
        
        assertThrows(IllegalArgumentException.class, () -> {
            usd.add(eur);
        });
    }
    
    @Test
    @DisplayName("Should be equal when same amount and currency")
    void shouldBeEqualWhenSameAmountAndCurrency() {
        Money money1 = Money.of("100.00", "USD");
        Money money2 = Money.of("100.00", "USD");
        
        assertEquals(money1, money2);
        assertEquals(money1.hashCode(), money2.hashCode());
    }
    
    @Test
    @DisplayName("Should not be equal when different amount or currency")
    void shouldNotBeEqualWhenDifferentAmountOrCurrency() {
        Money money1 = Money.of("100.00", "USD");
        Money money2 = Money.of("200.00", "USD");
        Money money3 = Money.of("100.00", "EUR");
        
        assertNotEquals(money1, money2);
        assertNotEquals(money1, money3);
    }
}
```

### Service Testing with Mocks

```java
@ExtendWith(MockitoExtension.class)
class OrderApplicationServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private PaymentService paymentService;
    
    @Mock
    private NotificationService notificationService;
    
    @InjectMocks
    private OrderApplicationService orderService;
    
    private OrderId orderId;
    private CustomerId customerId;
    
    @BeforeEach
    void setUp() {
        orderId = OrderId.generate();
        customerId = CustomerId.generate();
    }
    
    @Test
    @DisplayName("Should create order successfully")
    void shouldCreateOrderSuccessfully() {
        CreateOrderRequest request = new CreateOrderRequest(
            customerId,
            Arrays.asList(
                new OrderItemRequest(ProductId.generate(), Quantity.of(2))
            )
        );
        
        when(orderRepository.save(any(Order.class))).thenAnswer(invocation -> {
            Order order = invocation.getArgument(0);
            return order;
        });
        
        OrderId result = orderService.createOrder(request);
        
        assertNotNull(result);
        verify(orderRepository).save(any(Order.class));
        verify(notificationService).sendOrderCreatedNotification(any(), any());
    }
    
    @Test
    @DisplayName("Should throw exception when customer not found")
    void shouldThrowExceptionWhenCustomerNotFound() {
        CreateOrderRequest request = new CreateOrderRequest(customerId, Arrays.asList());
        
        when(orderRepository.save(any(Order.class)))
            .thenThrow(new CustomerNotFoundException(customerId));
        
        assertThrows(CustomerNotFoundException.class, () -> {
            orderService.createOrder(request);
        });
        
        verify(notificationService, never()).sendOrderCreatedNotification(any(), any());
    }
    
    @Test
    @DisplayName("Should confirm order with valid payment")
    void shouldConfirmOrderWithValidPayment() {
        Order order = new Order(orderId, customerId, createTestItems());
        
        when(orderRepository.findById(orderId)).thenReturn(Optional.of(order));
        when(paymentService.processPayment(any())).thenReturn(true);
        
        orderService.confirmOrder(orderId);
        
        verify(paymentService).processPayment(any());
        verify(orderRepository).save(any(Order.class));
    }
    
    private List<OrderItem> createTestItems() {
        return Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("100.00"))
        );
    }
}
```

## Integration Testing

### Repository Integration Tests

```java
@SpringBootTest
@Transactional
class OrderRepositoryIntegrationTest {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private TestEntityManager entityManager;
    
    private OrderId orderId;
    private CustomerId customerId;
    
    @BeforeEach
    void setUp() {
        orderId = OrderId.generate();
        customerId = CustomerId.generate();
    }
    
    @Test
    @DisplayName("Should save and retrieve order")
    void shouldSaveAndRetrieveOrder() {
        Order order = new Order(
            orderId,
            customerId,
            Arrays.asList(
                new OrderItem(ProductId.generate(), Quantity.of(2), Money.of("100.00"))
            )
        );
        
        Order saved = orderRepository.save(order);
        entityManager.flush();
        entityManager.clear();
        
        Optional<Order> found = orderRepository.findById(orderId);
        
        assertTrue(found.isPresent());
        assertEquals(saved.getId(), found.get().getId());
        assertEquals(saved.getCustomerId(), found.get().getCustomerId());
    }
    
    @Test
    @DisplayName("Should find orders by customer ID")
    void shouldFindOrdersByCustomerId() {
        Order order1 = new Order(
            OrderId.generate(),
            customerId,
            createTestItems()
        );
        Order order2 = new Order(
            OrderId.generate(),
            customerId,
            createTestItems()
        );
        
        orderRepository.save(order1);
        orderRepository.save(order2);
        entityManager.flush();
        
        List<Order> orders = orderRepository.findByCustomerId(customerId);
        
        assertEquals(2, orders.size());
    }
    
    @Test
    @DisplayName("Should delete order")
    void shouldDeleteOrder() {
        Order order = new Order(orderId, customerId, createTestItems());
        orderRepository.save(order);
        entityManager.flush();
        
        orderRepository.delete(order);
        entityManager.flush();
        
        Optional<Order> found = orderRepository.findById(orderId);
        assertFalse(found.isPresent());
    }
    
    private List<OrderItem> createTestItems() {
        return Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("100.00"))
        );
    }
}
```

### REST Controller Integration Tests

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
class OrderControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private OrderApplicationService orderService;
    
    @Test
    @DisplayName("Should create order and return 200")
    void shouldCreateOrderAndReturn200() throws Exception {
        CreateOrderRequest request = new CreateOrderRequest(
            CustomerId.generate(),
            Arrays.asList(
                new OrderItemRequest(ProductId.generate(), Quantity.of(2))
            )
        );
        
        OrderId orderId = OrderId.generate();
        when(orderService.createOrder(any())).thenReturn(orderId);
        
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.orderId").value(orderId.getValue()));
    }
    
    @Test
    @DisplayName("Should return 400 for invalid request")
    void shouldReturn400ForInvalidRequest() throws Exception {
        CreateOrderRequest request = new CreateOrderRequest(null, null);
        
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest());
    }
    
    @Test
    @DisplayName("Should get order by ID")
    void shouldGetOrderById() throws Exception {
        OrderId orderId = OrderId.generate();
        OrderDTO orderDTO = new OrderDTO(
            orderId.getValue(),
            CustomerId.generate().getValue(),
            "CREATED",
            BigDecimal.valueOf(100.00)
        );
        
        when(orderService.findOrderById(orderId)).thenReturn(orderDTO);
        
        mockMvc.perform(get("/api/orders/{orderId}", orderId.getValue()))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.orderId").value(orderId.getValue()))
            .andExpect(jsonPath("$.status").value("CREATED"));
    }
}
```

## Axon Framework Testing

### Aggregate Test Fixture

```java
class OrderAggregateTest {
    
    private AggregateTestFixture<Order> fixture;
    
    @BeforeEach
    void setUp() {
        fixture = new AggregateTestFixture<>(Order.class);
    }
    
    @Test
    @DisplayName("Should create order and publish event")
    void shouldCreateOrderAndPublishEvent() {
        OrderId orderId = OrderId.generate();
        CustomerId customerId = CustomerId.generate();
        List<OrderItem> items = Arrays.asList(
            new OrderItem(ProductId.generate(), Quantity.of(2), Money.of("100.00"))
        );
        
        fixture.givenNoPriorActivity()
            .when(new CreateOrderCommand(orderId, customerId, items))
            .expectEventsMatching(payloadsMatching(
                matches(event -> {
                    OrderCreatedEvent e = (OrderCreatedEvent) event;
                    return e.getOrderId().equals(orderId) &&
                           e.getCustomerId().equals(customerId);
                })
            ))
            .expectSuccessfulHandlerExecution();
    }
    
    @Test
    @DisplayName("Should not confirm empty order")
    void shouldNotConfirmEmptyOrder() {
        OrderId orderId = OrderId.generate();
        
        fixture.given(new OrderCreatedEvent(
                orderId,
                CustomerId.generate(),
                Collections.emptyList(),
                LocalDateTime.now()
            ))
            .when(new ConfirmOrderCommand(orderId))
            .expectException(IllegalStateException.class)
            .expectExceptionMessage("Cannot confirm an empty order");
    }
    
    @Test
    @DisplayName("Should add item to order")
    void shouldAddItemToOrder() {
        OrderId orderId = OrderId.generate();
        ProductId productId = ProductId.generate();
        
        fixture.given(new OrderCreatedEvent(
                orderId,
                CustomerId.generate(),
                Arrays.asList(
                    new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("50.00"))
                ),
                LocalDateTime.now()
            ))
            .when(new AddOrderItemCommand(orderId, productId, Quantity.of(2)))
            .expectEventsMatching(payloadsMatching(
                matches(event -> {
                    OrderItemAddedEvent e = (OrderItemAddedEvent) event;
                    return e.getOrderId().equals(orderId) &&
                           e.getProductId().equals(productId);
                })
            ));
    }
}
```

### Saga Test Fixture

```java
class OrderManagementSagaTest {
    
    private SagaTestFixture<OrderManagementSaga> fixture;
    
    @BeforeEach
    void setUp() {
        fixture = new SagaTestFixture<>(OrderManagementSaga.class);
    }
    
    @Test
    @DisplayName("Should start saga on order created")
    void shouldStartSagaOnOrderCreated() {
        OrderId orderId = OrderId.generate();
        
        fixture.givenNoPriorActivity()
            .whenPublishingA(new OrderCreatedEvent(
                orderId,
                CustomerId.generate(),
                Arrays.asList(new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("100.00")))
            ))
            .expectActiveSagas(1)
            .expectDispatchedCommandsMatching(
                payloadsMatching(exactMatch(any(CreatePaymentCommand.class)))
            );
    }
    
    @Test
    @DisplayName("Should complete saga when payment and shipment done")
    void shouldCompleteSagaWhenPaymentAndShipmentDone() {
        OrderId orderId = OrderId.generate();
        PaymentId paymentId = PaymentId.generate();
        ShipmentId shipmentId = ShipmentId.generate();
        
        fixture.givenAggregate(orderId.toString())
            .published(new OrderCreatedEvent(
                orderId,
                CustomerId.generate(),
                Arrays.asList(new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("100.00")))
            ))
            .andThenAggregate(paymentId.toString())
            .published(new PaymentCompletedEvent(paymentId, orderId))
            .andThenAggregate(shipmentId.toString())
            .published(new ShipmentCompletedEvent(shipmentId, orderId))
            .whenPublishingA(new ShipmentCompletedEvent(shipmentId, orderId))
            .expectDispatchedCommandsMatching(
                payloadsMatching(exactMatch(any(CompleteOrderCommand.class)))
            )
            .expectActiveSagas(0);
    }
}
```

## Test Configuration

### Test Properties

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  axon:
    axonserver:
      enabled: false

axon:
  eventhandling:
    processors:
      order-group:
        mode: subscribing
```

### Test Configuration Class

```java
@TestConfiguration
public class TestConfiguration {
    
    @Bean
    @Primary
    public Clock testClock() {
        return Clock.fixed(
            Instant.parse("2024-01-01T10:00:00Z"),
            ZoneId.of("UTC")
        );
    }
    
    @Bean
    @Primary
    public EventStore testEventStore() {
        return new InMemoryEventStore();
    }
    
    @Bean
    @Primary
    public CommandGateway testCommandGateway() {
        return DefaultCommandGateway.builder()
            .commandBus(new SimpleCommandBus())
            .build();
    }
}
```

## Test Data Builders

### Test Data Builder Pattern

```java
public class OrderTestDataBuilder {
    
    private OrderId orderId = OrderId.generate();
    private CustomerId customerId = CustomerId.generate();
    private List<OrderItem> items = Arrays.asList(
        new OrderItem(ProductId.generate(), Quantity.of(1), Money.of("100.00"))
    );
    private OrderStatus status = OrderStatus.CREATED;
    
    public OrderTestDataBuilder withOrderId(OrderId orderId) {
        this.orderId = orderId;
        return this;
    }
    
    public OrderTestDataBuilder withCustomerId(CustomerId customerId) {
        this.customerId = customerId;
        return this;
    }
    
    public OrderTestDataBuilder withItems(List<OrderItem> items) {
        this.items = items;
        return this;
    }
    
    public OrderTestDataBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }
    
    public Order build() {
        Order order = new Order(orderId, customerId, items);
        if (status != OrderStatus.CREATED) {
            order.setStatus(status);
        }
        return order;
    }
    
    public static OrderTestDataBuilder anOrder() {
        return new OrderTestDataBuilder();
    }
}

// Usage in tests
Order order = OrderTestDataBuilder.anOrder()
    .withCustomerId(customerId)
    .withStatus(OrderStatus.CONFIRMED)
    .build();
```

## Test Coverage

### JaCoCo Configuration

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Best Practices

### 1. Test Naming
- Use descriptive test names with @DisplayName
- Follow "should[ExpectedBehavior]When[ConditionUnderTest]" pattern
- Use parameterized tests for multiple scenarios

### 2. Test Independence
- Each test should be independent
- Use @BeforeEach to set up test state
- Clean up resources in @AfterEach

### 3. Test Readability
- Follow Given-When-Then structure
- Use meaningful variable names
- Keep tests simple and focused

### 4. Mocking Strategy
- Mock external dependencies
- Use real objects for domain logic
- Verify interactions when necessary

### 5. Test Coverage Goals
- Aim for 80%+ code coverage
- Focus on critical business logic
- Don't test getters/setters

### 6. Performance Testing
- Use @Timed for performance tests
- Set reasonable timeouts
- Profile slow tests

## References

- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Documentation](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html)
- [Axon Testing Guide](https://docs.axoniq.io/reference-guide/axon-framework/testing)
- [Test Driven Development](https://www.oreilly.com/library/view/test-driven-development/0321146530/)
