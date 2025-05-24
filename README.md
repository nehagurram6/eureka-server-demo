# Spring Boot Microservices with Eureka Service Discovery - Complete Guide

## Table of Contents
1. [What is Microservices Architecture?](#what-is-microservices-architecture)
2. [Service Discovery Pattern](#service-discovery-pattern)
3. [Eureka Server Setup](#eureka-server-setup)
4. [Eureka Client Implementation](#eureka-client-implementation)
5. [Feign Client for Service Communication](#feign-client-for-service-communication)
6. [Load Balancing](#load-balancing)
7. [Configuration Deep Dive](#configuration-deep-dive)
8. [How Everything Works Together](#how-everything-works-together)

## What is Microservices Architecture?

Microservices architecture breaks down a large application into smaller, independent services that communicate over a network. Each service:
- Runs in its own process
- Has its own database
- Can be deployed independently
- Communicates via HTTP/REST APIs

**Traditional Monolith vs Microservices:**
```
Monolith:          Microservices:
┌─────────────┐    ┌──────────┐  ┌──────────┐  ┌──────────┐
│             │    │  Order   │  │ Payment  │  │ Eureka   │
│    All      │    │ Service  │  │ Service  │  │ Server   │
│  Features   │    │ (8083)   │  │ (8082)   │  │ (8761)   │
│             │    └──────────┘  └──────────┘  └──────────┘
└─────────────┘         │            │            │
                        └────────────┼────────────┘
                                     │
                              Service Discovery
```

## Service Discovery Pattern

**The Problem:** In microservices, services need to find and communicate with each other. Hard-coding IP addresses and ports is not scalable.

**The Solution:** Service Discovery pattern where:
1. Services register themselves with a **Discovery Server** (Eureka)
2. Services query the Discovery Server to find other services
3. Communication happens using **service names** instead of IP:Port

## Eureka Server Setup

### 1. Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

**What this does:** Brings in all the Netflix Eureka Server libraries needed to run a service registry.

### 2. Main Application Class

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerDemoApplication.class, args);
    }
}
```

**Annotations Explained:**
- `@SpringBootApplication`: Standard Spring Boot starter annotation
- `@EnableEurekaServer`: **Critical annotation** that tells Spring to start this application as a Eureka Server

### 3. Configuration (application.yml)

```yaml
spring:
  application:
    name: eureka-server-demo  # Name of this service
server:
  port: 8761                  # Standard Eureka Server port

eureka:
  instance:
    hostname: localhost       # Where this server runs
  client:
    register-with-eureka: false    # Don't register itself
    fetch-registry: false          # Don't fetch registry (it IS the registry)
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

**Key Configuration Explained:**
- `register-with-eureka: false`: The server doesn't register with itself
- `fetch-registry: false`: The server doesn't need to fetch registry (it maintains it)
- `defaultZone`: URL where the Eureka server is accessible

**What Eureka Server Provides:**
- Web dashboard at `http://localhost:8761`
- REST API for service registration/discovery
- Heartbeat monitoring of registered services
- Service health checks

## Eureka Client Implementation

### Payment Service (Simple Client)

#### Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### Main Class
```java
@SpringBootApplication
public class MicroservicesPaymentServiceDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MicroservicesPaymentServiceDemoApplication.class, args);
    }
}
```

**Note:** No `@EnableEurekaClient` needed in modern Spring Cloud versions. The presence of the dependency is enough.

#### Configuration
```yaml
spring:
  application:
    name: microservices-payment-service-demo  # Service identifier

server:
  port: 8082

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/  # Where to register
  instance:
    prefer-ip-address: true                       # Use IP instead of hostname
    instance-id: ${spring.application.name}:${server.port}  # Unique instance ID
```

**Configuration Explained:**
- `spring.application.name`: **Most important** - this is how other services will find this service
- `prefer-ip-address: true`: Uses IP address instead of hostname (better for containers)
- `instance-id`: Creates unique identifier like "microservices-payment-service-demo:8082"

### Order Service (Client + Feign Consumer)

#### Additional Dependencies
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**Dependencies Explained:**
- `spring-cloud-starter-openfeign`: Enables declarative HTTP client
- `spring-cloud-starter-loadbalancer`: Provides client-side load balancing

#### Main Class
```java
@SpringBootApplication
@EnableFeignClients(basePackages = "com.microservice.demo.orderservice.outbound")
public class MicroservicesOrderServiceDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(MicroservicesOrderServiceDemoApplication.class, args);
    }
}
```

**Annotations Explained:**
- `@EnableFeignClients`: Enables Feign client support
- `basePackages`: Tells Spring where to scan for Feign client interfaces

## Feign Client for Service Communication

### What is Feign?
Feign is a **declarative HTTP client** that makes writing HTTP clients easier. Instead of writing RestTemplate code, you just define an interface.

### Traditional vs Feign Approach

**Traditional RestTemplate:**
```java
@Service
public class OrderService {
    private RestTemplate restTemplate = new RestTemplate();
    
    public PaymentStatus getPaymentStatus(String orderId) {
        String url = "http://payment-service-ip:8082/api/payment/status/" + orderId;
        return restTemplate.getForObject(url, PaymentStatus.class);
    }
}
```

**Feign Client:**
```java
@FeignClient(name = "microservices-payment-service-demo")
public interface PaymentServiceClient {
    @GetMapping("/api/payment/status/{orderId}")
    PaymentStatus getPaymentStatus(@PathVariable String orderId);
}
```

### Feign Client Deep Dive

```java
@FeignClient(name = "microservices-payment-service-demo")
public interface PaymentServiceClient {
    @GetMapping("/api/payment/status/{orderId}")
    PaymentStatus getPaymentStatus(@PathVariable String orderId);
}
```

**How it works:**
1. `@FeignClient(name = "microservices-payment-service-demo")`:
    - `name` refers to the `spring.application.name` of the target service
    - Feign will use Eureka to resolve this name to actual IP:Port
2. `@GetMapping`: Standard Spring MVC annotation for HTTP GET
3. `@PathVariable`: Maps method parameter to URL path variable

**Behind the scenes:**
1. Spring creates a proxy implementation of this interface
2. When `getPaymentStatus()` is called, Feign:
    - Queries Eureka for services named "microservices-payment-service-demo"
    - Gets list of available instances
    - Uses load balancer to pick one instance
    - Makes HTTP call to that instance
    - Deserializes response to PaymentStatus object

## Load Balancing

### What is Load Balancing?
When multiple instances of a service are running, load balancing distributes requests across all instances.

```
Order Service
     │
     ▼
Load Balancer
     │
     ├─► Payment Service Instance 1 (192.168.1.10:8082)
     ├─► Payment Service Instance 2 (192.168.1.11:8082)
     └─► Payment Service Instance 3 (192.168.1.12:8082)
```

### Client-Side Load Balancing
Spring Cloud uses **client-side load balancing**, meaning the Order Service (client) decides which Payment Service instance to call.

### Load Balancer Configuration
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: false  # Disable caching for immediate discovery updates
```

**Default Algorithm:** Round-robin (requests are distributed evenly across instances)

## Configuration Deep Dive

### Eureka Client Configuration

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/  # Eureka server location
    fetch-registry: true                          # Download service registry
    register-with-eureka: true                    # Register this service
    registry-fetch-interval-seconds: 5            # How often to fetch updates
  instance:
    prefer-ip-address: true                       # Use IP instead of hostname
    instance-id: ${spring.application.name}:${server.port}  # Unique ID
    hostname: ${spring.application.name}          # Hostname for this instance
```

**Configuration Explained:**

#### Client Settings:
- `fetch-registry: true`: Download list of all services from Eureka
- `register-with-eureka: true`: Register this service with Eureka
- `registry-fetch-interval-seconds: 5`: Check for updates every 5 seconds

#### Instance Settings:
- `prefer-ip-address: true`: Use 192.168.1.10 instead of "my-computer.local"
- `instance-id`: Creates unique identifier for this instance
- `hostname`: How this service identifies itself

### Spring Cloud Discovery

```yaml
spring:
  cloud:
    discovery:
      enabled: true  # Enable service discovery
```

### Logging Configuration

```yaml
logging:
  level:
    com.netflix.eureka: DEBUG                              # Eureka client logs
    org.springframework.cloud.openfeign: DEBUG            # Feign client logs  
    org.springframework.cloud.loadbalancer: DEBUG         # Load balancer logs
    com.microservice.demo.orderservice.outbound.PaymentServiceClient: DEBUG  # Specific Feign client
```

**Why Debug Logging?**
- See service registration/deregistration
- Monitor heartbeats
- Debug load balancing decisions
- Troubleshoot Feign client calls

## How Everything Works Together

### 1. Service Startup Sequence

```
1. Start Eureka Server (8761)
   └─► Eureka registry is empty

2. Start Payment Service (8082)
   ├─► Registers with Eureka as "microservices-payment-service-demo"
   ├─► Sends heartbeat every 30 seconds
   └─► Eureka registry now has 1 service

3. Start Order Service (8083)
   ├─► Registers with Eureka as "microservices-order-service-demo"
   ├─► Downloads service registry (knows about Payment Service)
   └─► Eureka registry now has 2 services
```

### 2. Service Communication Flow

```
1. User calls: GET http://localhost:8083/api/order/ORDER001/status

2. Order Service Controller receives request

3. Order Service calls PaymentServiceClient.getPaymentStatus("ORDER001")

4. Feign Client Process:
   ├─► Queries Eureka: "Give me instances of 'microservices-payment-service-demo'"
   ├─► Eureka returns: [192.168.1.10:8082]
   ├─► Load Balancer picks: 192.168.1.10:8082
   └─► Makes HTTP call: GET http://192.168.1.10:8082/api/payment/status/ORDER001

5. Payment Service processes request and returns PaymentStatus

6. Order Service combines Order data + Payment data and returns response
```

### 3. Fault Tolerance

**What happens if Payment Service is down?**

```java
try {
    PaymentStatus paymentStatus = paymentServiceClient.getPaymentStatus(orderId);
    orderStatus.setPaymentStatus(paymentStatus);
} catch (Exception e) {
    // Fallback: Return order with "SERVICE_UNAVAILABLE" payment status
    PaymentStatus fallback = new PaymentStatus();
    fallback.setStatus("SERVICE_UNAVAILABLE");
    orderStatus.setPaymentStatus(fallback);
}
```

### 4. Multiple Instance Scenario

If you start 3 Payment Service instances:
```
Payment Service Instance 1: localhost:8082
Payment Service Instance 2: localhost:8084  
Payment Service Instance 3: localhost:8086
```

Eureka registry shows:
```
MICROSERVICES-PAYMENT-SERVICE-DEMO:
├─► microservices-payment-service-demo:8082 (UP)
├─► microservices-payment-service-demo:8084 (UP)
└─► microservices-payment-service-demo:8086 (UP)
```

Load balancer distributes requests:
```
Request 1 → Instance 1 (8082)
Request 2 → Instance 2 (8084)  
Request 3 → Instance 3 (8086)
Request 4 → Instance 1 (8082)  # Round-robin back to first
```

## Key Benefits of This Architecture

### 1. **Dynamic Service Discovery**
- No hard-coded IP addresses
- Services can move between servers
- Auto-discovery of new service instances

### 2. **Load Balancing**
- Automatic distribution of requests
- Better performance and reliability
- Fault tolerance (if one instance fails, others handle traffic)

### 3. **Scalability**
- Easy to add more service instances
- Horizontal scaling without code changes
- Independent scaling of different services

### 4. **Maintainability**
- Clean separation of concerns
- Easy to understand service boundaries
- Simplified testing and deployment

## Common Patterns and Best Practices

### 1. Service Naming Convention
```yaml
spring:
  application:
    name: company-domain-service-environment
    # Examples:
    # myapp-order-service-prod
    # myapp-payment-service-dev
```

### 2. Health Checks
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: always
```

### 3. Circuit Breaker Pattern
```java
@FeignClient(name = "payment-service", fallback = PaymentServiceFallback.class)
public interface PaymentServiceClient {
    @GetMapping("/api/payment/status/{orderId}")
    PaymentStatus getPaymentStatus(@PathVariable String orderId);
}
```

This architecture provides a solid foundation for building scalable, maintainable microservices using Spring Boot and Netflix Eureka.