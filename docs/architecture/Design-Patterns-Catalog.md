# Design Patterns Catalog - NestJS Framework

## Overview

This document catalogs all design patterns identified in the NestJS core framework, with precise locations, implementation details, and rationale for their use.

---

## 1. Creational Patterns

### 1.1 Factory Pattern

**Location**: `packages/core/nest-factory.ts`

**Implementation**: `NestFactoryStatic` class

```plantuml
@startuml Pattern_Factory_Detail
skinparam classAttributeIconSize 0

class NestFactoryStatic <<Factory>> {
    -{static} logger: Logger
    -{static} abortOnError: boolean
    -{static} autoFlushLogs: boolean
    --
    +{static} create<T extends INestApplication>(module: Type, options?): Promise<T>
    +{static} createMicroservice<T extends object>(module: Type, options?): Promise<INestMicroservice>
    +{static} createApplicationContext(module: Type, options?): Promise<INestApplicationContext>
    --
    -{static} initialize(module, container, graphInspector, config): Promise<void>
    -{static} createNestInstance<T>(instance: T): T
    -{static} createHttpAdapter(httpServer?: any): AbstractHttpAdapter
    -{static} createGraphInspector(appOptions, container): GraphInspector
}

note right of NestFactoryStatic
  Factory pattern centralizes object creation
  with different creation strategies:
  - create() → HTTP application
  - createMicroservice() → Microservice
  - createApplicationContext() → Standalone context
end note

@enduml
```

**Rationale**:
- Encapsulates complex object creation logic
- Provides multiple entry points for different application types
- Handles initialization sequence consistently

**Usage Example**:
```typescript
// HTTP Application
const app = await NestFactory.create(AppModule);

// Microservice
const microservice = await NestFactory.createMicroservice(AppModule, {
  transport: Transport.TCP,
});

// Standalone Context
const context = await NestFactory.createApplicationContext(AppModule);
```

---

### 1.2 Singleton Pattern

**Location**: `packages/core/injector/instance-wrapper.ts`

**Implementation**: Default provider scope (Scope.DEFAULT)

```plantuml
@startuml Pattern_Singleton
skinparam classAttributeIconSize 0

class InstanceWrapper<T> {
    +name: string
    +scope: Scope = DEFAULT
    -values: WeakMap<ContextId, InstancePerContext<T>>
    --
    +getInstanceByContextId(contextId): InstancePerContext<T>
    +setInstanceByContextId(contextId, value): void
    --
    // For DEFAULT scope, same instance returned
    // for all contextIds (singleton behavior)
}

class STATIC_CONTEXT <<Singleton Key>> {
    +id: 1
}

note right of InstanceWrapper
  When scope = DEFAULT:
  - Instance created once during bootstrap
  - Same instance shared across all requests
  - Stored with STATIC_CONTEXT key
end note

@enduml
```

**Rationale**:
- Reduces memory footprint
- Ensures single source of truth for services
- Improves performance by avoiding repeated instantiation

**Code Reference** (`packages/core/injector/injector.ts:108-125`):
```typescript
public loadPrototype<T>(
  { token }: InstanceWrapper<T>,
  collection: Map<InstanceToken, InstanceWrapper<T>>,
  contextId = STATIC_CONTEXT,
): void {
  // Singleton instances use STATIC_CONTEXT
  const wrapper = collection.get(token);
  // ...creates prototype once for DEFAULT scope
}
```

---

### 1.3 Builder Pattern

**Location**: `packages/core/middleware/builder.ts`

**Implementation**: `MiddlewareBuilder` class

```plantuml
@startuml Pattern_Builder
skinparam classAttributeIconSize 0

class MiddlewareBuilder <<Builder>> {
    -middlewareCollection: Set<MiddlewareConfiguration>
    -excludedRoutes: ExcludeRouteMetadata[]
    --
    +apply(...middleware: Type<NestMiddleware>[]): MiddlewareConfigProxy
    +exclude(...routes: RouteInfo[]): this
    +build(): MiddlewareConfiguration[]
}

interface MiddlewareConfigProxy {
    +forRoutes(...routes: (string | Type | RouteInfo)[]): MiddlewareConsumer
    +exclude(...routes: RouteInfo[]): this
}

class MiddlewareConsumer {
    -builder: MiddlewareBuilder
    +apply(...middleware): MiddlewareConfigProxy
}

MiddlewareBuilder --> MiddlewareConfigProxy : returns
MiddlewareConfigProxy --> MiddlewareConsumer : returns

note bottom of MiddlewareBuilder
  Builder pattern allows fluent configuration:
  
  consumer
    .apply(LoggerMiddleware, AuthMiddleware)
    .exclude({ path: 'health', method: RequestMethod.GET })
    .forRoutes(UsersController);
end note

@enduml
```

**Rationale**:
- Provides fluent API for middleware configuration
- Separates construction from representation
- Enables step-by-step configuration

**Usage Example**:
```typescript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .exclude({ path: 'cats', method: RequestMethod.GET })
      .forRoutes(CatsController);
  }
}
```

---

## 2. Structural Patterns

### 2.1 Adapter Pattern

**Location**: `packages/core/adapters/http-adapter.ts`, `packages/platform-express/`, `packages/platform-fastify/`

**Implementation**: `AbstractHttpAdapter` and platform-specific implementations

```plantuml
@startuml Pattern_Adapter_Detail
skinparam classAttributeIconSize 0

abstract class AbstractHttpAdapter<TServer = any, TRequest = any, TResponse = any> {
    #instance: any
    #httpServer: TServer
    --
    +{abstract} close(): Promise<void>
    +{abstract} listen(port: number | string, callback?): Promise<void>
    +{abstract} listen(port: number | string, hostname: string, callback?): Promise<void>
    +{abstract} get(handler: RequestHandler): void
    +{abstract} get(path: string, handler: RequestHandler): void
    +{abstract} post(handler: RequestHandler): void
    +{abstract} post(path: string, handler: RequestHandler): void
    +{abstract} use(handler: RequestHandler | ErrorHandler): void
    +{abstract} use(path: string, handler: RequestHandler | ErrorHandler): void
    --
    +{abstract} getRequestHostname(request: TRequest): string
    +{abstract} getRequestMethod(request: TRequest): string
    +{abstract} getRequestUrl(request: TRequest): string
    +{abstract} reply(response: TResponse, body: any, statusCode?: number): void
    +{abstract} status(response: TResponse, statusCode: number): void
    +{abstract} render(response: TResponse, view: string, options: any): void
}

class ExpressAdapter extends AbstractHttpAdapter {
    -readonly instance: Express
    --
    +close(): Promise<void>
    +listen(port, callback?): Promise<void>
    +get(path, handler): void
    +post(path, handler): void
    +use(handler): void
    +getRequestUrl(request): string
    +reply(response, body, statusCode?): void
}

class FastifyAdapter extends AbstractHttpAdapter {
    -readonly instance: FastifyInstance
    --
    +close(): Promise<void>
    +listen(port, callback?): Promise<void>
    +get(path, handler): void
    +post(path, handler): void
    +use(handler): void
    +getRequestUrl(request): string
    +reply(response, body, statusCode?): void
}

note right of AbstractHttpAdapter
  Adapter pattern allows NestJS to work
  with different HTTP frameworks by
  adapting their APIs to a common interface
end note

@enduml
```

**Rationale**:
- Allows framework-agnostic core
- Enables platform switching without code changes
- Isolates platform-specific quirks

**File Locations**:
- Base adapter: `packages/core/adapters/http-adapter.ts`
- Express adapter: `packages/platform-express/adapters/express-adapter.ts`
- Fastify adapter: `packages/platform-fastify/adapters/fastify-adapter.ts`

---

### 2.2 Decorator Pattern (Gang of Four)

**Location**: `packages/core/router/router-execution-context.ts`

**Implementation**: Guards, Interceptors, Pipes wrapping handler execution

```plantuml
@startuml Pattern_Decorator_GoF
skinparam classAttributeIconSize 0

interface RequestHandler {
    +handle(req, res, next): any
}

class BaseHandler implements RequestHandler {
    +handle(req, res, next): any
}

abstract class HandlerDecorator implements RequestHandler {
    #wrapped: RequestHandler
    +handle(req, res, next): any
}

class GuardDecorator extends HandlerDecorator {
    -guards: CanActivate[]
    +handle(req, res, next): any
}

class InterceptorDecorator extends HandlerDecorator {
    -interceptors: NestInterceptor[]
    +handle(req, res, next): any
}

class PipeDecorator extends HandlerDecorator {
    -pipes: PipeTransform[]
    +handle(req, res, next): any
}

BaseHandler <-- GuardDecorator : wraps
GuardDecorator <-- InterceptorDecorator : wraps
InterceptorDecorator <-- PipeDecorator : wraps

note bottom of HandlerDecorator
  Each decorator adds behavior without
  modifying the original handler:
  - Guards: Authorization check
  - Interceptors: Pre/post processing
  - Pipes: Validation/transformation
end note

@enduml
```

**Rationale**:
- Adds behavior dynamically
- Maintains single responsibility
- Enables flexible composition

**Code Reference** (`packages/core/router/router-execution-context.ts:79-174`):
```typescript
public create(
  instance: Controller,
  callback: (...args: any[]) => unknown,
  methodName: string,
  moduleKey: string,
  requestMethod: RequestMethod,
  contextId?: ContextId,
  inquirerId?: string,
): (...args: any[]) => Promise<any> {
  // Creates decorated handler chain
  const guards = this.guardsContextCreator.create(/*...*/);
  const interceptors = this.interceptorsContextCreator.create(/*...*/);
  const pipes = this.pipesContextCreator.create(/*...*/);
  // ... combines into execution pipeline
}
```

---

### 2.3 Proxy Pattern

**Location**: `packages/core/router/router-proxy.ts`

**Implementation**: `RouterProxy` class

```plantuml
@startuml Pattern_Proxy_Detail
skinparam classAttributeIconSize 0

class RouterProxy <<Proxy>> {
    +createProxy(targetCallback: Function, exceptionsHandler: ExceptionsHandler): Function
    +createExceptionLayerProxy(target: Function, exceptionsHandler: ExceptionsHandler): Function
}

class ExceptionsHandler {
    -filters: ExceptionFilter[]
    +next(exception: Error, host: ArgumentsHost): void
    +setCustomFilters(filters: ExceptionFilter[]): void
}

class ExceptionsZone {
    -{static} exceptionHandler: ExceptionHandler
    +{static} run(callback: () => void, teardown?: (err: Error) => void): void
    +{static} asyncRun<T>(callback: () => Promise<T>, teardown?: (err: Error) => void): Promise<T>
}

RouterProxy --> ExceptionsHandler : uses
RouterProxy --> ExceptionsZone : uses

note right of RouterProxy
  Proxy wraps controller methods to:
  1. Handle exceptions uniformly
  2. Run in protected exception zone
  3. Apply exception filters
end note

@enduml
```

**Rationale**:
- Adds exception handling transparently
- Controls access to actual handler
- Provides execution zone protection

**Code Reference** (`packages/core/router/router-proxy.ts:8-28`):
```typescript
public createProxy(
  targetCallback: RouterProxyCallback,
  exceptionsHandler: ExceptionsHandler,
) {
  return async <TRequest, TResponse>(
    req: TRequest,
    res: TResponse,
    next: () => void,
  ) => {
    try {
      await targetCallback(req, res, next);
    } catch (e) {
      exceptionsHandler.next(e, new ExecutionContextHost([req, res, next]));
    }
  };
}
```

---

### 2.4 Facade Pattern

**Location**: `packages/core/nest-application.ts`

**Implementation**: `NestApplication` class

```plantuml
@startuml Pattern_Facade
skinparam classAttributeIconSize 0

class NestApplication <<Facade>> {
    -middlewareModule: MiddlewareModule
    -middlewareContainer: MiddlewareContainer
    -microservicesModule: MicroservicesModule
    -socketModule: SocketModule
    -routesResolver: RoutesResolver
    --
    +listen(port): Promise<void>
    +use(middleware): this
    +enableCors(options): void
    +setGlobalPrefix(prefix): void
    +useGlobalPipes(...pipes): this
    +useGlobalFilters(...filters): this
    +useGlobalGuards(...guards): this
    +useGlobalInterceptors(...interceptors): this
    +connectMicroservice(options): INestMicroservice
    +startAllMicroservices(): Promise<void>
}

class MiddlewareModule {
    +configure()
}

class RoutesResolver {
    +resolve()
    +registerRouters()
}

class MicroservicesModule {
    +register()
}

class SocketModule {
    +register()
}

NestApplication --> MiddlewareModule
NestApplication --> RoutesResolver
NestApplication --> MicroservicesModule
NestApplication --> SocketModule

note bottom of NestApplication
  NestApplication provides a simplified
  interface to complex subsystems:
  - Middleware configuration
  - Route resolution
  - Microservices
  - WebSockets
end note

@enduml
```

**Rationale**:
- Simplifies framework usage
- Hides complex initialization
- Provides unified configuration API

---

### 2.5 Composite Pattern

**Location**: `packages/core/injector/modules-container.ts`

**Implementation**: Module tree structure

```plantuml
@startuml Pattern_Composite
skinparam classAttributeIconSize 0

interface ModuleComponent {
    +token: string
    +providers: Map<InjectionToken, InstanceWrapper>
}

class Module implements ModuleComponent {
    -_token: string
    -_imports: Set<Module>
    -_providers: Map<InjectionToken, InstanceWrapper>
    -_controllers: Map<InjectionToken, InstanceWrapper>
    -_exports: Set<InjectionToken>
    +addImport(module: Module): void
    +imports: Set<Module>
}

class ModulesContainer {
    -modules: Map<string, Module>
    +get(token: string): Module
    +set(token: string, module: Module): void
    +values(): IterableIterator<Module>
}

class NestContainer {
    -modules: ModulesContainer
    -globalModules: Set<Module>
    +addModule(metatype): Promise<ModuleRef>
    +bindGlobalsToImports(module): void
}

ModulesContainer o-- Module : contains
Module o-- Module : imports (child)
NestContainer --> ModulesContainer

note bottom of Module
  Modules form a tree structure where:
  - Each module can import other modules
  - Global modules are implicitly imported
  - Exports define what's visible to importers
end note

@enduml
```

**Rationale**:
- Represents module hierarchy
- Enables uniform treatment of modules
- Supports recursive composition

---

## 3. Behavioral Patterns

### 3.1 Chain of Responsibility Pattern

**Location**: `packages/core/middleware/middleware-module.ts`, `packages/core/guards/guards-consumer.ts`

**Implementation**: Middleware chain, Guard chain, Interceptor chain

```plantuml
@startuml Pattern_Chain_Detail
skinparam classAttributeIconSize 0

package "Middleware Chain" {
    interface NestMiddleware {
        +use(req: Request, res: Response, next: NextFunction): void
    }
    
    class MiddlewareA implements NestMiddleware {
        +use(req, res, next): void
    }
    
    class MiddlewareB implements NestMiddleware {
        +use(req, res, next): void
    }
    
    class MiddlewareC implements NestMiddleware {
        +use(req, res, next): void
    }
}

package "Guard Chain" {
    interface CanActivate {
        +canActivate(context: ExecutionContext): boolean | Promise<boolean>
    }
    
    class GuardsConsumer {
        +tryActivate(guards: CanActivate[], context: ExecutionContext): Promise<boolean>
    }
}

package "Interceptor Chain" {
    interface NestInterceptor {
        +intercept(context: ExecutionContext, next: CallHandler): Observable<any>
    }
    
    class InterceptorsConsumer {
        +intercept(interceptors: NestInterceptor[], context, next): Observable<any>
    }
}

MiddlewareA ..> MiddlewareB : next()
MiddlewareB ..> MiddlewareC : next()
GuardsConsumer ..> CanActivate : iterates
InterceptorsConsumer ..> NestInterceptor : chains

note bottom of GuardsConsumer
  Each element in chain can:
  1. Process and continue (call next)
  2. Process and short-circuit (throw/return false)
  3. Transform input/output
end note

@enduml
```

**Rationale**:
- Decouples sender from receivers
- Allows dynamic chain configuration
- Each handler has single responsibility

**Code Reference** (`packages/core/guards/guards-consumer.ts:11-28`):
```typescript
public async tryActivate(
  guards: CanActivate[],
  context: ExecutionContext,
  instance: Controller,
  callback: (...args: any[]) => any,
): Promise<boolean> {
  for (const guard of guards) {
    const result = guard.canActivate(context);
    if (await this.pickResult(result) !== true) {
      throw new ForbiddenException(FORBIDDEN_MESSAGE);
    }
  }
  return true;
}
```

---

### 3.2 Strategy Pattern

**Location**: `packages/core/injector/opaque-key-factory/`

**Implementation**: Module token generation strategies

```plantuml
@startuml Pattern_Strategy_Detail
skinparam classAttributeIconSize 0

interface ModuleOpaqueKeyFactory {
    +create(moduleId: string, moduleName: string, dynamicModuleMetadata?: Partial<DynamicModule>): string
}

class ByReferenceModuleOpaqueKeyFactory implements ModuleOpaqueKeyFactory {
    +create(moduleId, moduleName, metadata?): string
    // Returns: module reference as key
}

class DeepHashedModuleOpaqueKeyFactory implements ModuleOpaqueKeyFactory {
    -hashFn: HashFunction
    +create(moduleId, moduleName, metadata?): string
    // Returns: hash of module + metadata
}

class ModuleCompiler {
    -moduleOpaqueKeyFactory: ModuleOpaqueKeyFactory
    +compile(metatype): ModuleFactory
    +setModuleOpaqueKeyFactory(factory: ModuleOpaqueKeyFactory): void
}

ModuleCompiler --> ModuleOpaqueKeyFactory : uses

note right of ModuleOpaqueKeyFactory
  Different strategies for generating
  module tokens based on configuration:
  - ByReference: Use object reference
  - DeepHashed: Hash module + dynamic metadata
end note

@enduml
```

**Rationale**:
- Allows different module identification strategies
- Encapsulates algorithm variations
- Enables runtime strategy selection

**File Locations**:
- Interface: `packages/core/injector/opaque-key-factory/interfaces/module-opaque-key-factory.interface.ts`
- By Reference: `packages/core/injector/opaque-key-factory/by-reference-module-opaque-key-factory.ts`
- Deep Hashed: `packages/core/injector/opaque-key-factory/deep-hashed-module-opaque-key-factory.ts`

---

### 3.3 Observer Pattern

**Location**: `packages/core/hooks/`

**Implementation**: Lifecycle hooks (OnModuleInit, OnModuleDestroy, etc.)

```plantuml
@startuml Pattern_Observer_Detail
skinparam classAttributeIconSize 0

interface OnModuleInit {
    +onModuleInit(): any
}

interface OnModuleDestroy {
    +onModuleDestroy(): any
}

interface OnApplicationBootstrap {
    +onApplicationBootstrap(): any
}

interface OnApplicationShutdown {
    +onApplicationShutdown(signal?: string): any
}

interface BeforeApplicationShutdown {
    +beforeApplicationShutdown(signal?: string): any
}

class NestApplicationContext <<Subject>> {
    -container: NestContainer
    --
    #callInitHook(): Promise<void>
    #callDestroyHook(): Promise<void>
    #callBootstrapHook(): Promise<void>
    #callShutdownHook(signal?: string): Promise<void>
    #callBeforeShutdownHook(signal?: string): Promise<void>
}

class ServiceA <<Observer>> implements OnModuleInit {
    +onModuleInit(): void
}

class ServiceB <<Observer>> implements OnModuleDestroy, OnApplicationShutdown {
    +onModuleDestroy(): void
    +onApplicationShutdown(signal): void
}

NestApplicationContext --> ServiceA : notifies
NestApplicationContext --> ServiceB : notifies

note bottom of NestApplicationContext
  Application context iterates through
  all providers and calls lifecycle
  hooks in order of module distance
end note

@enduml
```

**Rationale**:
- Loose coupling between framework and services
- Enables initialization/cleanup logic
- Supports graceful shutdown

**Code Reference** (`packages/core/hooks/on-module-init.hook.ts`):
```typescript
export async function callModuleInitHook(module: Module): Promise<void> {
  const providers = module.getNonAliasProviders();
  const [_, moduleClassHost] = providers.shift();
  const instances = [...providers, moduleClassHost];
  
  await Promise.all(
    this.mapToInitPromise(instances),
  );
}
```

---

### 3.4 Template Method Pattern

**Location**: `packages/core/helpers/context-creator.ts`

**Implementation**: `ContextCreator` abstract class

```plantuml
@startuml Pattern_Template_Method
skinparam classAttributeIconSize 0

abstract class ContextCreator {
    #config: ApplicationConfig
    #container: NestContainer
    --
    +create(instance, callback, module, contextId?, inquirerId?): T[]
    #{abstract} createContext(args: unknown[], type: ContextType, host: Module): T[]
    #getGlobalMetadata<T>(contextId?): T[]
    #getInstanceByMetatype<T extends object>(metatype: Type<T>): T | undefined
}

class GuardsContextCreator extends ContextCreator {
    #createContext(args, type, host): CanActivate[]
    -getGuardInstance(guard, contextId): CanActivate
    -reflectClassMetadata(type): any[]
    -reflectMethodMetadata(callback): any[]
}

class PipesContextCreator extends ContextCreator {
    #createContext(args, type, host): PipeTransform[]
    -getPipeInstance(pipe, contextId): PipeTransform
}

class InterceptorsContextCreator extends ContextCreator {
    #createContext(args, type, host): NestInterceptor[]
    -getInterceptorInstance(interceptor, contextId): NestInterceptor
}

note bottom of ContextCreator
  Template Method defines skeleton:
  1. Get global enhancers
  2. Get class-level enhancers
  3. Get method-level enhancers
  4. Merge and return
  
  Subclasses implement createContext()
  for specific enhancer types
end note

@enduml
```

**Rationale**:
- Defines algorithm skeleton
- Allows subclasses to customize steps
- Avoids code duplication

**Code Reference** (`packages/core/helpers/context-creator.ts`):
```typescript
public abstract class ContextCreator {
  public create(
    instance: Controller,
    callback: Function,
    module: string,
    contextId?: ContextId,
    inquirerId?: string,
  ): T[] {
    const globalMetadata = this.getGlobalMetadata<T[]>(contextId);
    const classMetadata = this.reflectClassMetadata<T>(type);
    const methodMetadata = this.reflectMethodMetadata<T>(callback);
    // Template algorithm - merge all metadata
    return [...globalMetadata, ...classMetadata, ...methodMetadata];
  }
  
  protected abstract createContext(...): T[];
}
```

---

### 3.5 Command Pattern

**Location**: `packages/core/repl/` (REPL Commands)

**Implementation**: REPL native functions

```plantuml
@startuml Pattern_Command
skinparam classAttributeIconSize 0

abstract class ReplFunction <<Command>> {
    #{abstract} name: string
    #{abstract} signature: string
    #{abstract} description: string
    +{abstract} action(...args: unknown[]): any
}

class GetReplFn extends ReplFunction {
    +name = 'get'
    +signature = 'get(token)'
    +description = 'Retrieves an instance of a provider'
    +action(token: string): any
}

class DebugReplFn extends ReplFunction {
    +name = 'debug'
    +signature = 'debug(moduleCls?)'
    +description = 'Print all modules'
    +action(moduleCls?: Type): void
}

class SelectReplFn extends ReplFunction {
    +name = 'select'
    +signature = 'select(moduleCls)'
    +description = 'Navigate to a specific module'
    +action(moduleCls: Type): INestApplicationContext
}

class ResolveReplFn extends ReplFunction {
    +name = 'resolve'
    +signature = 'resolve(token, contextId?)'
    +description = 'Resolves transient or request-scoped providers'
    +action(token, contextId?): Promise<any>
}

class ReplContext <<Invoker>> {
    -nativeFunctions: Map<string, ReplFunction>
    +registerNativeFunction(fn: ReplFunction): void
    +execute(fnName: string, ...args: any[]): any
}

ReplContext --> ReplFunction : invokes

note bottom of ReplFunction
  Each REPL command encapsulates:
  - Command name and signature
  - Execution logic
  - Help description
end note

@enduml
```

**Rationale**:
- Encapsulates request as object
- Allows parameterization
- Supports undo/redo (potential)

**File Locations**:
- `packages/core/repl/native-functions/get-repl-fn.ts`
- `packages/core/repl/native-functions/debug-repl-fn.ts`
- `packages/core/repl/native-functions/select-relp-fn.ts`
- `packages/core/repl/native-functions/resolve-repl-fn.ts`

---

### 3.6 Visitor Pattern

**Location**: `packages/core/metadata-scanner.ts`

**Implementation**: `MetadataScanner` class

```plantuml
@startuml Pattern_Visitor
skinparam classAttributeIconSize 0

class MetadataScanner <<Visitor>> {
    +scanFromPrototype<T extends Injectable, R = any>(instance: T, prototype: object, callback: (name: string) => R): R[]
    +getAllMethodNames(prototype: object): IterableIterator<string>
}

class PathsExplorer {
    -metadataScanner: MetadataScanner
    +scanForPaths(instance: Controller): RoutePathProperties[]
    +exploreMethodMetadata(instance, prototype, methodName): RoutePathProperties
}

class DependenciesScanner {
    -metadataScanner: MetadataScanner
    +reflectInjectables(instance, metatype): void
    +reflectKeyMetadata(metatype, key, methodKey): any[]
}

MetadataScanner <-- PathsExplorer : uses
MetadataScanner <-- DependenciesScanner : uses

note right of MetadataScanner
  Visitor pattern allows scanning
  class prototypes and applying
  different operations (callbacks)
  to each method found
end note

@enduml
```

**Rationale**:
- Separates algorithm from object structure
- Adds operations without modifying classes
- Enables metadata extraction from decorators

**Code Reference** (`packages/core/metadata-scanner.ts:12-34`):
```typescript
public scanFromPrototype<T extends Injectable, R = any>(
  instance: T,
  prototype: object,
  callback: (name: string) => R,
): R[] {
  const methodNames = this.getAllMethodNames(prototype);
  return iterate(methodNames)
    .filter(method => !isConstructor(method) && isFunction(prototype[method]))
    .map(callback)
    .toArray();
}
```

---

## 4. Architectural Patterns

### 4.1 Dependency Injection / Inversion of Control

**Location**: `packages/core/injector/`

**Implementation**: `Injector`, `NestContainer`, `InstanceWrapper`

```plantuml
@startuml Pattern_DI
skinparam classAttributeIconSize 0

package "IoC Container" {
    class NestContainer <<Container>> {
        -modules: ModulesContainer
        +addModule(metatype): ModuleRef
        +addProvider(provider, token): void
        +getModuleByKey(key): Module
    }
    
    class Injector <<Resolver>> {
        +loadInstance(wrapper, collection, moduleRef): Promise<void>
        +resolveConstructorParams(wrapper, moduleRef, inject): Promise<any[]>
        +lookupComponent(dependencies, moduleRef, wrapper): Promise<InstanceWrapper>
        +instantiateClass(instances, wrapper, targetMetatype): Promise<T>
    }
    
    class InstanceWrapper<T> <<Instance>> {
        +token: InjectionToken
        +metatype: Type<T>
        +instance: T
        +scope: Scope
        +inject: InjectorDependency[]
        +getInstanceByContextId(contextId): InstancePerContext<T>
    }
}

package "Decorators" {
    annotation Injectable {
        +scope?: Scope
    }
    
    annotation Inject {
        +token: InjectionToken
    }
    
    annotation Optional {
    }
}

NestContainer --> Injector : uses
Injector --> InstanceWrapper : resolves

note bottom of Injector
  DI/IoC Flow:
  1. @Injectable() marks class for DI
  2. @Inject() specifies dependency token
  3. Container stores provider metadata
  4. Injector resolves dependencies at runtime
end note

@enduml
```

**Core Implementation Files**:
- Container: `packages/core/injector/container.ts`
- Injector: `packages/core/injector/injector.ts`
- Instance Wrapper: `packages/core/injector/instance-wrapper.ts`
- Module: `packages/core/injector/module.ts`

---

### 4.2 Module Pattern

**Location**: `packages/common/decorators/modules/module.decorator.ts`

**Implementation**: `@Module()` decorator and `Module` class

```plantuml
@startuml Pattern_Module
skinparam classAttributeIconSize 0

annotation Module {
    +imports?: Array<Type | DynamicModule>
    +controllers?: Type[]
    +providers?: Provider[]
    +exports?: Array<Provider | string | symbol>
}

class Module <<Runtime>> {
    -_token: string
    -_imports: Set<Module>
    -_providers: Map<InjectionToken, InstanceWrapper>
    -_controllers: Map<InjectionToken, InstanceWrapper>
    -_injectables: Map<InjectionToken, InstanceWrapper>
    -_exports: Set<InjectionToken>
    --
    +addProvider(provider): string
    +addController(controller): void
    +addImport(module): void
    +addExportedProviderOrModule(export): void
    +hasProvider(token): boolean
}

interface DynamicModule {
    +module: Type
    +imports?: ModuleMetadata['imports']
    +controllers?: ModuleMetadata['controllers']
    +providers?: ModuleMetadata['providers']
    +exports?: ModuleMetadata['exports']
    +global?: boolean
}

Module ..> DynamicModule : can be

note right of Module
  Module Pattern provides:
  - Encapsulation of related functionality
  - Explicit dependency declaration
  - Controlled visibility via exports
  - Reusable component composition
end note

@enduml
```

**Rationale**:
- Organizes application into cohesive blocks
- Explicit dependency declaration
- Controlled visibility through exports
- Supports lazy loading

---

## 5. Summary Table

| Pattern | Type | Location | Purpose |
|---------|------|----------|---------|
| Factory | Creational | `nest-factory.ts` | Application creation |
| Singleton | Creational | `instance-wrapper.ts` | Provider scoping |
| Builder | Creational | `builder.ts` | Middleware configuration |
| Adapter | Structural | `http-adapter.ts` | Platform abstraction |
| Decorator (GoF) | Structural | `router-execution-context.ts` | Pipeline enhancement |
| Proxy | Structural | `router-proxy.ts` | Exception handling |
| Facade | Structural | `nest-application.ts` | Simplified API |
| Composite | Structural | `modules-container.ts` | Module hierarchy |
| Chain of Responsibility | Behavioral | `guards-consumer.ts` | Request processing |
| Strategy | Behavioral | `opaque-key-factory/` | Token generation |
| Observer | Behavioral | `hooks/` | Lifecycle events |
| Template Method | Behavioral | `context-creator.ts` | Enhancer creation |
| Command | Behavioral | `repl/` | REPL commands |
| Visitor | Behavioral | `metadata-scanner.ts` | Metadata extraction |
| Dependency Injection | Architectural | `injector/` | IoC container |
| Module | Architectural | `module.decorator.ts` | Encapsulation |

---

## 6. References

- [Gang of Four Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [NestJS Architecture](https://docs.nestjs.com/fundamentals/custom-providers)
- [Martin Fowler's Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)
