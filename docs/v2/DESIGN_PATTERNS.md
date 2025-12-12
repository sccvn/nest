# NestJS Design Patterns

This document details the 12 primary design patterns used throughout the NestJS framework and how they contribute to its architecture.

## Table of Contents

1. [Dependency Injection Pattern](#1-dependency-injection-pattern)
2. [Factory Pattern](#2-factory-pattern)
3. [Adapter Pattern](#3-adapter-pattern)
4. [Module Pattern](#4-module-pattern)
5. [Middleware Chain Pattern](#5-middleware-chain-pattern-chain-of-responsibility)
6. [Strategy Pattern](#6-strategy-pattern)
7. [Observer Pattern](#7-observer-pattern)
8. [Decorator Pattern](#8-decorator-pattern-gof)
9. [Lazy Initialization Pattern](#9-lazy-initialization-pattern)
10. [Context Object Pattern](#10-context-object-pattern)
11. [Singleton Pattern](#11-singleton-pattern)
12. [Chain of Responsibility](#12-chain-of-responsibility)

---

## 1. Dependency Injection Pattern

### Overview
Dependency Injection (DI) is the core pattern of NestJS. It enables loose coupling between components by injecting dependencies rather than creating them internally.

### Implementation in NestJS

**Constructor Injection** (Primary Method)
```typescript
@Injectable()
export class UserService {
  constructor(
    private readonly databaseService: DatabaseService,
    private readonly configService: ConfigService
  ) {}

  async getUser(id: string) {
    const config = this.configService.get('db');
    return this.databaseService.query(config);
  }
}
```

**Property Injection** (Less Common)
```typescript
@Injectable()
export class UserService {
  @Inject(DatabaseService)
  private db: DatabaseService;

  @Inject('CONFIG')
  private config: any;
}
```

**Custom Tokens**
```typescript
providers: [
  UserService,
  { provide: 'DATABASE_URL', useValue: 'postgres://...' },
  { provide: DatabaseService, useClass: PostgresDatabase }
]
```

### Benefits
- **Loose Coupling**: Services don't create their own dependencies
- **Testability**: Easy to inject mock implementations
- **Reusability**: Services can be used in different contexts
- **Maintainability**: Dependencies are explicit and documented
- **Flexibility**: Easy to swap implementations

### How NestJS Implements DI

```typescript
// The Injector class handles dependency resolution
class Injector {
  resolveConstructorParams(target: Type): any[] {
    // Get type metadata from decorators
    const paramTypes = Reflect.getMetadata('design:paramtypes', target);

    // Resolve each dependency recursively
    return paramTypes.map((type, index) => {
      return this.resolveDependency(type, index);
    });
  }

  resolveDependency(token: any, index: number): any {
    // Look up in container
    const wrapper = this.container.get(token);

    // Create instance based on scope
    return wrapper.getInstance();
  }
}
```

---

## 2. Factory Pattern

### Overview
Factory Pattern creates objects without specifying exact classes. NestJS providers implement this pattern for flexible instance creation.

### Implementation in NestJS

**Class Factory** (Most Common)
```typescript
@Injectable()
export class DatabaseService {
  // Encapsulates creation logic
}

providers: [DatabaseService]
```

**Function Factory**
```typescript
providers: [
  {
    provide: 'DATABASE',
    useFactory: (config: ConfigService) => {
      return new Database(config.getDatabaseUrl());
    },
    inject: [ConfigService]
  }
]
```

**Async Factory**
```typescript
providers: [
  {
    provide: 'DATABASE',
    useFactory: async (config: ConfigService) => {
      const db = new Database(config.getDatabaseUrl());
      await db.connect();
      return db;
    },
    inject: [ConfigService]
  }
]
```

**Value Factory**
```typescript
providers: [
  {
    provide: 'API_KEY',
    useValue: 'secret-key-12345'
  }
]
```

### Benefits
- **Separation of Concerns**: Creation logic separated from usage
- **Flexibility**: Different creation strategies for different contexts
- **Testability**: Easy to provide mock factories in tests
- **Dynamic Creation**: Can create instances based on configuration

### Real-World Example

```typescript
// Database factory
{
  provide: 'DATABASE_CONNECTION',
  useFactory: async (configService: ConfigService) => {
    const config = configService.getDatabaseConfig();

    if (config.type === 'postgres') {
      return new PostgresConnection(config);
    } else if (config.type === 'mongodb') {
      return new MongoConnection(config);
    } else {
      throw new Error('Unknown database type');
    }
  },
  inject: [ConfigService]
}
```

---

## 3. Adapter Pattern

### Overview
Adapter Pattern allows different implementations to share a common interface. NestJS uses this for HTTP framework adaptation.

### Implementation in NestJS

**HTTP Adapters**
```typescript
// Common interface for all adapters
interface HttpServer {
  use(middleware: Function): void;
  listen(port: number): void;
  close(): void;
}

// Express Adapter
class ExpressAdapter implements HttpServer {
  constructor(private express: Express) {}

  use(middleware) {
    this.express.use(middleware);
  }

  listen(port) {
    return this.express.listen(port);
  }
}

// Fastify Adapter
class FastifyAdapter implements HttpServer {
  constructor(private fastify: Fastify) {}

  use(middleware) {
    this.fastify.use(middleware);
  }

  listen(port) {
    return this.fastify.listen(port);
  }
}
```

**Usage**
```typescript
// Choose adapter at creation time
const app = await NestFactory.create(AppModule, new ExpressAdapter());
// OR
const app = await NestFactory.create(AppModule, new FastifyAdapter());

// Core remains unchanged regardless of adapter
```

### Benefits
- **Framework Agnostic**: Core logic independent of HTTP framework
- **Easy Switching**: Change frameworks with one line of code
- **Extensibility**: Easy to add new framework adapters
- **Testing**: Can create mock adapters for testing

---

## 4. Module Pattern

### Overview
Module Pattern encapsulates related functionality and manages visibility through exports. It's fundamental to NestJS organization.

### Implementation in NestJS

**Basic Module**
```typescript
@Module({
  imports: [DatabaseModule, ConfigModule],
  providers: [UserService, UserRepository],
  controllers: [UserController],
  exports: [UserService]  // What this module provides
})
export class UserModule {}
```

**Module Encapsulation**
```typescript
// UserService is exported - available to other modules
export UserService

// UserRepository is NOT exported - only internal use
// → Other modules cannot inject UserRepository directly
```

**Module Nesting**
```typescript
@Module({
  imports: [
    DatabaseModule,
    ConfigModule.register({ env: 'development' })
  ]
})
export class AppModule {}

@Module({
  providers: [UserService],
  exports: [UserService]
})
export class UserModule {}
```

### Benefits
- **Encapsulation**: Hide implementation details
- **Clear Boundaries**: Explicit module contracts
- **Reusability**: Modules can be imported multiple times
- **Testability**: Can mock entire modules in tests
- **Organization**: Logical grouping of related code

---

## 5. Middleware Chain Pattern (Chain of Responsibility)

### Overview
Request passes through a chain of middleware, each can process or pass to next. Also called Chain of Responsibility.

### Implementation in NestJS

**Request Flow**
```
Request
  ↓
Middleware 1 → pass to next
  ↓
Middleware 2 → pass to next
  ↓
Guard → check condition
  ↓
Pipe → transform/validate
  ↓
Handler
  ↓
Interceptor → post-process
  ↓
Response
```

**Custom Middleware**
```typescript
@Injectable()
export class LoggingMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`${req.method} ${req.url}`);
    next();  // Pass to next middleware
  }
}

@Module({})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggingMiddleware)
      .forRoutes(UserController);
  }
}
```

**Guard Chain**
```typescript
@UseGuards(AuthGuard, RoleGuard)
@Controller('/admin')
export class AdminController {
  @Post('/users')
  create() {}
}

// Execution: AuthGuard → RoleGuard → Handler
```

### Benefits
- **Sequential Processing**: Clear request flow
- **Composability**: Chain multiple handlers
- **Flexibility**: Add/remove handlers dynamically
- **Separation**: Each handler has single responsibility

---

## 6. Strategy Pattern

### Overview
Strategy Pattern encapsulates algorithms as interchangeable objects. NestJS uses this for exception filters and pipes.

### Implementation in NestJS

**Exception Filters (Strategy)**
```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      message: exception.message,
      timestamp: new Date()
    });
  }
}

@Catch(DatabaseException)
export class DatabaseExceptionFilter implements ExceptionFilter {
  catch(exception: DatabaseException, host: ArgumentsHost) {
    // Different handling strategy for database errors
    const response = host.switchToHttp().getResponse();
    response.status(500).json({
      error: 'Database error occurred'
    });
  }
}

// Usage: Apply different strategies based on exception type
@UseFilters(HttpExceptionFilter, DatabaseExceptionFilter)
@Controller('/users')
export class UserController {}
```

**Pipes (Strategy)**
```typescript
// Validation strategy
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    if (!value) throw new BadRequestException('Missing value');
    return value;
  }
}

// Parsing strategy
@Injectable()
export class ParseIntPipe implements PipeTransform {
  transform(value: string) {
    const int = parseInt(value);
    if (isNaN(int)) throw new BadRequestException('Not a number');
    return int;
  }
}

// Usage: Different strategies for different data
@Post('/users/:id')
update(@Param('id', ParseIntPipe) id: number) {}
```

### Benefits
- **Runtime Selection**: Choose strategy at runtime
- **Extensibility**: Easy to add new strategies
- **Encapsulation**: Each strategy self-contained
- **Testability**: Easy to test strategies in isolation

---

## 7. Observer Pattern

### Overview
Observer Pattern notifies multiple objects about state changes. RxJS Observables implement this in NestJS.

### Implementation in NestJS

**RxJS Observables in Handlers**
```typescript
@Controller('/users')
export class UserController {
  @Get()
  findAll(): Observable<User[]> {
    return this.userService.getAllUsers();
  }

  @Get(':id')
  findOne(@Param('id') id: string): Observable<User> {
    return this.userService.getUserById(id);
  }
}
```

**Observable Chaining**
```typescript
@Injectable()
export class UserService {
  getUserById(id: string): Observable<User> {
    return this.http.get(`/users/${id}`).pipe(
      map(response => response.data),
      catchError(error => throwError(() => new NotFoundException()))
    );
  }
}
```

**Interceptor with Observables**
```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    return next.handle().pipe(
      tap(() => console.log('After...')),
      catchError(error => {
        console.error('Error!', error);
        throw error;
      })
    );
  }
}
```

### Benefits
- **Reactive Programming**: Compose async operations
- **Operator Chaining**: Use RxJS operators for transformations
- **Error Handling**: Built-in error handling with catchError
- **Composition**: Chain multiple observables together

---

## 8. Decorator Pattern (GoF)

### Overview
Decorator Pattern dynamically adds behavior to objects. TypeScript decorators (not GoF pattern) provide framework extensibility.

### Implementation in NestJS

**Guard Decorator**
```typescript
@UseGuards(AuthGuard)
@Controller('/users')
export class UserController {
  @Post()
  create(@Body() dto: CreateUserDto) {}
}

// Guards applied at class and method level
@UseGuards(AuthGuard, RoleGuard)
@Post(':id')
update(@Param('id') id: string) {}
```

**Pipe Decorator**
```typescript
@Post('/users')
create(
  @Body(new ValidationPipe()) dto: CreateUserDto,
  @Param('id', ParseIntPipe) id: number
) {
  // Pipes transform/validate parameters
}
```

**Interceptor Decorator**
```typescript
@UseInterceptors(LoggingInterceptor, CacheInterceptor)
@Controller('/users')
export class UserController {
  @Get()
  findAll() {}
}
```

**Custom Decorator**
```typescript
export const RequireRole = (role: string) => {
  return applyDecorators(
    SetMetadata('roles', [role]),
    UseGuards(RoleGuard)
  );
};

@RequireRole('admin')
@Delete(':id')
delete(@Param('id') id: string) {}
```

### Benefits
- **Composition**: Combine multiple enhancers
- **Reusability**: Create custom decorators for common patterns
- **Readability**: Declarative syntax is clear
- **Separation**: Concerns separated from handler

---

## 9. Lazy Initialization Pattern

### Overview
Lazy Initialization defers object creation until first use. NestJS uses this for providers.

### Implementation in NestJS

**Default Behavior**
```typescript
@Module({
  providers: [
    ExpensiveService  // Created on first injection, not at startup
  ]
})
export class AppModule {}
```

**Lifecycle**
```
Startup
  ↓
Module scanning (metadata only)
  ↓
First injection of ExpensiveService
  ↓
Injector creates instance (expensive operation)
  ↓
Instance cached (if SINGLETON)
  ↓
Return instance
```

**Eager Initialization** (If Needed)
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Eagerly initialize expensive service
  const service = app.get(ExpensiveService);

  await app.listen(3000);
}
```

### Benefits
- **Startup Performance**: Faster application boot
- **Resource Efficiency**: Only create what's used
- **Circular Dependency Handling**: Deferred creation helps avoid cycles
- **Conditional Loading**: Create only if actually needed

---

## 10. Context Object Pattern

### Overview
Context Object encapsulates all data related to a request. Provides abstraction over different protocol contexts.

### Implementation in NestJS

**ExecutionContext**
```typescript
interface ExecutionContext {
  getClass(): Type<any>
  getHandler(): Function
  getArgs(): any[]
  switchToHttp(): HttpArgumentsHost
  switchToWs(): WsArgumentsHost
  switchToRpc(): RpcArgumentsHost
}
```

**Protocol-Agnostic Guard**
```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    if (context.getType() === 'http') {
      const request = context.switchToHttp().getRequest();
      return !!request.user;
    } else if (context.getType() === 'ws') {
      const socket = context.switchToWs().getClient();
      return !!socket.auth;
    }
    return false;
  }
}
```

**Usage Across Protocols**
```typescript
// Same guard works for HTTP
@UseGuards(AuthGuard)
@Controller('/users')
export class UserController {}

// Same guard works for WebSocket
@UseGuards(AuthGuard)
@WebSocketGateway()
export class ChatGateway {}

// Same guard works for gRPC
@UseGuards(AuthGuard)
@Controller()
export class RpcController {}
```

### Benefits
- **Protocol Abstraction**: Same code for HTTP, WebSocket, gRPC
- **Type Safety**: Compile-time checking of context access
- **Flexibility**: Easy to support new protocols
- **Decoupling**: Handlers don't know about transport layer

---

## 11. Singleton Pattern

### Overview
Singleton Pattern ensures only one instance of a class exists. It's the default scope in NestJS.

### Implementation in NestJS

**Default Singleton**
```typescript
@Injectable()
export class DatabaseService {
  private connection: Connection;

  connect() {
    // Only called once
    this.connection = createConnection();
  }

  getConnection(): Connection {
    return this.connection;  // Always return same instance
  }
}

// Module
@Module({
  providers: [DatabaseService]
})
export class DatabaseModule {}

// All modules get same instance
@Module({
  imports: [DatabaseModule],
  providers: [UserService]
})
export class UserModule {
  constructor(private db: DatabaseService) {
    // This is the SAME instance everywhere
  }
}
```

**Single Instance Across Application**
```typescript
// User Module
constructor(private db: DatabaseService) {}

// Product Module
constructor(private db: DatabaseService) {}

// Auth Module
constructor(private db: DatabaseService) {}

// All three DatabaseService instances are the SAME object
```

### Benefits
- **Memory Efficiency**: One instance shared globally
- **Consistency**: Shared state across application
- **Performance**: No repeated initialization
- **Shared State**: Stateful services maintain state

### Comparison with Other Scopes

```typescript
// SINGLETON - one instance, shared globally
@Injectable()
class CacheService {}

// TRANSIENT - new instance for each injection
@Injectable({ scope: Scope.TRANSIENT })
class RequestIdGenerator {}

// REQUEST - one instance per HTTP request
@Injectable({ scope: Scope.REQUEST })
class RequestContext {}
```

---

## 12. Chain of Responsibility

### Overview
Chain of Responsibility passes request through chain of handlers until handled. Each handler can process or pass.

### Implementation in NestJS

**Complete Request Chain**
```
HTTP Request
  ↓ (1)
Middleware Chain (express middleware)
  ↓ (2)
Guard.canActivate() - Can block request
  ↓ (3)
Pipe.transform() - Validate and transform data
  ↓ (4)
Interceptor.intercept() before - Pre-processing
  ↓ (5)
Route Handler - Business logic
  ↓ (6)
Interceptor.intercept() after - Post-processing
  ↓ (7)
Exception Filter - Error handling
  ↓ (8)
Response sent to client
```

**Example Chain**
```typescript
// 1. Middleware
@Injectable()
export class LoggingMiddleware implements NestMiddleware {
  use(req, res, next) {
    console.log('Middleware: Logging request');
    next();
  }
}

// 2. Guard
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext) {
    console.log('Guard: Checking authentication');
    return true;
  }
}

// 3. Pipe
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value) {
    console.log('Pipe: Validating data');
    return value;
  }
}

// 4. Interceptor (before)
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context, next) {
    console.log('Interceptor: Before handler');
    return next.handle().pipe(
      tap(() => console.log('Interceptor: After handler'))
    );
  }
}

// 5. Handler
@Post('/users')
@UseGuards(AuthGuard)
@UsePipes(ValidationPipe)
@UseInterceptors(LoggingInterceptor)
create(@Body() dto: CreateUserDto) {
  console.log('Handler: Processing request');
  return { id: 1, ...dto };
}

// Output order:
// Middleware: Logging request
// Guard: Checking authentication
// Pipe: Validating data
// Interceptor: Before handler
// Handler: Processing request
// Interceptor: After handler
```

### Benefits
- **Separation of Concerns**: Each handler has single responsibility
- **Composability**: Chain multiple handlers
- **Flexibility**: Add/remove handlers dynamically
- **Maintainability**: Clear request flow

---

## Summary

NestJS leverages these 12 design patterns to create a flexible, maintainable, and scalable framework:

| Pattern | Purpose | Example |
|---------|---------|---------|
| Dependency Injection | Loose coupling | Constructor injection of services |
| Factory | Flexible object creation | Provider system |
| Adapter | Framework abstraction | HTTP adapters (Express, Fastify) |
| Module | Code organization | @Module decorator |
| Middleware Chain | Request processing | Middleware → Guard → Pipe → Handler |
| Strategy | Interchangeable algorithms | Exception filters, pipes |
| Observer | Reactive programming | RxJS Observables |
| Decorator | Dynamic behavior | @UseGuards, @UsePipes, @UseInterceptors |
| Lazy Initialization | Performance | Provider creation on first use |
| Context Object | Protocol abstraction | ExecutionContext |
| Singleton | Shared state | Default provider scope |
| Chain of Responsibility | Sequential processing | Complete request pipeline |

These patterns work together to create a cohesive, professional framework that balances power with simplicity.
