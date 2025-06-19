# 💼 Business Logic Implementation

This module delves into the core principles and patterns for implementing business logic in backend services, ensuring maintainable, testable, and consistent business rules. Think of business logic as the brain of your application—it makes decisions, enforces rules, and ensures everything operates smoothly.

## 🎯 Learning Goals
- Master service layer patterns and their responsibilities
- Learn effective domain modeling and entity design
- Implement robust business rules validation
- Understand transaction management and data consistency

---

## 🏗️ Service Layer Patterns

### Service Responsibilities

The service layer is responsible for orchestrating business operations and enforcing business rules. It acts as a mediator between the presentation layer (e.g., APIs) and the data access layer, ensuring that business rules are consistently applied. This layer is like a project manager who coordinates tasks without getting bogged down in technical details.

For example, when placing an order:
1. The service validates business rules (e.g., ensuring the cart isn’t empty).  
2. It interacts with repositories to save the order.  
3. It processes payments via external gateways.  
4. Finally, it sends notifications to the customer.

```javascript
// Application Service
class OrderService {
  constructor(orderRepository, paymentGateway, notificationService) {
    this.orderRepository = orderRepository;
    this.paymentGateway = paymentGateway;
    this.notificationService = notificationService;
  }
  async createOrder(orderData) {
    // Validate business rules
    if (orderData.items.length === 0) {
      throw new BusinessError('Order must have at least one item');
    }
    // Create and save order
    const order = new Order(orderData);
    await this.orderRepository.save(order);
    // Process payment
    const payment = await this.paymentGateway.processPayment(order);
    order.setPaymentStatus(payment.status);
    // Send notification
    await this.notificationService.sendOrderConfirmation(order);
    return order;
  }
}
```

> **Critical Point:**  
> Services should focus on business operations rather than technical details, delegating infrastructure concerns to appropriate layers. The key is to maintain a clear separation between business logic and technical implementation.

**Misconception:** 
The service layer is just a wrapper around database calls.  \
**Reality:** It’s the heart of business logic, orchestrating operations and enforcing rules.

> **Do’s and Don’ts:**  
> - **Do:** Keep services focused on business operations and delegate technical concerns to other layers.  
> - **Don’t:** Mix business logic with infrastructure code—it leads to tightly coupled systems.

**Anti-Pattern:** Placing all logic in controllers creates a "fat controller" anti-pattern, making the system hard to test and maintain.

---

### Service Implementation Patterns

The **Command Pattern** is particularly useful for implementing business operations. It encapsulates each operation as an object, providing a clear structure for handling business rules. For example, creating an order becomes a `CreateOrderCommand` that can be executed consistently across the system.

```javascript
// Command Pattern
class CreateOrderCommand {
  constructor(orderData) {
    this.orderData = orderData;
  }
  async execute(orderService) {
    return orderService.createOrder(this.orderData);
  }
}

// Command Handler
class OrderCommandHandler {
  constructor(orderService) {
    this.orderService = orderService;
  }
  async handle(command) {
    if (command instanceof CreateOrderCommand) {
      return this.orderService.createOrder(command.orderData);
    }
    throw new Error('Unknown command');
  }
}
```

> **Decision Framework:**  
> Use the Command Pattern for complex operations requiring clear boundaries and reusability. For simpler operations, direct service methods may suffice.

---

## 🎨 Domain Modeling

### Entity Design

Entities are the core building blocks of the domain model. They have a unique identity and lifecycle, encapsulating both data and behavior. For example, a `Customer` entity might manage its orders, while an `Order` entity tracks its items and status.

```javascript
// Entity with Identity
class Customer {
  constructor(id, name, email) {
    this.id = id;
    this.name = name;
    this.email = email;
    this.orders = [];
  }
  placeOrder(orderData) {
    const order = new Order(orderData);
    this.orders.push(order);
    return order;
  }
  getOrderHistory() {
    return this.orders;
  }
}
```

Value objects, on the other hand, represent immutable attributes like addresses or prices. They ensure consistency by being compared based on value, not identity.

```javascript
// Value Object
class Address {
  constructor(street, city, state, zipCode) {
    this.street = street;
    this.city = city;
    this.state = state;
    this.zipCode = zipCode;
  }
  equals(other) {
    return this.street === other.street &&
           this.city === other.city &&
           this.state === other.state &&
           this.zipCode === other.zipCode;
  }
}
```

> **Critical Point:**  
> Effective domain modeling requires deep understanding of the business domain and careful consideration of entity relationships and boundaries. The key is to model the domain in a way that reflects the business reality.

### Trade-Offs:
- **Entities:** Provide identity and lifecycle but require more complexity.  
- **Value Objects:** Simplify immutability and equality checks but lack identity.

---

### Domain Rules

Business rules can range from simple validations to complex workflows. For instance, an order might require at least one item, while complex rules might involve seasonal discounts or credit limits.

```javascript
// Business Rule Implementation
class OrderValidator {
  validateOrder(order) {
    const errors = [];
    // Required fields
    if (!order.customerId) {
      errors.push('Customer ID is required');
    }
    // Business rules
    if (order.items.length === 0) {
      errors.push('Order must have at least one item');
    }
    return errors;
  }
}
```

Rule engines take this further by allowing dynamic evaluation of rules based on context.

```javascript
// Rule Engine
class RuleEngine {
  constructor(rules) {
    this.rules = rules;
  }
  evaluate(context) {
    return this.rules
      .map(rule => rule.evaluate(context))
      .filter(result => !result.isValid)
      .map(result => result.error);
  }
}
```

> **Do’s and Don’ts:**  
> - **Do:** Centralize business rules to avoid duplication and inconsistency.  
> - **Don’t:** Scatter rules across layers—it leads to fragmented logic.

**Anti-Pattern:** Hardcoding rules directly into controllers or services reduces flexibility and maintainability.

---

## ✅ Business Rules Validation

### Validation Strategies

Input validation ensures data integrity, while business rule validation enforces domain-specific constraints. For example, validating an order involves checking both input formats and business policies like credit limits.

```javascript
// Input Validation
class OrderInputValidator {
  validate(orderData) {
    const errors = [];
    if (!orderData.customerId.match(/^CUST-\d{6}$/)) {
      errors.push('Invalid customer ID format');
    }
    return errors;
  }
}

// Business Rule Validation
class OrderBusinessValidator {
  validate(order, customer) {
    const errors = [];
    if (customer.isBlocked) {
      errors.push('Customer account is blocked');
    }
    return errors;
  }
}
```

> **Critical Point:**  
> Business rules should be consistently enforced across all entry points to the system, with clear error handling and user feedback. The key is to validate both input data and business rules at appropriate levels.

---

## 🔄 Transaction Management

### Transaction Patterns

Transaction management ensures data consistency, especially in distributed systems. Local transactions use database sessions to group operations, while distributed transactions rely on patterns like the Saga pattern to coordinate across services.

```javascript
// Local Transaction
class OrderService {
  async createOrder(orderData) {
    const session = await this.database.startSession();
    try {
      session.startTransaction();
      // Create order
      const order = await this.orderRepository.save(orderData, session);
      // Update inventory
      await this.inventoryService.updateStock(order.items, session);
      await session.commitTransaction();
      return order;
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}
```

For distributed systems, compensating transactions reverse partial changes during failures.

```javascript
// Distributed Transaction (Saga Pattern)
class OrderSaga {
  async execute(orderData) {
    try {
      // Create order
      const order = await this.orderService.createOrder(orderData);
      // Process payment
      await this.paymentService.processPayment(order);
      return order;
    } catch (error) {
      // Compensating transactions
      await this.compensate(order);
      throw error;
    }
  }
  async compensate(order) {
    if (order) {
      await this.orderService.cancelOrder(order.id);
    }
  }
}
```

> **Decision Framework:**  
> Use local transactions for single-service operations and Sagas for distributed systems. Choose based on consistency requirements and failure tolerance.

---

## 📝 Quiz

1. Explain the key responsibilities of the service layer and how it differs from domain services.
   - **Hint:** Focus on orchestration, business logic, and separation of concerns.

2. How does the command pattern help in implementing business operations, and what are its benefits?
   - **Hint:** Highlight encapsulation, reusability, and clear boundaries.

3. What are the key considerations when designing domain entities and implementing business rules?
   - **Hint:** Discuss identity, behavior, and rule centralization.

---

## 🎯 What's Next?

In the next module, we'll explore Inter-Service Communication, focusing on how services communicate with each other in a distributed system, including synchronous and asynchronous communication patterns, message formats, and error handling.

---

[⬅️ Previous: Service Architecture Patterns](10-service-architecture-patterns.md) | [⬅️ Back to Course Index](README.md) | [➡️ Next: Inter-Service Communication](12-inter-service-communication.md)
