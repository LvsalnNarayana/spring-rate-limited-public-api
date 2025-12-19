
# Spring-Rate-Limited-Public-Api

## Overview

This project is a **multi-microservice demonstration** of **rate limiting** for **public-facing APIs** using **Bucket4j** (token bucket algorithm) in **Spring Boot 3.x**. It showcases how to protect APIs from abuse, ensure fair usage, and prevent overload by enforcing per-client (user/IP/API key) request limits.

The focus is on **production-grade rate limiting**: distributed state with Redis, multiple strategies (fixed/per-second/refill), graceful 429 responses, and observability.

## Real-World Scenario (Simulated)

In public APIs like **Twitter (X)**, **GitHub**, or **Stripe**:
- Free-tier users are limited (e.g., 100 requests/hour) to prevent abuse and DoS attacks.
- Paid tiers get higher limits.
- Limits are enforced per user, IP, or API key.
- Exceeding limits returns `429 Too Many Requests` with `Retry-After` header.
- Distributed systems require shared rate limit state across instances.

We simulate a public search/analytics API where clients are rate-limited per API key or IP, using Redis for distributed bucket state to handle horizontal scaling.

## Microservices Involved

| Service                  | Responsibility                                                                 | Port  |
|--------------------------|--------------------------------------------------------------------------------|-------|
| **redis**                | Distributed bucket store (Redis)                                                | 6379  |
| **eureka-server**        | Service discovery (Netflix Eureka)                                             | 8761  |
| **public-gateway**       | Entry point: rate limiting filters, routes to downstream                       | 8080  |
| **search-service**       | Handles search queries (simulated load)                                        | 8081  |
| **analytics-service**    | Provides usage analytics (heavier endpoint)                                    | 8082  |

Rate limiting applied at gateway level (edge) with shared Redis state.

## Tech Stack

- Spring Boot 3.x
- Bucket4j (core + Redis extension)
- Spring Cloud Gateway (for filters) or WebFilter (Servlet)
- Lettuce (Redis client)
- Spring Boot Actuator + Micrometer (rate limit metrics)
- Spring Cloud Netflix Eureka
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --save 60 1 --loglevel warning

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  public-gateway:
    build: ./public-gateway
    depends_on:
      - redis
      - eureka-server
    ports:
      - "8080:8080"

  search-service:
    build: ./search-service
    depends_on:
      - eureka-server
    ports:
      - "8081:8081"

  analytics-service:
    build: ./analytics-service
    depends_on:
      - eureka-server
    ports:
      - "8082:8082"
```

Run with: `docker-compose up --build`

## Rate Limiting Strategy

| Strategy                   | Implementation Details                                                  |
|----------------------------|-------------------------------------------------------------------------|
| **Token Bucket**           | Bucket4j with capacity + refill rate                                    |
| **Key Resolver**           | By API key header, IP, or composite                                     |
| **Distributed State**      | Redis-backed buckets (shared across instances)                          |
| **Multiple Limits**        | Free tier: 100/hour, Premium: 1000/minute, IP fallback                  |
| **Graceful Response**      | HTTP 429 with `Retry-After` and custom message                           |
| **Bypass**                 | Admin keys or internal services exempt                                  |
| **Metrics**                | Bucket probes, refused requests via Micrometer                          |

## Key Features

- Distributed rate limiting with Redis
- Multiple bucket configurations (per endpoint/tier)
- Custom key resolvers (header, IP, user)
- Graceful 429 responses with retry info
- Metrics: refused requests, available tokens
- Configurable via application.yml
- Horizontal scaling ready (shared Redis)
- Bypass for internal/admin traffic

## Expected Endpoints

### Public Gateway (`http://localhost:8080`)

| Method | Endpoint                        | Description                                      | Limit Example |
|--------|---------------------------------|--------------------------------------------------|---------------|
| GET    | `/api/search`                   | Public search (light)                            | 100/hour per key |
| GET    | `/api/analytics/report`         | Heavy analytics report                           | 10/minute per key |
| GET    | `/api/admin/internal`           | Internal endpoint (bypasses limit)               | Unlimited     |

**Headers**:
- `X-API-Key: your-key` (required)
- Response on limit exceed: `429 Too Many Requests`, `Retry-After: 3600`

### Actuator (Gateway)

| Endpoint                                      | Description                                      |
|-----------------------------------------------|--------------------------------------------------|
| GET `/actuator/metrics/bucket4j.refused`      | Total refused requests                           |
| GET `/actuator/metrics/bucket4j.available`    | Available tokens gauge                           |

## Architecture Overview

```
Clients (with X-API-Key or IP)
   ↓
Public Gateway → Rate Limit Filter (Bucket4j + Redis)
   ↓ (allowed)
Route to Search / Analytics Services
   ↓ (refused)
429 Too Many Requests + Retry-After
   ↓
Metrics → Prometheus / Actuator
```

**Rate Limit Flow**:
1. Request → extract key (API key or IP)
2. Resolve bucket from Redis
3. Try consume token → success → forward
4. No token → 429 response
5. Refill tokens over time

## How to Run

1. Clone repository
2. Start Docker: `docker-compose up --build`
3. Access Eureka: `http://localhost:8761`
4. Test with key:
   ```bash
   curl -H "X-API-Key: free-tier-key" http://localhost:8080/api/search
   ```
5. Hammer endpoint → after limit, 429 responses
6. Use different key → separate limit
7. Check metrics → refused count increases

## Testing Rate Limiting

1. Normal usage → 200 OK
2. Exceed limit → 429 + Retry-After
3. Wait refill period → works again
4. Multiple instances → shared Redis maintains limit
5. Admin key → unlimited
6. No key → fallback to IP limiting

## Skills Demonstrated

- Distributed rate limiting with Bucket4j + Redis
- Token bucket algorithm implementation
- Custom key resolvers and multiple limits
- Graceful degradation and client feedback
- Metrics and observability of rate limiting
- Protecting public APIs from abuse
- Horizontal scaling considerations

## Future Extensions

- Tiered limits with user database
- Dynamic limits via config server
- Sliding window algorithm
- Dashboard for rate limit usage
- Integration with API management (Kong, Apigee)
- Leaky bucket alternative
- Geo-based limiting

