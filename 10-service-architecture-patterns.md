# 🏗️ Service Architecture Patterns

This module explores fundamental architectural patterns that shape how we structure backend services, from traditional layered designs to modern domain-driven approaches. Think of these patterns as blueprints for building houses—each design serves a specific purpose and has its own strengths and trade-offs.

## 🎯 Learning Goals
- Understand the evolution and trade-offs of different architectural patterns
- Master the principles of hexagonal architecture and clean architecture
- Learn how to apply domain-driven design in service architecture
- Evaluate architectural patterns for different use cases

## 🏢 Three-Tier Architecture

### Traditional Layered Design
The three-tier architecture divides an application into three logical layers, each with specific responsibilities and concerns. This separation helps manage complexity but can lead to tight coupling if not implemented carefully.

**Presentation Layer:**  
Handles HTTP requests and responses. It's like the reception desk at a hotel—it interacts directly with clients, taking their requests and delivering responses.

```javascript
// Presentation Layer (Controllers)
class UserController {
  constructor(userService) {
    this.userService = userService;
  }
  async createUser(req, res) {
    try {
      const user = await this.userService.createUser(req.body);
      res.status(201).json(user); // Respond with created user
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}
```

**Business Logic Layer:**  
Contains the core business rules. Imagine this layer as the kitchen staff preparing meals—it ensures everything is cooked according to the recipe.

```javascript
// Business Logic Layer (Services)
class UserService {
  constructor(userRepository) {
    this.userRepository = userRepository;
  }
  async createUser(userData) {
    // Business logic and validation
    if (userData.age < 18) {
      throw new Error('User must be 18 or older');
    }
    return this.userRepository.save(userData);
  }
}
```

**Data Access Layer:**  
Manages database operations. This layer acts as the pantry, storing ingredients (data) and retrieving them when needed.

```javascript
// Data Access Layer (Repositories)
class UserRepository {
  constructor(database) {
    this.database = database;
  }
  async save(userData) {
    return this.database.users.create(userData);
  }
}
```

> **Critical Point:**  
> While three-tier architecture provides clear separation of concerns, it can lead to tight coupling between layers and make the system harder to maintain as it grows. The key is to maintain proper boundaries and avoid leaking implementation details across layers.

**Misconception:**  
Three-tier architecture is outdated and should be replaced by modern patterns.  \
**Reality:** Three-tier architecture is still widely used and effective for many applications. It provides a solid foundation and can be enhanced with modern patterns like dependency injection and clean architecture principles.

### Layered Design Challenges
One of the main challenges in layered architecture is managing dependencies between layers. Traditional layered architecture creates top-down dependencies, where changes in lower layers can ripple up to affect upper layers. This rigidity makes the system harder to modify over time.

**Dependency Injection Example:**  
Instead of tightly coupling layers, dependency injection allows flexibility by passing dependencies through constructors. Think of it as ordering food à la carte instead of being stuck with a fixed menu.

```javascript
// Problematic dependency flow
class UserService {
  constructor() {
    // Direct dependency on database implementation
    this.database = new MySQLDatabase();
  }
}

// Better approach with dependency injection
class UserService {
  constructor(database) {
    // Depends on abstraction, not implementation
    this.database = database;
  }
}
```

Cross-cutting concerns like logging, security, and caching often span multiple layers, making it challenging to maintain clean separation. These concerns should be handled through middleware or aspects rather than scattering logic across layers.

> **Do's and Don'ts:**  
> - **Do:** Use dependency injection to decouple layers and improve testability.  
> - **Don't:** Scatter cross-cutting concerns across layers—centralize them using middleware or decorators.

**Anti-Pattern:** Creating an "anemic domain model," where business logic resides mostly in service classes rather than domain entities, undermines the purpose of layered architecture.

**Misconception:**  
Layered architecture always leads to tight coupling between layers.  \
**Reality:** Properly implemented layered architecture with dependency injection and clear boundaries can achieve loose coupling while maintaining separation of concerns.

---

## 🔄 Hexagonal Architecture

### Ports & Adapters Pattern
Hexagonal architecture, also known as ports and adapters, inverts the dependency flow by placing the business logic at the center and surrounding it with adapters that handle external concerns. Imagine the core domain as a fortress protected by walls (adapters), which shield it from outside threats.

**Core Domain:**  
Contains the business logic and defines ports (interfaces) for external interactions. It's like the brain of the operation, focused solely on decision-making without worrying about technical details.

```javascript
// Core Domain (Business Logic)
class Order {
  constructor(id, items) {
    this.id = id;
    this.items = items;
    this.status = 'PENDING';
  }
  process() {
    if (this.items.length === 0) {
      throw new Error('Order must have items');
    }
    this.status = 'PROCESSING';
  }
}
```

**Ports:**  
Define interfaces for external interactions. These are like doors in the fortress, controlling access to the core domain.

```javascript
// Port (Interface)
class OrderRepository {
  async save(order) {
    throw new Error('Not implemented');
  }
}
```

**Adapters:**  
Implement these ports and handle technical details. Primary adapters (driving) handle incoming requests, while secondary adapters (driven) interact with external systems like databases.

```javascript
// Primary Adapter (Driving)
class OrderController {
  constructor(orderService) {
    this.orderService = orderService;
  }
  async createOrder(req, res) {
    const order = await this.orderService.createOrder(req.body);
    res.status(201).json(order);
  }
}

// Secondary Adapter (Driven)
class SQLOrderRepository extends OrderRepository {
  constructor(database) {
    super();
    this.database = database;
  }
  async save(order) {
    return this.database.orders.create(order);
  }
}
```

> **Critical Point:**  
> Hexagonal architecture enables better testability and flexibility by inverting dependencies, but requires careful design of ports and adapters to avoid complexity. The key is to keep the core domain pure and independent of external concerns.

**Misconception:**  
Hexagonal architecture is only for complex enterprise applications.  \
**Reality:** Hexagonal architecture can benefit applications of any size by improving testability and maintainability, though the overhead may not be justified for very simple applications.

---

## 🧹 Clean Architecture

### Dependency Inversion
Clean architecture takes the principles of hexagonal architecture further by organizing the system into concentric layers, with dependencies pointing inward. Think of it as Russian nesting dolls—the outer layers depend on the inner ones but not vice versa.

**Domain Layer:**  
Contains the core business logic and entities. It's the heart of the system, free from external influences.

```javascript
// Domain Layer (Entities)
class User {
  constructor(id, email, name) {
    this.id = id;
    this.email = email;
    this.name = name;
  }
  validate() {
    if (!this.email.includes('@')) {
      throw new Error('Invalid email');
    }
  }
}
```

**Use Case Layer:**  
Contains application-specific business rules and orchestrates the flow of data between the domain and external layers.

```javascript
// Use Case Layer
class CreateUserUseCase {
  constructor(userRepository, emailService) {
    this.userRepository = userRepository;
    this.emailService = emailService;
  }
  async execute(userData) {
    const user = new User(userData.id, userData.email, userData.name);
    user.validate();
    const savedUser = await this.userRepository.save(user);
    await this.emailService.sendWelcomeEmail(savedUser.email);
    return savedUser;
  }
}
```

**Interface Adapters Layer:**  
Converts data between the use cases and external agencies like databases, web frameworks, or external APIs.

```javascript
// Interface Adapters Layer
class UserController {
  constructor(createUserUseCase) {
    this.createUserUseCase = createUserUseCase;
  }
  async createUser(req, res) {
    try {
      const user = await this.createUserUseCase.execute(req.body);
      res.status(201).json(user);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}
```

**Frameworks & Drivers Layer:**  
Contains the tools and frameworks that power the application, such as databases, web servers, and external services.

**Misconception:**  
Clean architecture requires more code and complexity than simpler patterns.  \
**Reality:** While clean architecture may require more initial setup, it often reduces complexity in the long run by making the codebase more maintainable, testable, and adaptable to change.

### Clean Architecture Benefits
**Independence of Frameworks:**  
The business logic is independent of the frameworks used. You can change frameworks without affecting the core business rules.

**Testability:**  
The business logic can be tested without external dependencies, making tests faster and more reliable.

**Independence of UI:**  
The user interface can be changed easily without affecting the business logic.

**Independence of Database:**  
You can swap out databases without affecting the business logic.

**Independence of External Agencies:**  
The business logic doesn't depend on external services or APIs.

**Misconception:**  
Clean architecture is only about organizing code into layers.  \
**Reality:** Clean architecture is about dependency management and ensuring that business logic remains independent of external concerns, not just about folder structure.

---

## 🎯 Domain-Driven Design (DDD)

### Strategic Design
Domain-Driven Design focuses on modeling software to match a domain according to input from domain experts. It's like building a house based on the needs and preferences of the people who will live in it.

**Bounded Contexts:**  
A bounded context is a boundary within which a particular model is defined and applicable. Think of it as different departments in a company—each has its own language, rules, and processes.

**Ubiquitous Language:**  
A common language shared between developers and domain experts. This ensures that the code reflects the business domain accurately.

**Context Mapping:**  
Defines relationships between different bounded contexts, such as shared kernels, customer-supplier relationships, or conformist patterns.

**Misconception:**  
DDD is only for complex domains with many business rules.  \
**Reality:** DDD principles can be applied to any domain to improve code clarity and alignment with business requirements, though the level of implementation may vary.

### Tactical Design
**Entities:**  
Objects that have identity and continuity. They are defined by their identity rather than their attributes.

```javascript
// Entity
class Order {
  constructor(id, customerId, items) {
    this.id = id; // Identity
    this.customerId = customerId;
    this.items = items;
    this.status = 'PENDING';
  }
  addItem(item) {
    this.items.push(item);
  }
  process() {
    this.status = 'PROCESSING';
  }
}
```

**Value Objects:**  
Objects that are defined by their attributes rather than their identity. They are immutable and can be shared.

```javascript
// Value Object
class Money {
  constructor(amount, currency) {
    this.amount = amount;
    this.currency = currency;
  }
  add(other) {
    if (this.currency !== other.currency) {
      throw new Error('Cannot add different currencies');
    }
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

**Aggregates:**  
Clusters of domain objects that can be treated as a single unit for data changes. They have a root entity that controls access to the other entities.

**Repositories:**  
Objects that provide access to aggregates, hiding the complexity of data persistence.

**Services:**  
Domain services that contain business logic that doesn't belong to any specific entity or value object.

**Misconception:**  
DDD requires creating complex domain models for every piece of data.  \
**Reality:** DDD should be applied where it adds value. Not every piece of data needs to be a rich domain object—sometimes simple data structures are more appropriate.

### When to Use DDD
**Complex Domains:**  
When the business domain is complex and requires deep understanding.

**Collaborative Development:**  
When working closely with domain experts who can provide insights into the business.

**Long-term Projects:**  
When the software will be maintained and evolved over a long period.

**Misconception:**  
DDD is an all-or-nothing approach that must be applied to the entire system.  \
**Reality:** DDD can be applied selectively to parts of a system where it provides the most value, while other parts may use simpler patterns.

---

## 🔄 Architectural Pattern Selection

### Decision Framework
When choosing an architectural pattern, consider:

**Project Size and Complexity:**  
- Small projects: Simple layered architecture
- Medium projects: Hexagonal architecture with dependency injection
- Large projects: Clean architecture with DDD

**Team Experience:**  
- Junior teams: Start with layered architecture
- Experienced teams: Can handle more complex patterns

**Business Requirements:**  
- Simple CRUD: Layered architecture
- Complex business rules: DDD with clean architecture
- High performance: Consider CQRS or event sourcing

**Technology Constraints:**  
- Framework limitations
- Performance requirements
- Integration needs

**Misconception:**  
There's one "best" architectural pattern that should be used for all projects.  \
**Reality:** The best architectural pattern depends on the specific requirements, team capabilities, and project constraints. Different patterns serve different purposes.

### Migration Strategies
**Gradual Migration:**  
Start with the current architecture and gradually introduce new patterns.

**Strangler Fig Pattern:**  
Replace parts of the system incrementally while keeping the old system running.

**Parallel Implementation:**  
Build the new system alongside the old one and switch over when ready.

**Misconception:**  
Architectural changes require rewriting the entire system.  \
**Reality:** Architectural improvements can often be implemented incrementally through refactoring, dependency injection, and careful design of new components.

## 📝 Quiz

1. Compare and contrast three-tier architecture, hexagonal architecture, and clean architecture. What are the key differences and when would you choose each?
   - **Hint:** Consider complexity, testability, and maintainability factors.

2. Explain the concept of dependency inversion and how it improves system design.
   - **Hint:** Discuss how it affects coupling, testability, and flexibility.

3. What are the key principles of Domain-Driven Design, and how do they help create better software?
   - **Hint:** Focus on ubiquitous language, bounded contexts, and tactical design patterns.

4. How would you approach migrating from a traditional layered architecture to clean architecture?
   - **Hint:** Consider incremental approaches and the strangler fig pattern.

## 🎯 What's Next?

In the next module, we'll explore Business Logic Implementation, diving into how to structure and implement the core business rules that drive your application. You'll learn about service layer patterns, domain modeling, and transaction management strategies.

---

[⬅️ Previous: Authentication & Authorization](09-authentication-authorization.md) | [⬅️ Back to Course Index](README.md) | [➡️ Next: Business Logic Implementation](11-business-logic-implementation.md)
