# 🏗️ Advanced Architectural Patterns

This module covers advanced architectural patterns for building scalable and maintainable distributed systems. Understanding these patterns is crucial for solving complex architectural challenges in modern applications. Think of these patterns as blueprints for constructing skyscrapers—they provide the structure and guidance needed to handle the complexity of large-scale systems.

## 🎯 Learning Goals
- Master the Saga pattern for distributed transactions  
- Learn Event sourcing and CQRS advanced patterns  
- Understand the Strangler fig pattern for legacy migration  
- Implement Micro-frontends and backend-for-frontend (BFF)  

---

## 🔄 Saga Pattern

### Distributed Transaction Implementation

The Saga pattern addresses the challenge of managing distributed transactions across multiple services in a microservices architecture. Instead of relying on traditional ACID transactions, it uses a sequence of local transactions coordinated through compensating actions.

**Key Features:**  
- **Orchestrated vs. Choreographed Sagas:**  
  - **Orchestrated Sagas:** A central orchestrator manages the sequence of steps and triggers compensating actions if something fails.  
  - **Choreographed Sagas:** Each service listens for events and performs its part, with no central coordinator.  
- **Compensating Actions:** If a step fails, the system executes rollback operations to undo previous steps.  

> **Analogy:**  
> The Saga pattern is like planning a multi-leg journey—each leg must succeed, but if one fails, you have a backup plan to return to the starting point.

**Misconception:**  
Sagas are only for financial transactions. \
**Reality:** They can be applied to any distributed process that requires coordination.

> **Do’s and Don’ts:**  
> - **Do:** Use orchestrated Sagas for complex workflows where centralized control is beneficial.  
> - **Don’t:** Overcomplicate choreographed Sagas—they work best for simple, decoupled processes.

**Anti-Pattern:**  
Implementing a Saga without proper compensating actions can leave the system in an inconsistent state.

**Code Snippet:**
```javascript
// OrderSaga.js
class OrderSaga {
    constructor(orderService, paymentService, inventoryService) {
        this.orderService = orderService;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }

    async execute(orderData) {
        const sagaId = this.generateSagaId();
        const steps = [
            {
                name: 'createOrder',
                action: () => this.orderService.createOrder(orderData),
                compensation: (order) => this.orderService.cancelOrder(order.id)
            },
            {
                name: 'processPayment',
                action: (order) => this.paymentService.processPayment(order),
                compensation: (payment) => this.paymentService.refundPayment(payment.id)
            },
            {
                name: 'updateInventory',
                action: (order) => this.inventoryService.updateInventory(order),
                compensation: (inventory) => this.inventoryService.revertInventory(inventory.id)
            }
        ];

        const context = {
            sagaId,
            orderData,
            results: {},
            status: 'in_progress'
        };

        try {
            for (const step of steps) {
                context.results[step.name] = await step.action(context.results);
            }
            context.status = 'completed';
            return context;
        } catch (error) {
            await this.compensate(steps, context);
            throw error;
        }
    }

    async compensate(steps, context) {
        for (const step of steps.reverse()) {
            if (context.results[step.name]) {
                try {
                    await step.compensation(context.results[step.name]);
                } catch (error) {
                    console.error(`Compensation failed for step ${step.name}:`, error);
                }
            }
        }
        context.status = 'failed';
    }

    generateSagaId() {
        return `saga_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }
}

// Usage example
const orderSaga = new OrderSaga(
    new OrderService(),
    new PaymentService(),
    new InventoryService()
);

try {
    const result = await orderSaga.execute({
        userId: 'user123',
        items: [
            { productId: 'prod1', quantity: 2 },
            { productId: 'prod2', quantity: 1 }
        ],
        total: 150.00
    });
    console.log('Saga completed successfully:', result);
} catch (error) {
    console.error('Saga failed:', error);
}
```


---

## 📝 Event Sourcing & CQRS

### Event Store Implementation

Event Sourcing and Command Query Responsibility Segregation (CQRS) are powerful patterns for managing state and scaling read-heavy systems. Event Sourcing stores changes as a sequence of events, while CQRS separates read and write operations into distinct models.

**Key Features:**  
- **Event Sourcing:** Captures every state change as an immutable event, enabling replayability and auditability.  
- **CQRS:** Separates the read model (queries) from the write model (commands), allowing independent scaling and optimization.  

> **Decision Framework:**  
> Use Event Sourcing when auditability and traceability are critical, and CQRS when read and write loads differ significantly.

**Misconception:**  
Event Sourcing and CQRS must always be used together. \
**Reality:** While they complement each other, they can also be implemented independently.

> **Do’s and Don’ts:**  
> - **Do:** Use projections to derive read models from the event store for efficient querying.  
> - **Don’t:** Overuse Event Sourcing—it adds complexity and may not be suitable for all use cases.

**Anti-Pattern:**  
Storing derived state in the event store instead of raw events undermines the benefits of Event Sourcing.

**Code Snippet:**
```javascript
// EventStore.js
class EventStore {
    constructor() {
        this.events = new Map();
        this.subscribers = new Set();
    }

    async append(streamId, event) {
        const stream = this.events.get(streamId) || [];
        const eventWithMetadata = {
            ...event,
            id: this.generateEventId(),
            streamId,
            version: stream.length + 1,
            timestamp: new Date()
        };
        stream.push(eventWithMetadata);
        this.events.set(streamId, stream);
        await this.notifySubscribers(eventWithMetadata);
        return eventWithMetadata;
    }

    async getEvents(streamId) {
        return this.events.get(streamId) || [];
    }

    async subscribe(subscriber) {
        this.subscribers.add(subscriber);
    }

    async notifySubscribers(event) {
        for (const subscriber of this.subscribers) {
            await subscriber(event);
        }
    }

    generateEventId() {
        return `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }
}

// CQRS Implementation
class OrderCommandHandler {
    constructor(eventStore) {
        this.eventStore = eventStore;
    }

    async handleCreateOrder(command) {
        const event = {
            type: 'OrderCreated',
            data: {
                orderId: command.orderId,
                userId: command.userId,
                items: command.items,
                total: command.total
            }
        };
        await this.eventStore.append(`order_${command.orderId}`, event);
    }

    async handleUpdateOrderStatus(command) {
        const event = {
            type: 'OrderStatusUpdated',
            data: {
                orderId: command.orderId,
                status: command.status
            }
        };
        await this.eventStore.append(`order_${command.orderId}`, event);
    }
}

class OrderQueryHandler {
    constructor(eventStore) {
        this.eventStore = eventStore;
        this.orders = new Map();
    }

    async handleEvent(event) {
        if (event.type === 'OrderCreated') {
            this.orders.set(event.data.orderId, {
                ...event.data,
                status: 'created'
            });
        } else if (event.type === 'OrderStatusUpdated') {
            const order = this.orders.get(event.data.orderId);
            if (order) {
                order.status = event.data.status;
            }
        }
    }

    async getOrder(orderId) {
        return this.orders.get(orderId);
    }

    async getAllOrders() {
        return Array.from(this.orders.values());
    }
}
```


---

## 🌳 Strangler Fig Pattern

### Legacy Migration Implementation

The Strangler Fig pattern facilitates the gradual migration of legacy systems by incrementally replacing components with new implementations. It minimizes risk by allowing old and new systems to coexist during the transition.

**Key Features:**  
- **Incremental Replacement:** Gradually replace parts of the legacy system without downtime.  
- **Facade Layer:** Acts as a bridge between the old and new systems, routing requests appropriately.  

> **Analogy:**  
> The Strangler Fig pattern is like renovating a house room by room—you don’t demolish the entire structure at once.

**Misconception:**  
Some assume the pattern is only for monolithic systems. \
**Reality:** It can also be applied to modularize or modernize parts of a distributed system.

> **Do’s and Don’ts:**  
> - **Do:** Start with low-risk components to build confidence in the migration process.  
> - **Don’t:** Attempt a big-bang migration—it increases risk and disrupts users.

**Anti-Pattern:**  
Replacing too many components at once can lead to integration issues and regression bugs.

**Code Snippet:**
```javascript
// StranglerFig.js
class StranglerFig {
    constructor(legacySystem, newSystem) {
        this.legacySystem = legacySystem;
        this.newSystem = newSystem;
        this.migrationState = new Map();
    }

    async handleRequest(request) {
        const featureId = this.identifyFeature(request);
        const state = this.migrationState.get(featureId) || 'legacy';

        if (state === 'new') {
            return this.newSystem.handleRequest(request);
        } else if (state === 'legacy') {
            return this.legacySystem.handleRequest(request);
        } else {
            // Parallel run mode
            const [legacyResult, newResult] = await Promise.all([
                this.legacySystem.handleRequest(request),
                this.newSystem.handleRequest(request)
            ]);

            if (this.compareResults(legacyResult, newResult)) {
                return newResult;
            } else {
                console.error('Results mismatch:', { legacyResult, newResult });
                return legacyResult;
            }
        }
    }

    async migrateFeature(featureId, targetState) {
        if (!['legacy', 'parallel', 'new'].includes(targetState)) {
            throw new Error('Invalid target state');
        }

        this.migrationState.set(featureId, targetState);
        await this.persistMigrationState();
    }

    identifyFeature(request) {
        // Implementation to identify which feature the request belongs to
        return request.path.split('/')[1];
    }

    compareResults(legacyResult, newResult) {
        // Implementation to compare results from both systems
        return JSON.stringify(legacyResult) === JSON.stringify(newResult);
    }

    async persistMigrationState() {
        // Implementation to persist migration state
    }
}
```

---

## 🎨 Micro-frontends & BFF

### Backend-for-Frontend Implementation

Micro-frontends and Backend-for-Frontend (BFF) enable teams to build scalable, modular frontends tailored to specific user experiences. Micro-frontends break the frontend into smaller, independent pieces, while BFF provides backend APIs optimized for each frontend.

**Key Features:**  
- **Micro-frontends:** Allow different teams to develop and deploy frontend components independently.  
- **BFF:** Creates tailored APIs for specific clients (e.g., web, mobile) to reduce over-fetching and under-fetching of data.  

> **Analogy:**  
> Micro-frontends are like assembling a puzzle—each piece fits together to form the complete picture. BFF ensures each piece gets exactly what it needs.

**Misconception:**  
Micro-frontends require a single framework. \
**Reality:** In reality, teams can use different technologies for each micro-frontend.

> **Do’s and Don’ts:**  
> - **Do:** Use a shared design system to maintain consistency across micro-frontends.  
> - **Don’t:** Overload the BFF with unnecessary logic—it should focus on optimizing API responses.

**Anti-Pattern:**  
Tightly coupling micro-frontends can negate the benefits of modularity and independence.

**Code Snippet:**
```javascript
// BFFService.js
class BFFService {
    constructor(userService, orderService, productService) {
        this.userService = userService;
        this.orderService = orderService;
        this.productService = productService;
    }

    async getUserDashboard(userId) {
        const [user, orders, recommendations] = await Promise.all([
            this.userService.getUser(userId),
            this.orderService.getUserOrders(userId),
            this.productService.getRecommendations(userId)
        ]);

        return {
            user: {
                id: user.id,
                name: user.name,
                email: user.email
            },
            orders: orders.map(order => ({
                id: order.id,
                status: order.status,
                total: order.total,
                items: order.items.map(item => ({
                    id: item.id,
                    name: item.name,
                    quantity: item.quantity,
                    price: item.price
                }))
            })),
            recommendations: recommendations.map(rec => ({
                id: rec.id,
                name: rec.name,
                price: rec.price,
                image: rec.image
            }))
        };
    }

    async getProductDetails(productId, userId) {
        const [product, userPreferences, relatedProducts] = await Promise.all([
            this.productService.getProduct(productId),
            this.userService.getPreferences(userId),
            this.productService.getRelatedProducts(productId)
        ]);

        return {
            product: {
                id: product.id,
                name: product.name,
                description: product.description,
                price: product.price,
                images: product.images,
                stock: product.stock
            },
            userPreferences: {
                currency: userPreferences.currency,
                language: userPreferences.language
            },
            relatedProducts: relatedProducts.map(prod => ({
                id: prod.id,
                name: prod.name,
                price: prod.price,
                image: prod.image
            }))
        };
    }
}
```

---

## 📝 Quiz

1. How does the Saga pattern handle distributed transactions? What are its advantages and limitations?  
   - **Hint:** Discuss compensating actions, orchestration vs. choreography, and trade-offs.

2. Explain the benefits of Event Sourcing and CQRS. When would you choose to implement these patterns?  
   - **Hint:** Address auditability, scalability, and separation of concerns.

3. How does the Strangler fig pattern help in migrating legacy systems? What are the key considerations?  
   - **Hint:** Focus on incremental replacement, facade layers, and minimizing risk.

4. What are the benefits of using a Backend-for-Frontend pattern? How does it improve the development experience?  
   - **Hint:** Discuss tailored APIs, reduced over-fetching, and team independence.

---

## 🎯 What's Next?

In the next module, we'll explore emerging technologies that are shaping the future of backend development. We'll dive into serverless architecture, edge computing, GraphQL Federation, and WebAssembly, learning how these technologies can be leveraged to build more efficient and scalable systems.

---

[⬅️ Previous: Code Quality & Standards](32-code-quality-standards.md) | [⬅️ Back to Course Index](README.md) | [➡️ Next: Emerging Technologies](34-emerging-technologies.md)
