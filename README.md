# Microservices Assessment, Manasa P.

Two Java/Spring Boot microservices for a multi-tenant e-commerce platform: **order-service** and **notification-service**. This README explains how the system works, what I decided and why, how to run it, and what's left for next steps.

---

## What this system does



- **order-service** lets you create an order, update its status, and cancel it. Every order belongs to a tenant (a business/merchant), and tenants can never see each other's data.
- **notification-service** listens for order events and records a notification: a receipt when an order is created or updated, and a completion message when an order reaches a final state (completed or cancelled).
- The two services talk to each other through an **outbox pattern**, not a direct call. I'll explain why below.

---

## How the two services talk to each other (the outbox pattern)

When order-service creates or updates an order, it doesn't call notification-service directly over the network in that same moment. Instead, it writes an "event" row into its own database, in the **same transaction** as the order itself. A background job (running every 5 seconds) reads unsent events and pushes them to notification-service over REST.

**Why I did it this way, not a direct call:**
If order-service called notification-service directly and notification-service was slow or down, the order creation could fail too, even though the order itself succeeded. That's a real coupling problem. With the outbox approach, the order always saves successfully on its own, and the notification gets delivered whenever notification-service is ready, even if that's a few seconds (or minutes) later. If the first delivery attempt fails, it just gets retried on the next cycle, up to 5 times.

**Why I didn't use Kafka or a message broker:**
A real Kafka setup needs its own infrastructure (broker, topic config, consumer groups) and that's a lot of extra moving parts for the scope of this assessment. The outbox + retry approach gives the same core guarantee — never lose an event, never block the caller — without that overhead. This is also the same approach I discussed live with Neerav in our interview when he asked what happens if a message broker goes down: store the event with the order, retry delivery later.

---

## Multi-tenancy, how I made sure tenants can't see each other's data

This was the most important thing to get right, since it was called out specifically as something being evaluated at every layer.

**How a request says which tenant it belongs to:**
Every request must include a header: `X-Tenant-Id: tenant-a`. A filter checks for this header before anything else runs. If it's missing, the request is rejected immediately with a 400 error.

**Why a header and not a real login/token system:**
In a real production app, you'd never trust a header like this, anyone could fake it and read someone else's data. You'd get the tenant ID from a verified login token instead. I used a plain header here on purpose, to keep the focus on the actual tenant-isolation logic without building a whole authentication system that wasn't asked for. The header acts as a stand-in for a verified tenant identity, no matter how that identity is actually obtained.

**How the tenant ID is carried through the request, without passing it manually everywhere:**
I used a Java 21 feature called **ScopedValue** instead of the older, more common approach (ThreadLocal). Once the filter reads the tenant ID from the header, it's automatically available anywhere in that request's code, and it's automatically cleaned up when the request finishes, no manual cleanup code needed, and no risk of one request accidentally leaking its tenant ID into another.

**How the database enforces it:**
Every single database query that fetches an order goes through methods that require a tenant ID as part of the query, there is no method anywhere in the code that fetches an order without one. I deliberately avoided using the easy/default methods Spring gives you for free (like a plain "find by ID") because those don't know about tenants at all, and someone could accidentally use them and leak data across tenants.

**How I proved this actually works, not just that it's written correctly:**
I wrote an automated test that creates an order for tenant A, then deliberately tries to change its tenant to tenant B and save it. The test confirms the database still says tenant A afterward, proving the tenant can never be changed once an order is created, at the actual database level, not just in the Java code.

---

## Order lifecycle and business rules

An order can be in one of four states: `CREATED`, `CONFIRMED`, `COMPLETED`, `CANCELLED`.

Allowed moves:
- CREATED → CONFIRMED
- CREATED → CANCELLED
- CONFIRMED → COMPLETED
- CONFIRMED → CANCELLED

`COMPLETED` and `CANCELLED` are final, nothing can change after that. I put these rules directly on the status itself (as a method you can call, like "can this status move to that status") instead of writing a bunch of if-statements inside the order service. That way, if some other part of the app ever needs to check whether a move is valid, there's exactly one place that knows the rule, it doesn't have to be copied.

Trying an invalid move (like going from `COMPLETED` back to `CREATED`) returns a clear 409 error explaining exactly why.

---

## Why some specific design choices were made

**UUIDs instead of auto-incrementing numbers for IDs:**
Sequential numbers leak information (a competitor could guess how many orders you've processed by watching the numbers go up). UUIDs avoid that, and they're safer if data ever needs to move between environments.

**Each service has its own database:**
order-service and notification-service never read each other's tables directly. They only communicate through the outbox/event mechanism. This is a basic rule of real microservices: if one service can directly read another's database, they're not really separate services anymore.

**Why the notification "send" is just a database record:**
The assessment says notifications don't need to connect to a real email/SMS provider, just keep a record of what would have been sent. So `recordOrderEvent` in notification-service is the entire "send," no external API calls, just a clean, structured row saved to its own table.

**Why I added health checks in docker-compose:**
Without them, a service can try to connect to its database before the database has actually finished starting up, and crash on the very first run. The health check makes everything wait until Postgres says "I'm ready" before starting the dependent service.

---

## Testing

I wrote two kinds of tests, on purpose, because they catch different things:

1. **Unit tests** (`OrderServiceTest`): these test the business logic by itself, using fake/mock versions of the database. They check things like: does creating an order generate the right event, does an invalid status change get rejected, does cancelling an order actually work.

2. **Integration tests** (`OrderRepositoryTest`): these run against a real, temporary database (H2, in-memory) to prove the tenant-scoped queries actually work at the SQL level, not just that the Java code looks right. The most important one of these deliberately tries to break tenant isolation by changing an order's tenant after the fact, and confirms the database blocks it.

**A real bug the tests caught, while building this:**
My first version of the tenant-immutability test passed, but for the wrong reason. Hibernate was returning a cached, in-memory copy of the object instead of actually reading fresh from the database, which would have hidden a real bug if one had existed. I added `entityManager.clear()` to force a genuine database read, and only then could I be sure the protection was real and not just a coincidence of caching.

---

## How to run it

You need Docker and Docker Compose installed. Then, from the project root:

```bash
docker-compose up
```

This pulls the pre-built images from Docker Hub and starts everything, both services and both databases. No manual setup steps needed.

- order-service runs on `http://localhost:8081`
- notification-service runs on `http://localhost:8082`

### Example requests

**Create an order:**
```bash
curl -X POST http://localhost:8081/api/orders \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: tenant-a" \
  -d '{"customerId": "cust-001", "totalAmount": 49.99}'
```

**Update an order's status:**
```bash
curl -X PATCH http://localhost:8081/api/orders/{orderId}/status \
  -H "Content-Type: application/json" \
  -H "X-Tenant-Id: tenant-a" \
  -d '{"status": "CONFIRMED"}'
```

**Cancel an order:**
```bash
curl -X POST http://localhost:8081/api/orders/{orderId}/cancel \
  -H "X-Tenant-Id: tenant-a"
```

**Get all orders for a tenant:**
```bash
curl http://localhost:8081/api/orders -H "X-Tenant-Id: tenant-a"
```

**Check notifications for an order:**
```bash
curl http://localhost:8082/api/notifications/order/{orderId} -H "X-Tenant-Id: tenant-a"
```

### Running the tests

```bash
cd order-service
mvn test
```

---

## Assumptions I made (since the spec didn't say)

- `totalAmount` must be a positive number, an order can't have a zero or negative total.
- `customerId` is just a plain string identifier, not validated against any external customer system.
- The four order statuses listed above are the full lifecycle, nothing else was specified, so I kept it to what the assessment explicitly asked for (receipt + completion notifications).
- Notifications don't expire or get deleted, they're a permanent record.
- An order can only be cancelled from `CREATED` or `CONFIRMED`, not after it's already `COMPLETED`.

---

## What I'd do with more time

- **Real authentication**: swap the `X-Tenant-Id` header for a verified JWT claim, extracted after login. The seam for this is already isolated, only the filter that reads the tenant ID would need to change.
- **A formal event contract** between the two services (right now it's an informal shared JSON shape), something like a shared schema with versioning, so the two services can't drift apart silently.
- **Dead-letter handling** for outbox events that fail repeatedly (right now they're just logged and skipped after 5 attempts, not stored anywhere for manual review).
- **Pagination** on the "get all orders" endpoint, for tenants with a lot of orders.
- **Hibernate `@Filter`** as an alternative to the explicit tenant-scoped repository methods I used, I chose the explicit approach because it's easier to verify by reading the code directly, but it's worth comparing against the automatic-filter approach at scale.
