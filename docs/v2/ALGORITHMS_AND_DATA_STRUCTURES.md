# NestJS Algorithms & Data Structures

This document details the key algorithms and data structures used internally by NestJS for dependency injection, routing, and request processing.

## Table of Contents

1. [Module Dependency Resolution](#1-module-dependency-resolution)
2. [Provider Dependency Graph](#2-provider-dependency-graph)
3. [Route Matching Algorithm](#3-route-matching-algorithm)
4. [Provider Instance Caching](#4-provider-instance-caching)
5. [Middleware Chain Execution](#5-middleware-chain-execution)
6. [Exception Filter Matching](#6-exception-filter-matching)
7. [Metadata Storage & Retrieval](#7-metadata-storage--retrieval)

---

## 1. Module Dependency Resolution

### Problem
Given a root module with imports, recursively load all modules while:
- Avoiding infinite loops (circular dependencies)
- Calculating distance for each module
- Registering global modules
- Extracting metadata from each module

### Algorithm: Depth-First Search (DFS)

```
DependenciesScanner.scan(rootModule)
  ↓
ModuleCompiler.compile(rootModule)
  ↓
visitModule(rootModule, distance=0)
  ↓
For each import in module:
  ├─ Check if already visited
  ├─ If circular → skip
  ├─ If not visited → recursively call visitModule(import, distance+1)
  └─ Register module in container
  ↓
Mark global modules
```

### Pseudocode

```typescript
// Simplified implementation
class DependenciesScanner {
  visited = new Set<Type>();
  modules = new Map<Type, Module>();

  scan(root: Type) {
    this.visitModule(root, 0);
  }

  private visitModule(type: Type, distance: number) {
    if (this.visited.has(type)) {
      return;  // Circular dependency - skip
    }

    this.visited.add(type);

    // Extract @Module metadata
    const metadata = Reflect.getMetadata('nest:module', type);

    // Create Module instance
    const module = new Module(type, metadata);
    module.distance = distance;

    // Save module
    this.modules.set(type, module);

    // Recursively visit imports
    for (const imported of metadata.imports || []) {
      this.visitModule(imported, distance + 1);
    }

    // Mark as global if needed
    if (metadata.global) {
      this.globalModules.add(module);
    }
  }
}
```

### Complexity Analysis

- **Time Complexity**: O(M + E)
  - M = number of modules
  - E = number of import edges
  - Each module visited once
  - Each import traversed once

- **Space Complexity**: O(M)
  - Visited set stores M modules
  - Call stack depth = max nesting

### Example

```
AppModule (distance: 0)
├── DatabaseModule (distance: 1)
├── AuthModule (distance: 1)
│   ├── JwtModule (distance: 2)
│   └── ConfigModule (distance: 2)
└── UserModule (distance: 1)
    ├── DatabaseModule (already visited - skip)
    └── ConfigModule (already visited - skip)

Result: 5 unique modules scanned
```

---

## 2. Provider Dependency Graph

### Problem
For each provider, identify its dependencies and instantiate in correct order:
- Detect circular dependencies between providers
- Resolve transitive dependencies
- Handle optional dependencies
- Execute in dependency order

### Algorithm: Topological Sort

```
For each provider:
  ↓
Extract constructor dependencies via metadata
  ↓
Build dependency graph:
  provider → dependencies[]
  ↓
Topological sort (DFS):
  - Visit node if not visited
  - Recursively visit all dependencies first
  - Add node to sorted list when all dependencies processed
  ↓
Create instances in sorted order
```

### Pseudocode

```typescript
class InstanceLoader {
  dependencyGraph = new Map<string, string[]>();
  sorted = [];
  visiting = new Set<string>();
  visited = new Set<string>();

  createInstancesOfProviders() {
    // Build dependency graph
    for (const provider of this.providers) {
      const deps = this.getProviderDependencies(provider);
      this.dependencyGraph.set(provider.token, deps);
    }

    // Topological sort
    for (const provider of this.providers) {
      if (!this.visited.has(provider.token)) {
        this.topologicalSort(provider.token);
      }
    }

    // Create instances in sorted order
    for (const token of this.sorted) {
      const wrapper = this.container.get(token);
      const instance = this.instantiate(wrapper);
      wrapper.setInstance(instance);
    }
  }

  private topologicalSort(token: string) {
    if (this.visited.has(token)) return;

    if (this.visiting.has(token)) {
      throw new CircularDependencyException(token);
    }

    this.visiting.add(token);

    // Visit all dependencies first
    const dependencies = this.dependencyGraph.get(token) || [];
    for (const dep of dependencies) {
      this.topologicalSort(dep);
    }

    this.visiting.delete(token);
    this.visited.add(token);
    this.sorted.push(token);
  }

  private getProviderDependencies(provider): string[] {
    if (provider.useFactory) {
      return provider.inject || [];  // Dependencies specified in inject
    }

    // For class providers, extract from constructor
    const paramTypes = Reflect.getMetadata('design:paramtypes', provider.useClass);
    return paramTypes.map(type => this.container.getTokenForType(type));
  }
}
```

### Complexity Analysis

- **Time Complexity**: O(P + D)
  - P = number of providers
  - D = number of dependencies (edges)
  - Each provider processed once
  - Each dependency traversed once

- **Space Complexity**: O(P + D)
  - Graph storage: O(P + D)
  - Call stack: O(P) in worst case (chain)
  - Visited/visiting sets: O(P)

### Example

```
UserService depends on: [DatabaseService, ConfigService]
AuthService depends on: [ConfigService]
ConfigService depends on: []
DatabaseService depends on: []

Topological order:
1. ConfigService
2. DatabaseService
3. AuthService
4. UserService

(Dependencies created before dependents)
```

---

## 3. Route Matching Algorithm

### Problem
Given HTTP request path, find matching controller method:
- Support path parameters: `/users/:id`
- Support wildcards: `/files/*`
- Support regex patterns
- Match fastest way possible

### Algorithm: Path-to-Regexp Matching

NestJS uses the `path-to-regexp` library for route matching:

```typescript
const route = pathToRegexp('/users/:id/posts/:postId');
const match = route.exec('/users/123/posts/456');
// match = ['123', '456']
```

### Route Registration Process

```
For each controller:
  ├─ Extract @Controller path
  │
  └─ For each route handler method:
       ├─ Extract @Get/@Post/@Put/@Delete path
       ├─ Combine paths: `/users` + `/profile`
       ├─ Extract @Param/@Query/@Body metadata
       ├─ Create route handler wrapper
       └─ Register with HTTP adapter
```

### Pseudocode

```typescript
class RoutesResolver {
  private routes = [];

  explore(controller: Type) {
    const controllerPath = this.getControllerPath(controller);

    // Get all route methods
    const methods = this.getRouteMethods(controller);

    for (const method of methods) {
      const methodPath = this.getMethodPath(method);
      const httpMethod = this.getHttpMethod(method);
      const fullPath = this.combinePaths(controllerPath, methodPath);

      // Compile path pattern
      const pathRegex = pathToRegexp(fullPath);

      // Extract parameter metadata
      const params = this.extractParameterMetadata(method);

      // Create handler
      const handler = this.createHandler(controller, method, params);

      // Register route
      this.registerRoute(httpMethod, fullPath, handler, pathRegex);
    }
  }

  private registerRoute(method, path, handler, regex) {
    this.routes.push({
      method,
      path,
      regex,
      handler
    });

    // Register with Express/Fastify
    this.httpAdapter[method.toLowerCase()](path, handler);
  }

  match(request): RouteMatch | null {
    for (const route of this.routes) {
      if (request.method !== route.method) continue;

      const match = route.regex.exec(request.path);
      if (match) {
        return {
          handler: route.handler,
          params: match.slice(1)
        };
      }
    }
    return null;
  }
}
```

### Complexity Analysis

- **Registration**: O(R * M)
  - R = number of routes
  - M = path complexity (usually small constant)
  - Path compilation is O(path length)

- **Matching**: O(R)
  - Linear search through routes
  - Each regex test is O(path length)
  - In practice, very fast due to HTTP adapter optimizations

### Optimization

Express/Fastify use internal trie structures for route matching:

```
Routes:
  GET /users
  GET /users/:id
  POST /users
  GET /users/:id/posts
  GET /posts

Trie structure:
  /
  ├─ users
  │  ├─ [GET] handler1
  │  ├─ [POST] handler2
  │  ├─ :id
  │  │  ├─ [GET] handler3
  │  │  └─ /posts
  │  │     └─ [GET] handler4
  └─ posts
     └─ [GET] handler5

Matching /users/123/posts:
  / → users → :id → /posts [GET] → handler4
```

---

## 4. Provider Instance Caching

### Problem
Store provider instances based on scope:
- SINGLETON: cache forever
- TRANSIENT: never cache
- REQUEST: cache per request context

### Data Structure: InstanceWrapper

```typescript
class InstanceWrapper<T> {
  private singleton: T | null = null;           // SINGLETON cache
  private transients: T[] = [];                 // Pool (for debugging)
  private requestCache = new WeakMap<Req, T>(); // REQUEST cache

  getInstance(context?: ExecutionContext): T {
    switch (this.scope) {
      case Scope.SINGLETON:
        if (!this.singleton) {
          this.singleton = this.create();
        }
        return this.singleton;

      case Scope.TRANSIENT:
        return this.create();

      case Scope.REQUEST:
        if (context) {
          const request = context.switchToHttp().getRequest();
          if (!this.requestCache.has(request)) {
            this.requestCache.set(request, this.create());
          }
          return this.requestCache.get(request);
        }
        return this.create();
    }
  }

  private create(): T {
    // Resolve dependencies
    const deps = this.resolveDependencies();

    // Call constructor
    return new this.type(...deps);
  }
}
```

### Complexity Analysis

- **SINGLETON**:
  - First call: O(D) where D = dependency depth
  - Subsequent calls: O(1)
  - Memory: O(1) - single instance

- **TRANSIENT**:
  - Every call: O(D)
  - Memory: O(N) where N = calls made

- **REQUEST**:
  - First request call: O(D)
  - Same request subsequent calls: O(1)
  - Memory: O(R) where R = concurrent requests

### WeakMap Benefits

Using `WeakMap` for REQUEST scope:
- Automatic garbage collection when request ends
- No manual cleanup needed
- No memory leaks from forgotten requests
- O(1) lookup time

---

## 5. Middleware Chain Execution

### Problem
Execute multiple middleware functions in sequence:
- Pass control through chain
- Allow middleware to modify request/response
- Support async middleware
- Handle errors

### Algorithm: Express-Style Middleware Chain

```
Middleware Chain:
  middleware1 → middleware2 → middleware3 → next()

Execution:
  middleware1(req, res, next)
    ↓
    next() calls middleware2(req, res, next)
      ↓
      next() calls middleware3(req, res, next)
        ↓
        next() calls actual handler
```

### Pseudocode

```typescript
class MiddlewareConsumer {
  private middlewares: NestMiddleware[] = [];

  use(middleware: NestMiddleware): void {
    this.middlewares.push(middleware);
  }

  async executeChain(req, res, finalHandler): Promise<void> {
    let index = -1;

    const next = async () => {
      index++;

      if (index < this.middlewares.length) {
        const middleware = this.middlewares[index];
        // Execute middleware with next callback
        await new Promise((resolve) => {
          middleware.use(req, res, (err) => {
            if (err) throw err;
            resolve(undefined);
          });
        });
        await next();  // Recursive call to next middleware
      } else if (index === this.middlewares.length) {
        // All middleware executed, call final handler
        return await finalHandler(req, res);
      }
    };

    await next();
  }
}
```

### Complexity Analysis

- **Time**: O(M * H)
  - M = number of middleware
  - H = time per middleware
  - Sequential execution

- **Space**: O(M)
  - Call stack depth = M (can hit stack limit with many middleware)

### Optimization: Flatten Chain

For better performance, compile middleware chain once:

```typescript
const compiledChain = [
  middleware1,
  middleware2,
  middleware3,
  finalHandler
].reduce(
  (next, handler) => () => handler(req, res, next),
  () => { /* end */ }
);

await compiledChain();  // Single call, no recursion
```

---

## 6. Exception Filter Matching

### Problem
When exception thrown, find matching filter based on:
- Exception type
- Scope (method → class → module → global)
- Multiple filters can match

### Algorithm: Scope-Based Matching with Inheritance

```
When exception thrown:
  ↓
Find filters in scope order:
  1. Method-level @UseFilters
  2. Class-level @UseFilters
  3. Module-level providers (@Catch)
  4. Global filters app.useGlobalFilters()
  ↓
For each filter, check if matches exception type:
  exception instanceof FilterCatchType
  ↓
Execute first matching filter
```

### Pseudocode

```typescript
class ExceptionFiltersContext {
  findFilterForException(exception, context): ExceptionFilter | null {
    const methodLevelFilters = this.getMethodFilters(context);
    const classLevelFilters = this.getClassFilters(context);
    const moduleLevelFilters = this.getModuleFilters(context);
    const globalFilters = this.getGlobalFilters();

    // Check scopes in order
    const allFilters = [
      ...methodLevelFilters,
      ...classLevelFilters,
      ...moduleLevelFilters,
      ...globalFilters
    ];

    // Find first matching filter
    for (const filter of allFilters) {
      const catchTypes = Reflect.getMetadata('catch', filter);

      for (const catchType of catchTypes) {
        if (exception instanceof catchType) {
          return filter;
        }
      }
    }

    return null;  // No matching filter found
  }
}
```

### Example Execution

```typescript
// Define filters
@Catch(HttpException)
class HttpExceptionFilter {}

@Catch(DatabaseException)
class DatabaseExceptionFilter {}

// Apply globally
app.useGlobalFilters(
  new HttpExceptionFilter(),
  new DatabaseExceptionFilter()
);

// Exception thrown in handler
throw new HttpException('Not found', 404);

// Filter matching:
// 1. Method-level filters: (none)
// 2. Class-level filters: (none)
// 3. Module-level filters: (none)
// 4. Global filters: [HttpExceptionFilter, DatabaseExceptionFilter]
// 5. Match HttpException against HttpExceptionFilter → MATCH
// 6. Execute HttpExceptionFilter.catch()
```

### Complexity Analysis

- **Time**: O(S * F)
  - S = number of scopes checked (usually 4)
  - F = number of filters per scope
  - In practice, usually very small

- **Space**: O(F)
  - Store all filters in memory

---

## 7. Metadata Storage & Retrieval

### Problem
Store and retrieve metadata attached to classes/methods/parameters:
- Efficient lookup
- Support class inheritance
- Handle decorators at different levels

### Algorithm: Reflect-Metadata with Prototypes

```
Class Definition:
  @Module()
  @Injectable()
  class MyService {
    @Get('/users')
    getUsers(
      @Param('id') id: number,
      @Query() query: QueryDto
    ) {}
  }

Metadata Storage:
  MyService.__metadata = {
    'nest:module': { imports, providers, ... },
    'nest:injectable': { scope: 'singleton' }
  }

  MyService.prototype.getUsers.__metadata = {
    'nest:route': { path: '/users', method: 'GET' }
  }

  MyService.prototype.getUsers.__parameterMetadata = {
    '0': { type: 'param', name: 'id' },
    '1': { type: 'query' }
  }
```

### Storage Structure

```typescript
// Simplified reflect-metadata implementation
const metadataStore = new WeakMap();

function defineMetadata(metadataKey, metadataValue, target, propertyKey?) {
  let metadata = metadataStore.get(target);
  if (!metadata) {
    metadata = {};
    metadataStore.set(target, metadata);
  }

  if (propertyKey) {
    if (!metadata[propertyKey]) {
      metadata[propertyKey] = {};
    }
    metadata[propertyKey][metadataKey] = metadataValue;
  } else {
    metadata[metadataKey] = metadataValue;
  }
}

function getMetadata(metadataKey, target, propertyKey?) {
  let metadata = metadataStore.get(target);
  if (!metadata) return undefined;

  if (propertyKey) {
    return metadata[propertyKey]?.[metadataKey];
  }
  return metadata[metadataKey];
}
```

### Complexity Analysis

- **defineMetadata**: O(1) amortized
  - WeakMap insertion
  - Property setting on object

- **getMetadata**: O(1)
  - WeakMap lookup
  - Property access

### Benefits of WeakMap

- Automatic cleanup when object is garbage collected
- No memory leaks
- Private metadata (not enumerable)
- O(1) operations
- No collision with own properties

---

## Summary

| Algorithm | Purpose | Complexity | Implementation |
|-----------|---------|-----------|-----------------|
| Module DFS | Load module tree | O(M + E) | DependenciesScanner |
| Provider Topological Sort | Instantiation order | O(P + D) | InstanceLoader |
| Path-to-Regexp | Route matching | O(R) | RoutesResolver |
| Instance Caching | Scope management | O(1) lookup | InstanceWrapper |
| Middleware Chain | Sequential execution | O(M * H) | MiddlewareConsumer |
| Filter Matching | Exception handling | O(S * F) | ExceptionFiltersContext |
| Reflect-Metadata | Metadata storage | O(1) | WeakMap-based system |

These algorithms work together to provide NestJS's high performance and flexibility for building scalable applications.
