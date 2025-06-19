# ⚙️ Background Jobs & Scheduling

This module explores strategies for implementing reliable background processing, job scheduling, and task management in distributed systems. Think of background jobs as the behind-the-scenes workers in a restaurant—they handle tasks like cleaning tables or preparing ingredients while the chefs focus on cooking. Similarly, background jobs allow your application to offload time-consuming tasks, ensuring responsiveness and scalability.

## 🎯 Learning Goals
- Understand the principles of background job processing  
- Master job scheduling and task coordination  
- Learn to implement distributed job queues  
- Handle failures, retries, and graceful shutdowns  

---

## 🔄 Background Job Processing

### Job Queues

Job queues enable asynchronous task execution by decoupling producers (tasks) from consumers (workers). This pattern is essential for handling long-running or resource-intensive tasks without blocking the main application flow.

**Local Job Queue:**  
A local job queue processes tasks within the same application instance. It’s simple to implement but lacks scalability. For example, sending emails directly from an API endpoint might work for small applications but can overwhelm the server as traffic grows.

```javascript
// Local Job Queue
class LocalJobQueue {
  constructor() {
    this.queue = [];
    this.processing = false;
  }
  async addJob(job) {
    this.queue.push({ ...job, status: 'pending', createdAt: new Date() });
    this.processQueue();
  }
  async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;
    const job = this.queue.shift();
    try {
      job.status = 'processing';
      job.startedAt = new Date();
      await this.executeJob(job);
      job.status = 'completed';
      job.completedAt = new Date();
    } catch (error) {
      job.status = 'failed';
      job.error = error.message;
      job.failedAt = new Date();
      if (this.shouldRetry(job)) this.queue.push(job);
    } finally {
      this.processing = false;
      this.processQueue();
    }
  }
  shouldRetry(job) {
    return job.retryCount < (job.maxRetries || 3);
  }
  async executeJob(job) {
    switch (job.type) {
      case 'email':
        await this.sendEmail(job.data);
        break;
      case 'report':
        await this.generateReport(job.data);
        break;
      default:
        throw new Error(`Unknown job type: ${job.type}`);
    }
  }
}
```

> **Critical Point:**  
> Local job queues are easy to implement but struggle with scalability and fault tolerance. Distributed job queues address these limitations by distributing tasks across multiple workers.

**Distributed Job Queue:**  
Distributed job queues use tools like Redis or RabbitMQ to manage tasks across multiple instances. For example, Sidekiq (Ruby) or Celery (Python) leverage Redis for task distribution, ensuring high availability and fault tolerance.

```javascript
// Distributed Job Queue
class DistributedJobQueue {
  constructor(redis) {
    this.redis = redis;
  }
  async addJob(job) {
    const jobId = uuid();
    const jobData = { id: jobId, ...job, status: 'pending', createdAt: new Date() };
    await this.redis.hset(`job:${jobId}`, jobData);
    await this.redis.lpush('jobs:pending', jobId);
    return jobId;
  }
  async startWorker(workerId) {
    const worker = new Worker(workerId, this.redis);
    await worker.start();
  }
}
```

> **Tool Examples:**  
> - **BullMQ / Bee-Queue (Node.js):** Ideal for lightweight, fast task distribution.  
> - **RabbitMQ / Kafka:** Message brokers with support for routing, partitions, and persistence. 
> - **Temporal / Prefect:** Distributed orchestration for workflows and retries.

---

## ⏰ Job Scheduling

### Cron-Based Scheduling

Cron-based scheduling automates recurring tasks, such as generating daily reports or performing nightly backups. Think of it as setting alarms for routine chores—it ensures tasks are executed at the right time without manual intervention.

```javascript
// Cron Scheduler
class CronScheduler {
  constructor() {
    this.jobs = new Map();
  }
  schedule(cronExpression, task) {
    const jobId = uuid();
    const job = {
      id: jobId,
      cronExpression,
      task,
      nextRun: this.calculateNextRun(cronExpression)
    };
    this.jobs.set(jobId, job);
    this.scheduleNextRun(job);
    return jobId;
  }
  calculateNextRun(cronExpression) {
    const parser = new CronParser(cronExpression);
    return parser.next();
  }
  scheduleNextRun(job) {
    const now = new Date();
    const delay = job.nextRun.getTime() - now.getTime();
    if (delay > 0) {
      setTimeout(async () => {
        try {
          await job.task();
        } catch (error) {
          console.error(`Error executing job ${job.id}:`, error);
        }
        job.nextRun = this.calculateNextRun(job.cronExpression);
        this.scheduleNextRun(job);
      }, delay);
    }
  }
}
```

**Cron Expression Example:**  
`0 0 * * *` schedules a task to run daily at midnight. Tools like `node-cron` or `cron-parser` simplify parsing and scheduling.

> **Do’s and Don’ts:**  
> - **Do:** Use cron expressions for predictable, recurring tasks.  
> - **Don’t:** Overload cron jobs with complex logic—it increases maintenance complexity.

**Anti-Pattern:** Running long-running tasks in cron jobs can lead to overlapping executions and resource contention.

> 💡 Use locking (e.g., Redis-based redlock) to prevent duplicate runs when scaling scheduled jobs.
---

## 🛠️ Task Coordination

### Distributed Workers

Distributed workers ensure tasks are processed efficiently across multiple nodes. Each worker pulls tasks from a shared queue, processes them, and updates their status. This is like assigning tasks to multiple employees in a warehouse—everyone works simultaneously to complete orders faster.

```javascript
// Distributed Worker
class Worker {
  constructor(id, redis) {
    this.id = id;
    this.redis = redis;
    this.running = false;
  }
  async start() {
    this.running = true;
    while (this.running) {
      const jobId = await this.redis.brpoplpush('jobs:pending', 'jobs:processing', 0);
      if (jobId) {
        await this.processJob(jobId);
      }
    }
  }
  async stop() {
    this.running = false;
  }
  async processJob(jobId) {
    const job = await this.redis.hgetall(`job:${jobId}`);
    try {
      await this.redis.hset(`job:${jobId}`, {
        status: 'processing',
        workerId: this.id,
        startedAt: new Date()
      });
      await this.executeJob(job);
      await this.redis.hset(`job:${jobId}`, {
        status: 'completed',
        completedAt: new Date()
      });
    } catch (error) {
      await this.redis.hset(`job:${jobId}`, {
        status: 'failed',
        error: error.message,
        failedAt: new Date()
      });
      if (this.shouldRetry(job)) {
        await this.redis.lpush('jobs:pending', jobId);
      }
    } finally {
      await this.redis.lrem('jobs:processing', 0, jobId);
    }
  }
}
```

> **Critical Point:**  
> Distributed workers improve throughput and fault tolerance but require robust coordination mechanisms to avoid duplicate processing.

---

## ❌ Handling Failures

### Retries and Dead-Letter Queues

Failures are inevitable in distributed systems, so implementing retry mechanisms and dead-letter queues (DLQs) is crucial. Retries give transient errors a second chance, while DLQs store unrecoverable tasks for later inspection.

**Retry Logic Example:**  
Imagine a delivery driver trying to reach a customer—if the first attempt fails, they wait a bit before retrying.

```javascript
// Retry Mechanism
class JobProcessor {
  constructor(maxRetries = 3) {
    this.maxRetries = maxRetries;
  }
  async processWithRetry(job, processor) {
    let retryCount = 0;
    while (retryCount < this.maxRetries) {
      try {
        await processor(job);
        return; // Success
      } catch (error) {
        retryCount++;
        if (retryCount >= this.maxRetries) {
          throw new Error(`Failed after ${this.maxRetries} retries`);
        }
        await this.wait(Math.pow(2, retryCount) * 100); // Exponential backoff
      }
    }
  }
  wait(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

**Dead-Letter Queue Example:**  
Think of a mail sorting center setting aside undeliverable packages for further investigation.

```javascript
// Dead-Letter Queue Handling
class DeadLetterQueueManager {
  constructor(queueClient) {
    this.queueClient = queueClient;
  }
  async handleFailedJob(job, error) {
    console.warn(`Moving failed job to DLQ: ${job.id}`);
    await this.queueClient.publish('dead-letter-queue', {
      originalJob: job,
      error: error.message
    });
  }
}
```

> **Do’s and Don’ts:**  
> - **Do:** Regularly monitor and process DLQs to prevent data loss.  
> - **Don’t:** Ignore DLQs—they accumulate unresolved issues over time.

**Anti-Pattern:** Failing to implement retries and DLQs leads to lost tasks and unreliable systems.

---

## 📝 Quiz

1. Compare local and distributed job queues, explaining when to use each.
   - **Hint:** Focus on scalability, fault tolerance, and implementation complexity.

2. How do you implement a reliable retry mechanism for background jobs?
   - **Hint:** Include exponential backoff, retries, and dead-letter queues.

3. What are the key considerations when designing a task coordination system?
   - **Hint:** Discuss worker distribution, task assignment, and failure handling.

4. Explain the challenges of implementing background job processing and how to overcome them.
   - **Hint:** Address concurrency, retries, and monitoring.

---

## 🎯 What's Next?

In the next module, we’ll explore Event Sourcing, a powerful design pattern that models changes in application state as a sequence of immutable events. You’ll also learn how to pair it with CQRS (Command Query Responsibility Segregation) to improve scalability and maintainability in complex systems.

---

[⬅️ Previous: Message Queues & Event Systems](16-message-queues-event-systems.md) | [⬅️ Back to Course Index](README.md) | [➡️ Next: Event Sourcing](18-event-sourcing-cqrs.md)
