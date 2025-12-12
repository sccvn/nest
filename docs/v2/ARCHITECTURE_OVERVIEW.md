# NestJS Architecture Overview

## Table of Contents

1. [Core Architecture](#core-architecture)
2. [Module System](#module-system)
3. [Dependency Injection](#dependency-injection)
4. [Request Processing Pipeline](#request-processing-pipeline)
5. [Design Patterns](#design-patterns)
6. [Key Subsystems](#key-subsystems)
7. [Technology Stack](#technology-stack)
8. [Application Initialization](#application-initialization)

---

## Core Architecture

### High-Level Architecture

NestJS is a progressive, production-grade Node.js framework that leverages TypeScript decorators, metadata, and modern JavaScript features to build scalable server-side applications. The architecture is built on three core pillars:

1. **Metadata-Driven Design**: Uses TypeScript decorators with `reflect-metadata` for declarative configuration
2. **Dependency Injection**: Built-in IoC container for managing dependencies and providers
3. **Modular Organization**: Module-based architecture for code organization and reusability

### Core Components

```
┌─────────────────────────────────────────────────────┐
│              NestApplication                         │
│  (Orchestrates all subsystems)                       │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  │   Router     │  │  Middleware  │  │  Interceptor │
│  │   System     │  │   Pipeline   │  │    Chain     │
│  └──────────────┘  └──────────────┘  └──────────────┘
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  │   Guards     │  │    Pipes     │  │   Filters    │
│  │  (Auth)      │  │  (Validation)│  │ (Errors)     │
│  └──────────────┘  └──────────────┘  └──────────────┘
│                                                      │
│  ┌──────────────────────────────────────────────────┐
│  │    Dependency Injection Container (DI)           │
│  │  - Module Registry                               │
│  │  - Provider Management                           │
│  │  - Instance Creation & Caching                   │
│  └──────────────────────────────────────────────────┘
│                                                      │
│  ┌──────────────────────────────────────────────────┐
│  │    HTTP Adapter (Express / Fastify)              │
│  │    WebSocket Adapter (Socket.io / WS)            │
│  │    Microservice Adapters (TCP, NATS, etc)        │
│  └──────────────────────────────────────────────────┘
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Module System

### Module Concept

Modules are the basic organizational unit in NestJS. Each module:
- Encapsulates a feature or domain
- Has explicit imports and exports
- Manages its own set of providers
- Can be shared globally or kept private

### Module Definition

```typescript
@Module({
  imports: [DatabaseModule, ConfigModule],
  providers: [UserService, UserRepository],
  controllers: [UserController],
  exports: [UserService]  // Available to other modules
})
export class UserModule {}
```

### Module Metadata

```
MODULE_METADATA = {
  IMPORTS: 'imports'        // Other modules to import
  PROVIDERS: 'providers'    // Services/factories to provide
  CONTROLLERS: 'controllers'// Route handlers
  EXPORTS: 'exports'        // What this module exports
}
```

### Module Resolution

When `DependenciesScanner.scan(RootModule)` is called:

1. **Graph Traversal**: Recursively scan all imported modules
2. **Provider Registration**: Register all providers from each module
3. **Distance Calculation**: Calculate module distance for dependency resolution
4. **Global Module Marking**: Identify global modules that bypass scope
5. **Metadata Extraction**: Extract and cache all metadata

### Module Hierarchy

```
AppModule (root)
├── UserModule
│   ├── DatabaseModule (global)
│   └── ConfigModule (global)
├── ProductModule
│   ├── DatabaseModule (shared from User)
│   └── CacheModule
├── AuthModule
│   └── JwtModule
└── SharedModule (global)
    ├── LoggerService
    └── ValidationService
```

---

## Dependency Injection

### Provider Types

#### 1. **Class Provider** (Most Common)

```typescript
@Injectable()
export class UserService {
  constructor(private db: DatabaseService) {}
}

// In Module
providers: [UserService, DatabaseService]
```

#### 2. **Factory Provider**

```typescript
providers: [
  {
    provide: 'DATABASE',
    useFactory: (configService: ConfigService) => {
      return createDatabaseConnection(configService.getDbConfig());
    },
    inject: [ConfigService]
  }
]
```

#### 3. **Value Provider**

```typescript
providers: [
  {
    provide: 'CONFIG',
    useValue: {
      apiKey: 'secret',
      apiUrl: 'https://api.example.com'
    }
  }
]
```

#### 4. **Existing Provider** (Alias)

```typescript
providers: [
  Service,
  { provide: 'ServiceAlias', useExisting: Service }
]
```

### Dependency Resolution Process

1. **Identify Target Class**: Get the class to be instantiated
2. **Extract Constructor Params**: Read constructor parameter types via metadata
3. **Resolve Each Param Recursively**:
   - Look up in container by token (type or string)
   - Recursively resolve dependencies
   - Apply scope rules (SINGLETON/TRANSIENT/REQUEST)
4. **Create Instance**: Call constructor with resolved dependencies
5. **Inject Properties**: Apply `@Inject()` decorated properties
6. **Cache Instance**: Store in appropriate cache based on scope

### Scopes

#### SINGLETON (Default)
- One instance for entire application lifetime
- Cached in `InstanceWrapper.singleton`
- Shared across all requests
- **Use for**: Services, repositories, utilities, configuration

#### TRANSIENT
- New instance created for every injection
- Never cached
- Each time provider is requested, new instance created
- **Use for**: Request-specific utilities, temporary objects

#### REQUEST
- One instance per HTTP request
- Cached in `InstanceWrapper.request` (WeakMap per request)
- Automatically cleaned up after response
- **Use for**: Request context, user information, per-request state

### Scope Selection

```typescript
// SINGLETON (default)
@Injectable()
class DatabaseService {}

// TRANSIENT
@Injectable({ scope: Scope.TRANSIENT })
class RequestLogger {}

// REQUEST
@Injectable({ scope: Scope.REQUEST })
class UserContext {}
```

---

## Request Processing Pipeline

### HTTP Request Lifecycle

```
1. HTTP Request
   ↓
2. Express/Fastify Adapter receives request
   ↓
3. Middleware Pipeline
   - Application middleware (global)
   - Route-specific middleware
   ↓
4. Router Matching
   - Find matching controller method
   - Extract route metadata
   ↓
5. Guards Execution
   - Check authorization
   - Can block request
   - Scoped: method → class → module → global
   ↓
6. Parameter Transformation & Validation
   - Extract: @Param, @Query, @Body, @Headers, etc.
   - Apply pipes for transformation/validation
   ↓
7. Interceptor Before Hook
   - Pre-processing logic
   - Can modify request
   ↓
8. Route Handler Execution
   - Call controller method
   - Execute business logic
   ↓
9. Interceptor After Hook
   - Post-processing logic
   - Transform response
   - Add metadata
   ↓
10. Exception Handling
    - Try-catch wraps entire flow
    - Matches exception filters
    - Sends error response if exception
    ↓
11. Response Sent
    - Status code and body sent to client
```

### Execution Context

The `ExecutionContext` provides abstraction over different protocols:

```typescript
interface ExecutionContext {
  getClass(): Type<any>
  getHandler(): Function
  getArgs(): any[]
  getArgByIndex(index: number): any
  switchToHttp(): HttpArgumentsHost
  switchToWs(): WsArgumentsHost
  switchToRpc(): RpcArgumentsHost
}
```

This allows the same guard/pipe/interceptor to work across HTTP, WebSocket, and gRPC protocols.

---

## Design Patterns

### 1. **Dependency Injection Pattern**
Loose coupling through constructor injection and IoC container. Enables testing and modularity.

### 2. **Decorator Pattern** (GoF)
Uses TypeScript decorators to add functionality:
- `@UseGuards()` - Authorization
- `@UsePipes()` - Validation
- `@UseInterceptors()` - AOP
- `@UseFilters()` - Error handling

### 3. **Adapter Pattern**
HTTP adapters (Express, Fastify) implement common interface, allowing framework-agnostic core.

### 4. **Factory Pattern**
Provider system creates instances using various factory methods (class, function, value).

### 5. **Middleware Chain Pattern** (Chain of Responsibility)
Request passes through chain of middleware, guards, pipes, handlers, and filters.

### 6. **Strategy Pattern**
Different exception handling strategies via exception filters. Interchangeable error handling.

### 7. **Observer Pattern** (Reactive)
RxJS Observable integration enables reactive programming. Interceptors use RxJS operators.

### 8. **Lazy Initialization Pattern**
Providers created on first injection, not at startup. Reduces boot time.

### 9. **Singleton Pattern**
Default scope for providers. Single instance shared across application.

### 10. **Module Pattern**
Modules create encapsulation boundaries with explicit imports/exports.

---

## Key Subsystems

### Router System
- **RouterExplorer**: Discovers route metadata from controller decorators
- **RoutesResolver**: Registers routes with HTTP adapter
- **RouteParamsFactory**: Extracts parameter values from requests
- **RouterProxy**: Applies guards, pipes, and interceptors

### Exception Handling
- Catches exceptions in try-catch wrapper
- Matches exception filters by type
- Applies filter transformations
- Sends error response

### WebSocket Support
- **GatewayMetadataExplorer**: Finds @WebSocketGateway decorated classes
- **WebSocketsController**: Routes messages to handlers
- Supports Socket.io and native WebSocket adapters

### Microservices
- **Hybrid Applications**: Can serve both HTTP and microservice simultaneously
- **Transport Layer**: TCP, UDP, MQTT, NATS, RabbitMQ, Kafka, gRPC
- **Message Patterns**: Request-Response and Event-Based

### Testing Support
- **TestingModule**: Override modules/providers for testing
- **Test.createTestingModule()**: Compile testing versions
- Mock services and get instances for testing

---

## Technology Stack

### Core Dependencies
- **TypeScript**: 5.9.3 - Type safety
- **reflect-metadata**: Metadata annotation system
- **rxjs**: Reactive programming (v7.x)
- **iterare**: Iterator utilities
- **path-to-regexp**: Route pattern matching

### Platform Adapters
- **@nestjs/platform-express**: Express.js integration (v5.x)
- **@nestjs/platform-fastify**: Fastify framework (v5.x)
- **@nestjs/platform-socket.io**: Socket.io adapter (v4.x)
- **@nestjs/platform-ws**: Native WebSocket adapter

### Optional Integrations
- **@nestjs/mongoose**: MongoDB ODM
- **@nestjs/typeorm**: SQL ORM
- **@nestjs/graphql**: GraphQL support
- **@nestjs/apollo**: GraphQL server
- **kafkajs**: Kafka transport
- **amqplib**: RabbitMQ transport
- **mqtt**: MQTT protocol
- **nats**: NATS messaging

### Validation & Serialization
- **class-validator**: Data validation (v0.14.x)
- **class-transformer**: DTO transformation (v0.5.x)

---

## Application Initialization

### Startup Sequence

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.setGlobalPrefix('api');
  app.useGlobalPipes(new ValidationPipe());
  app.useGlobalInterceptors(new LoggingInterceptor());

  await app.listen(3000);
  console.log('Server running on port 3000');
}

bootstrap();
```

### Initialization Steps

1. **NestFactory.create(AppModule)**
   - Create NestContainer
   - Create ApplicationConfig
   - Create DependenciesScanner
   - Scan module tree and register providers

2. **DependenciesScanner.scan()**
   - Traverse module dependencies recursively
   - Extract metadata from decorators
   - Register modules in container
   - Calculate module distances

3. **InstanceLoader.createInstancesOfProviders()**
   - Instantiate all singleton providers
   - Resolve dependencies using Injector
   - Call onModuleInit() lifecycle hooks
   - Cache instances in InstanceWrapper

4. **RouterResolver.explore()**
   - Find all controllers
   - Extract route metadata
   - Register routes with HTTP adapter
   - Bind guards, pipes, interceptors

5. **app.listen(port)**
   - Start HTTP server
   - Call onApplicationBootstrap() hooks
   - Server ready to accept connections

### Shutdown Sequence

On application shutdown (SIGTERM, SIGINT):

1. Stop accepting new requests
2. Wait for existing requests to complete
3. Call onModuleDestroy() lifecycle hooks
4. Call onApplicationShutdown() hooks
5. Close HTTP server
6. Close database connections
7. Exit process

---

## Key Files & Entry Points

| File | Purpose |
|------|---------|
| `/packages/core/nest-factory.ts` | Application bootstrap |
| `/packages/core/nest-application.ts` | Main application class |
| `/packages/core/scanner.ts` | Metadata discovery |
| `/packages/core/injector/container.ts` | DI container |
| `/packages/core/injector/module.ts` | Module implementation |
| `/packages/core/router/routes-resolver.ts` | HTTP routing |
| `/packages/common/decorators/` | All decorators |
| `/packages/microservices/` | Event-driven patterns |
| `/packages/websockets/` | WebSocket support |
| `/packages/testing/` | Testing utilities |

---

## Summary

NestJS provides a comprehensive, well-architected framework that:

- **Leverages TypeScript**: Full type safety and decorators for configuration
- **Implements Strong DI**: Built-in IoC container for loose coupling
- **Enables Modularity**: Module system for code organization
- **Supports Multiple Protocols**: HTTP, WebSocket, Microservices, gRPC
- **Provides AOP**: Guards, pipes, interceptors, filters for cross-cutting concerns
- **Ensures Scalability**: Designed for large, complex applications
- **Maintains Clean Code**: Separation of concerns, testability, readability

The architecture demonstrates mature engineering principles and is production-ready for building scalable Node.js applications.
