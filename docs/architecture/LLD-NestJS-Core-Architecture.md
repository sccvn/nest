# Low-Level Design: NestJS Core Architecture

## 1. Overview

This document provides a comprehensive low-level architectural analysis of the NestJS core framework. NestJS is a progressive Node.js framework for building efficient, reliable, and scalable server-side applications built with TypeScript.

### 1.1 Design Goals

- **Modularity**: Encapsulate functionality into cohesive, reusable modules
- **Extensibility**: Allow customization through decorators, providers, and middleware
- **Dependency Injection**: Provide a powerful IoC container for managing dependencies
- **Platform Agnosticism**: Support multiple HTTP adapters (Express, Fastify)
- **Type Safety**: Leverage TypeScript for compile-time type checking

### 1.2 Package Structure

```
packages/
├── common/          # Decorators, interfaces, DTOs, pipes, guards
├── core/            # IoC container, router, application lifecycle
├── microservices/   # Message-based communication
├── platform-express/# Express HTTP adapter
├── platform-fastify/# Fastify HTTP adapter
├── platform-socket.io/ # Socket.IO WebSocket adapter
├── platform-ws/     # WS WebSocket adapter
├── testing/         # Testing utilities
└── websockets/      # WebSocket gateway support
```

---

## 2. Core Architecture Components

### 2.1 C4 Container Diagram

```plantuml
@startuml C4_NestJS_Container
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container Diagram - NestJS Framework

Person(developer, "Developer", "Builds applications using NestJS")

System_Boundary(nestjs, "NestJS Framework") {
    Container(factory, "NestFactory", "TypeScript", "Application bootstrap and factory")
    Container(container, "NestContainer", "TypeScript", "IoC Container - manages modules and dependencies")
    Container(injector, "Injector", "TypeScript", "Dependency injection engine")
    Container(scanner, "DependenciesScanner", "TypeScript", "Scans modules for metadata")
    Container(router, "RoutesResolver", "TypeScript", "HTTP route registration")
    Container(middleware, "MiddlewareModule", "TypeScript", "Middleware chain management")
    Container(app, "NestApplication", "TypeScript", "HTTP application instance")
    ContainerDb(metadata, "Metadata Storage", "Reflect.metadata", "Stores decorator metadata")
}

System_Ext(express, "Express/Fastify", "HTTP Adapter")

Rel(developer, factory, "Uses")
Rel(factory, container, "Creates")
Rel(factory, scanner, "Uses to scan")
Rel(scanner, container, "Populates")
Rel(container, injector, "Provides to")
Rel(injector, metadata, "Reads from")
Rel(factory, app, "Creates")
Rel(app, router, "Uses")
Rel(app, middleware, "Uses")
Rel(router, container, "Resolves from")
Rel(app, express, "Adapts to")

@enduml
```

### 2.2 C4 Component Diagram - Core Package

```plantuml
@startuml C4_NestJS_Component
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Component Diagram - @nestjs/core Package

Container_Boundary(core, "@nestjs/core") {
    Component(factory, "NestFactoryStatic", "Class", "Bootstrap application instances")
    Component(app, "NestApplication", "Class", "HTTP application with lifecycle management")
    Component(appContext, "NestApplicationContext", "Class", "Standalone application context")
    
    Component(container, "NestContainer", "Class", "IoC container holding modules")
    Component(module, "Module", "Class", "Module wrapper with providers/controllers")
    Component(wrapper, "InstanceWrapper", "Class", "Provider instance wrapper with scoping")
    Component(injector, "Injector", "Class", "Dependency resolution engine")
    Component(compiler, "ModuleCompiler", "Class", "Module compilation and tokenization")
    
    Component(scanner, "DependenciesScanner", "Class", "Module/provider scanning")
    Component(metascanner, "MetadataScanner", "Class", "Metadata extraction from classes")
    
    Component(resolver, "RoutesResolver", "Class", "Route registration coordinator")
    Component(explorer, "RouterExplorer", "Class", "Controller method exploration")
    Component(execContext, "RouterExecutionContext", "Class", "Request handler factory")
    
    Component(guardsCreator, "GuardsContextCreator", "Class", "Guard chain creation")
    Component(pipesCreator, "PipesContextCreator", "Class", "Pipe chain creation")
    Component(interceptorsCreator, "InterceptorsContextCreator", "Class", "Interceptor chain creation")
    Component(filtersCreator, "BaseExceptionFilterContext", "Class", "Exception filter chain creation")
    
    Component(middlewareModule, "MiddlewareModule", "Class", "Middleware configuration")
}

Rel(factory, container, "Creates")
Rel(factory, scanner, "Uses")
Rel(factory, app, "Creates")

Rel(app, resolver, "Uses")
Rel(app, middlewareModule, "Uses")

Rel(container, module, "Stores")
Rel(module, wrapper, "Contains")
Rel(injector, wrapper, "Resolves")
Rel(injector, container, "Queries")

Rel(scanner, container, "Populates")
Rel(scanner, metascanner, "Uses")

Rel(resolver, explorer, "Uses")
Rel(explorer, execContext, "Uses")

Rel(execContext, guardsCreator, "Uses")
Rel(execContext, pipesCreator, "Uses")
Rel(execContext, interceptorsCreator, "Uses")

@enduml
```

---

## 3. Dependency Injection System

### 3.1 Class Diagram - IoC Container

```plantuml
@startuml Class_IoC_Container
skinparam classAttributeIconSize 0
skinparam linetype ortho

title Class Diagram - IoC Container System

package "Injector System" {
    class NestContainer {
        -modules: ModulesContainer
        -globalModules: Set<Module>
        -dynamicModulesMetadata: Map<string, Partial<DynamicModule>>
        -moduleCompiler: ModuleCompiler
        -internalProvidersStorage: InternalProvidersStorage
        +addModule(metatype, scope): Promise<ModuleRef>
        +addProvider(provider, token): string
        +addController(controller, token): string
        +addImport(module, token): void
        +getModules(): ModulesContainer
        +getModuleByKey(key): Module
        +bindGlobalScope(): void
    }
    
    class Module {
        -_id: string
        -_token: string
        -_imports: Set<Module>
        -_providers: Map<InjectionToken, InstanceWrapper>
        -_controllers: Map<InjectionToken, InstanceWrapper>
        -_injectables: Map<InjectionToken, InstanceWrapper>
        -_exports: Set<InjectionToken>
        -_middlewares: Map<InjectionToken, InstanceWrapper>
        -_metatype: Type
        -_distance: number
        +addProvider(provider): string
        +addController(controller): string
        +addInjectable(injectable, host): InstanceWrapper
        +addExportedProviderOrModule(export): Set
        +getProviderByKey(key): InstanceWrapper
        +hasProvider(token): boolean
        +createModuleReferenceType(): Type<ModuleRef>
    }
    
    class InstanceWrapper<T> {
        +name: string
        +token: InjectionToken
        +metatype: Type<T>
        +instance: T
        +scope: Scope
        +async: boolean
        +host: Module
        +inject: InjectorDependency[]
        +forwardRef: boolean
        +durable: boolean
        -values: WeakMap<ContextId, InstancePerContext<T>>
        -transientMap: WeakMap<ContextId, WeakMap>
        +getInstanceByContextId(contextId): InstancePerContext<T>
        +setInstanceByContextId(contextId, value): void
        +isDependencyTreeStatic(): boolean
        +isDependencyTreeDurable(): boolean
        +createPrototype(contextId): void
    }
    
    class Injector {
        -instanceDecorator: Function
        +loadInstance<T>(wrapper, collection, moduleRef, contextId): Promise<void>
        +loadPrototype<T>(wrapper, collection, contextId): void
        +loadProvider(wrapper, moduleRef, contextId): Promise<void>
        +loadController(wrapper, moduleRef, contextId): Promise<void>
        +loadInjectable(wrapper, moduleRef, contextId): Promise<void>
        +resolveConstructorParams<T>(wrapper, moduleRef, inject): Promise<any[]>
        +lookupComponent(dependencies, moduleRef, wrapper): Promise<InstanceWrapper>
        +lookupComponentInImports(moduleRef, metatype): Promise<InstanceWrapper>
        +instantiateClass<T>(instances, wrapper, targetMetatype): Promise<T>
    }
    
    class ModuleCompiler {
        -moduleOpaqueKeyFactory: ModuleOpaqueKeyFactory
        +compile(metatype): ModuleFactory
        +extractMetadata(metatype): ModuleMetadata
    }
    
    class InstanceLoader {
        -injector: Injector
        -container: NestContainer
        +createInstancesOfDependencies(modules): Promise<void>
        +createPrototypes(modules): void
        +createInstances(modules): Promise<void>
    }
}

package "Scoping" {
    enum Scope {
        DEFAULT
        TRANSIENT
        REQUEST
    }
    
    interface ContextId {
        +id: number
        +payload?: unknown
    }
    
    interface InstancePerContext<T> {
        +instance: T
        +isResolved: boolean
        +isPending: boolean
        +donePromise: Promise<void>
    }
}

NestContainer "1" *-- "*" Module : contains
Module "1" *-- "*" InstanceWrapper : stores
Injector --> NestContainer : queries
Injector --> InstanceWrapper : resolves
InstanceLoader --> Injector : uses
InstanceWrapper --> Scope : has
InstanceWrapper --> ContextId : keyed by
ModuleCompiler --> NestContainer : used by

@enduml
```

### 3.2 Sequence Diagram - Dependency Resolution

```plantuml
@startuml Sequence_DI_Resolution
title Dependency Injection Resolution Flow

actor Application
participant "NestFactory" as Factory
participant "DependenciesScanner" as Scanner
participant "NestContainer" as Container
participant "InstanceLoader" as Loader
participant "Injector" as Injector
participant "Module" as Module
participant "InstanceWrapper" as Wrapper

Application -> Factory: create(AppModule)
activate Factory

Factory -> Container: new NestContainer()
activate Container

Factory -> Scanner: scan(AppModule)
activate Scanner

Scanner -> Container: addModule(AppModule)
Container -> Module: new Module(AppModule)
activate Module
Module --> Container: moduleRef
Container --> Scanner: token

Scanner -> Scanner: scanModulesForDependencies()
Scanner -> Container: addProvider(ServiceA)
Container -> Module: addProvider(ServiceA)
Module -> Wrapper: new InstanceWrapper(ServiceA)
activate Wrapper
Wrapper --> Module: wrapper
deactivate Wrapper
Module --> Container: providerRef

Scanner -> Container: addController(ControllerA)
Container -> Module: addController(ControllerA)
Module -> Wrapper: new InstanceWrapper(ControllerA)
activate Wrapper
Wrapper --> Module: wrapper
deactivate Wrapper

deactivate Scanner

Factory -> Loader: createInstancesOfDependencies()
activate Loader

Loader -> Loader: createPrototypes(modules)
loop for each module
    Loader -> Injector: loadPrototype(wrapper)
    activate Injector
    Injector -> Wrapper: createPrototype(contextId)
    Wrapper --> Injector: prototype
    deactivate Injector
end

Loader -> Loader: createInstances(modules)
loop for each provider
    Loader -> Injector: loadProvider(wrapper, module)
    activate Injector
    Injector -> Injector: resolveConstructorParams(wrapper)
    
    loop for each dependency
        Injector -> Injector: lookupComponent(dependency, module)
        Injector -> Module: getProviderByKey(token)
        Module --> Injector: dependencyWrapper
        
        alt dependency not resolved
            Injector -> Injector: loadInstance(dependencyWrapper)
        end
    end
    
    Injector -> Injector: instantiateClass(params, wrapper)
    Injector -> Wrapper: setInstanceByContextId(instance)
    deactivate Injector
end

deactivate Loader
deactivate Module
deactivate Container

Factory --> Application: NestApplication
deactivate Factory

@enduml
```

### 3.3 Provider Scoping Strategy

```plantuml
@startuml State_Provider_Scope
title Provider Instance Lifecycle by Scope

state "DEFAULT Scope" as Default {
    [*] --> Singleton
    Singleton: Created once at app start
    Singleton: Shared across all requests
    Singleton: Lives until app shutdown
    Singleton --> [*] : onModuleDestroy
}

state "REQUEST Scope" as Request {
    [*] --> PerRequest
    PerRequest: Created for each request
    PerRequest: Isolated per request context
    PerRequest: Garbage collected after response
    
    state "Request Context" as ReqCtx {
        [*] --> Creating
        Creating --> Resolved : instance ready
        Resolved --> InUse : handler invoked
        InUse --> Disposed : response sent
        Disposed --> [*]
    }
}

state "TRANSIENT Scope" as Transient {
    [*] --> PerInjection
    PerInjection: New instance per injection
    PerInjection: Each consumer gets unique instance
    PerInjection: No caching
    PerInjection --> [*] : out of scope
}

@enduml
```

---

## 4. Request Processing Pipeline

### 4.1 Class Diagram - Router System

```plantuml
@startuml Class_Router_System
skinparam classAttributeIconSize 0

title Class Diagram - Router System

package "Router" {
    class RoutesResolver {
        -container: NestContainer
        -routerProxy: RouterProxy
        -routerExplorer: RouterExplorer
        -routePathFactory: RoutePathFactory
        +resolve(applicationRef, globalPrefix): void
        +registerRouters(routes, prefix, module): void
        +registerNotFoundHandler(): void
        +registerExceptionHandler(): void
    }
    
    class RouterExplorer {
        -executionContextCreator: RouterExecutionContext
        -pathsExplorer: PathsExplorer
        -routerMethodFactory: RouterMethodFactory
        -exceptionFiltersCache: WeakMap
        +explore(instance, module, appRef, routePath): RouteDefinition[]
        +applyPathsToRouterProxy(router, routes, module): void
        +applyCallbackToRouter(router, routeDefinition): void
        -createCallbackProxy(instance, callback, method): Function
        -createRequestScopedHandler(wrapper, moduleKey): Function
    }
    
    class RouterExecutionContext {
        -paramsFactory: RouteParamsFactory
        -pipesContextCreator: PipesContextCreator
        -pipesConsumer: PipesConsumer
        -guardsContextCreator: GuardsContextCreator
        -guardsConsumer: GuardsConsumer
        -interceptorsContextCreator: InterceptorsContextCreator
        -interceptorsConsumer: InterceptorsConsumer
        +create(instance, callback, methodName, moduleKey): Function
        +getMetadata(instance, callback, methodName): HandlerMetadata
        -createGuardsFn(guards, instance, callback): Function
        -createPipesFn(pipes, paramsOptions): Function
        -exchangeKeysForValues(keys, data): ParamProperties[]
    }
    
    class RouteParamsFactory {
        +exchangeKeyForValue(key, data, metadata): any
    }
    
    class PathsExplorer {
        -metadataScanner: MetadataScanner
        +scanForPaths(instance): RoutePathMetadata[]
        +exploreMethodMetadata(instance, prototype, methodName): RoutePathMetadata
    }
    
    class RouterProxy {
        +createProxy(targetCallback, exceptionHandler): Function
        +createExceptionLayerProxy(target, exceptionHandler): Function
    }
}

package "Enhancers" {
    abstract class ContextCreator {
        #config: ApplicationConfig
        #container: NestContainer
        +create(instance, callback, module, contextId): T[]
        #{abstract} createContext(args, type, host): T[]
    }
    
    class GuardsContextCreator extends ContextCreator {
        +create(instance, callback, module): CanActivate[]
        -getGuardInstance(guard): CanActivate
    }
    
    class PipesContextCreator extends ContextCreator {
        +create(instance, callback, module): PipeTransform[]
        -getPipeInstance(pipe): PipeTransform
    }
    
    class InterceptorsContextCreator extends ContextCreator {
        +create(instance, callback, module): NestInterceptor[]
        -getInterceptorInstance(interceptor): NestInterceptor
    }
}

RoutesResolver --> RouterExplorer : uses
RoutesResolver --> RouterProxy : uses
RouterExplorer --> RouterExecutionContext : uses
RouterExplorer --> PathsExplorer : uses
RouterExecutionContext --> GuardsContextCreator : uses
RouterExecutionContext --> PipesContextCreator : uses
RouterExecutionContext --> InterceptorsContextCreator : uses
RouterExecutionContext --> RouteParamsFactory : uses

@enduml
```

### 4.2 Sequence Diagram - Request Handler Execution

```plantuml
@startuml Sequence_Request_Handler
title HTTP Request Processing Pipeline

actor Client
participant "HTTP Adapter\n(Express/Fastify)" as Adapter
participant "MiddlewareModule" as Middleware
participant "RouterProxy" as Proxy
participant "GuardsConsumer" as Guards
participant "InterceptorsConsumer" as Interceptors
participant "PipesConsumer" as Pipes
participant "Controller" as Controller
participant "ExceptionFilter" as Filter

Client -> Adapter: HTTP Request
activate Adapter

Adapter -> Middleware: next(req, res)
activate Middleware
note right: Middleware chain execution

Middleware -> Proxy: handler(req, res, next)
deactivate Middleware
activate Proxy

Proxy -> Guards: tryActivate(guards, context)
activate Guards

loop for each guard
    Guards -> Guards: canActivate(context)
    alt guard returns false
        Guards --> Proxy: ForbiddenException
        Proxy --> Filter: handleException
        Filter --> Adapter: 403 Forbidden
    end
end
Guards --> Proxy: true
deactivate Guards

Proxy -> Interceptors: intercept(context, next)
activate Interceptors
note right: Pre-processing interceptors

Interceptors -> Pipes: transform(value, metadata)
activate Pipes
loop for each parameter
    Pipes -> Pipes: validate/transform
    alt validation fails
        Pipes --> Filter: BadRequestException
        Filter --> Adapter: 400 Bad Request
    end
end
Pipes --> Interceptors: transformed params
deactivate Pipes

Interceptors -> Controller: method(params)
activate Controller
Controller --> Interceptors: result
deactivate Controller

note right: Post-processing interceptors
Interceptors -> Interceptors: map/tap operators

Interceptors --> Proxy: Observable<result>
deactivate Interceptors

Proxy --> Adapter: response
deactivate Proxy

Adapter --> Client: HTTP Response
deactivate Adapter

@enduml
```

### 4.3 Activity Diagram - Request Flow

```plantuml
@startuml Activity_Request_Flow
title NestJS Request Processing Activity

start

:Receive HTTP Request;

partition "Middleware Phase" {
    :Execute Global Middleware;
    :Execute Module Middleware;
    :Execute Route Middleware;
}

partition "Guard Phase" {
    :Collect Guards\n(Global → Controller → Method);
    
    while (More Guards?) is (yes)
        :Execute Guard.canActivate();
        if (Returns true?) then (yes)
            :Continue;
        else (no)
            :Throw ForbiddenException;
            stop
        endif
    endwhile (no)
}

partition "Interceptor Pre-Processing" {
    :Collect Interceptors\n(Global → Controller → Method);
    :Create Execution Context;
    
    while (More Pre-Interceptors?) is (yes)
        :Execute Interceptor.intercept()\nwith next() callback;
    endwhile (no)
}

partition "Parameter Processing" {
    :Extract Route Params;
    :Extract Query Params;
    :Extract Body;
    :Extract Headers;
    
    while (More Pipes?) is (yes)
        :Execute Pipe.transform();
        if (Validation Passes?) then (yes)
            :Continue;
        else (no)
            :Throw BadRequestException;
            stop
        endif
    endwhile (no)
}

partition "Handler Execution" {
    :Invoke Controller Method;
    :Capture Return Value;
}

partition "Interceptor Post-Processing" {
    while (More Post-Interceptors?) is (yes)
        :Process RxJS operators\n(map, tap, catchError);
    endwhile (no)
}

partition "Response Phase" {
    if (Return type?) then (Observable)
        :Subscribe and await;
    elseif (Return type?) then (Promise)
        :Await resolution;
    else (Value)
        :Use directly;
    endif
    
    :Serialize Response;
    :Send HTTP Response;
}

stop

@enduml
```

---

## 5. Design Patterns Implemented

### 5.1 Factory Pattern - NestFactory

```plantuml
@startuml Pattern_Factory
skinparam classAttributeIconSize 0

title Factory Pattern - NestFactory

interface INestApplicationContext {
    +select<T>(module: Type<T>): INestApplicationContext
    +get<T>(typeOrToken: Type<T>): T
    +resolve<T>(typeOrToken: Type<T>): Promise<T>
    +close(): Promise<void>
}

interface INestApplication extends INestApplicationContext {
    +listen(port: number): Promise<void>
    +use(middleware: any): this
    +enableCors(options?: CorsOptions): void
    +setGlobalPrefix(prefix: string): void
}

interface INestMicroservice extends INestApplicationContext {
    +listen(): Promise<void>
    +close(): Promise<void>
}

class NestFactoryStatic <<Factory>> {
    -{static} logger: Logger
    -{static} abortOnError: boolean
    +{static} create(module: Type): Promise<INestApplication>
    +{static} createMicroservice(module: Type, options): Promise<INestMicroservice>
    +{static} createApplicationContext(module: Type): Promise<INestApplicationContext>
    -{static} initialize(module, container, graphInspector, config): Promise<void>
    -{static} createHttpAdapter(): AbstractHttpAdapter
}

class NestApplication implements INestApplication {
    -httpServer: http.Server
    -middlewareModule: MiddlewareModule
    -routesResolver: RoutesResolver
    +listen(port, hostname?): Promise<void>
    +init(): Promise<this>
    +use(middleware): this
}

class NestApplicationContext implements INestApplicationContext {
    -container: NestContainer
    -injector: Injector
    +select<T>(module): INestApplicationContext
    +get<T>(typeOrToken): T
    +resolve<T>(typeOrToken, contextId?): Promise<T>
}

class NestMicroservice implements INestMicroservice {
    -server: Server
    +listen(): Promise<void>
    +close(): Promise<void>
}

NestFactoryStatic ..> NestApplication : creates
NestFactoryStatic ..> NestApplicationContext : creates
NestFactoryStatic ..> NestMicroservice : creates
NestApplication --|> NestApplicationContext

@enduml
```

### 5.2 Decorator Pattern - Request Pipeline Enhancers

```plantuml
@startuml Pattern_Decorator
skinparam classAttributeIconSize 0

title Decorator Pattern - Pipeline Enhancers

interface CanActivate {
    +canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean>
}

interface NestInterceptor<T, R> {
    +intercept(context: ExecutionContext, next: CallHandler<T>): Observable<R>
}

interface PipeTransform<T, R> {
    +transform(value: T, metadata: ArgumentMetadata): R
}

interface ExceptionFilter<T> {
    +catch(exception: T, host: ArgumentsHost): void
}

class AuthGuard implements CanActivate {
    +canActivate(context): boolean
}

class RolesGuard implements CanActivate {
    +canActivate(context): boolean
}

class LoggingInterceptor implements NestInterceptor {
    +intercept(context, next): Observable
}

class TransformInterceptor implements NestInterceptor {
    +intercept(context, next): Observable
}

class ValidationPipe implements PipeTransform {
    +transform(value, metadata): any
}

class ParseIntPipe implements PipeTransform {
    +transform(value, metadata): number
}

class HttpExceptionFilter implements ExceptionFilter {
    +catch(exception, host): void
}

note bottom of CanActivate
  Guards "decorate" the handler
  by adding authorization checks
end note

note bottom of NestInterceptor
  Interceptors "decorate" the handler
  with pre/post processing logic
end note

note bottom of PipeTransform
  Pipes "decorate" parameters
  with transformation/validation
end note

@enduml
```

### 5.3 Strategy Pattern - HTTP Adapters

```plantuml
@startuml Pattern_Strategy
skinparam classAttributeIconSize 0

title Strategy Pattern - Platform Adapters

abstract class AbstractHttpAdapter<TServer, TRequest, TResponse> {
    #{abstract} instance: any
    +{abstract} listen(port: number, hostname?: string): Promise<void>
    +{abstract} close(): Promise<void>
    +{abstract} get(path: string, handler: RequestHandler): void
    +{abstract} post(path: string, handler: RequestHandler): void
    +{abstract} put(path: string, handler: RequestHandler): void
    +{abstract} delete(path: string, handler: RequestHandler): void
    +{abstract} patch(path: string, handler: RequestHandler): void
    +{abstract} options(path: string, handler: RequestHandler): void
    +{abstract} use(handler: RequestHandler): void
    +{abstract} useStaticAssets(path: string, options: any): void
    +{abstract} setViewEngine(engine: string): void
    +{abstract} getRequestHostname(request: TRequest): string
    +{abstract} getRequestMethod(request: TRequest): string
    +{abstract} getRequestUrl(request: TRequest): string
    +{abstract} reply(response: TResponse, body: any, statusCode?: number): void
}

class ExpressAdapter extends AbstractHttpAdapter {
    -instance: Express
    +listen(port, hostname?): Promise<void>
    +get(path, handler): void
    +post(path, handler): void
    +use(handler): void
    +reply(response, body, statusCode?): void
    +getRequestUrl(request): string
}

class FastifyAdapter extends AbstractHttpAdapter {
    -instance: FastifyInstance
    +listen(port, hostname?): Promise<void>
    +get(path, handler): void
    +post(path, handler): void
    +use(handler): void
    +reply(response, body, statusCode?): void
    +getRequestUrl(request): string
}

class NestApplication {
    -httpAdapter: AbstractHttpAdapter
    +getHttpAdapter(): AbstractHttpAdapter
    +listen(port, hostname?): Promise<void>
}

NestApplication --> AbstractHttpAdapter : uses
AbstractHttpAdapter <|-- ExpressAdapter
AbstractHttpAdapter <|-- FastifyAdapter

note right of AbstractHttpAdapter
  Strategy interface allows
  swapping HTTP server implementations
  without changing application code
end note

@enduml
```

### 5.4 Observer Pattern - Lifecycle Hooks

```plantuml
@startuml Pattern_Observer
skinparam classAttributeIconSize 0

title Observer Pattern - Module Lifecycle Hooks

interface OnModuleInit {
    +onModuleInit(): any
}

interface OnModuleDestroy {
    +onModuleDestroy(): any
}

interface OnApplicationBootstrap {
    +onApplicationBootstrap(): any
}

interface BeforeApplicationShutdown {
    +beforeApplicationShutdown(signal?: string): any
}

interface OnApplicationShutdown {
    +onApplicationShutdown(signal?: string): any
}

class NestApplicationContext <<Subject>> {
    -container: NestContainer
    +init(): Promise<this>
    +close(): Promise<void>
    -callInitHook(): Promise<void>
    -callDestroyHook(): Promise<void>
    -callBootstrapHook(): Promise<void>
    -callBeforeShutdownHook(): Promise<void>
    -callShutdownHook(): Promise<void>
}

class ServiceA <<Observer>> implements OnModuleInit, OnModuleDestroy {
    +onModuleInit(): void
    +onModuleDestroy(): void
}

class ServiceB <<Observer>> implements OnApplicationBootstrap {
    +onApplicationBootstrap(): void
}

class ServiceC <<Observer>> implements OnApplicationShutdown {
    +onApplicationShutdown(signal): void
}

NestApplicationContext --> ServiceA : notifies
NestApplicationContext --> ServiceB : notifies
NestApplicationContext --> ServiceC : notifies

note bottom of NestApplicationContext
  Application context acts as Subject
  notifying all providers implementing
  lifecycle interfaces at appropriate times
end note

@enduml
```

### 5.5 Chain of Responsibility - Middleware/Guards/Interceptors

```plantuml
@startuml Pattern_Chain
skinparam classAttributeIconSize 0

title Chain of Responsibility - Request Processing

interface Handler {
    +handle(request: any, next: () => void): void
}

class MiddlewareChain <<Chain>> {
    -middlewares: NestMiddleware[]
    +use(middleware: NestMiddleware): void
    +execute(req, res, next): void
}

abstract class NestMiddleware implements Handler {
    +{abstract} use(req: Request, res: Response, next: Function): void
}

class LoggerMiddleware extends NestMiddleware {
    +use(req, res, next): void
}

class AuthMiddleware extends NestMiddleware {
    +use(req, res, next): void
}

class CorsMiddleware extends NestMiddleware {
    +use(req, res, next): void
}

class GuardsConsumer <<Chain>> {
    +tryActivate(guards: CanActivate[], context: ExecutionContext): Promise<boolean>
}

class InterceptorsConsumer <<Chain>> {
    +intercept(interceptors: NestInterceptor[], context, next): Observable<any>
}

class PipesConsumer <<Chain>> {
    +apply(value: any, metadata, pipes: PipeTransform[]): Promise<any>
}

MiddlewareChain --> NestMiddleware : chains
LoggerMiddleware --> AuthMiddleware : next()
AuthMiddleware --> CorsMiddleware : next()

note bottom of MiddlewareChain
  Each handler in the chain processes
  the request and calls next() to
  pass control to the next handler
end note

@enduml
```

### 5.6 Proxy Pattern - RouterProxy

```plantuml
@startuml Pattern_Proxy
skinparam classAttributeIconSize 0

title Proxy Pattern - Router Proxy

interface RequestHandler {
    +handle(req: Request, res: Response, next: Function): any
}

class RouterProxy <<Proxy>> {
    +createProxy(targetCallback: Function, exceptionsHandler: ExceptionsHandler): Function
    +createExceptionLayerProxy(targetCallback: Function, exceptionsHandler: ExceptionsHandler): Function
}

class ControllerMethod <<RealSubject>> {
    +execute(params): any
}

class ExceptionsHandler {
    -filters: ExceptionFilter[]
    +next(exception: Error, context: ArgumentsHost): void
}

class ExceptionsZone {
    +run(callback: Function): Promise<void>
    +asyncRun(callback: Function): Promise<void>
}

RouterProxy --> ControllerMethod : wraps
RouterProxy --> ExceptionsHandler : uses
RouterProxy --> ExceptionsZone : uses

note bottom of RouterProxy
  RouterProxy wraps controller methods
  to add exception handling and
  execution zone protection
end note

@enduml
```

---

## 6. Module System Architecture

### 6.1 Module Compilation Flow

```plantuml
@startuml Sequence_Module_Compilation
title Module Compilation and Registration

participant "DependenciesScanner" as Scanner
participant "ModuleCompiler" as Compiler
participant "ModuleOpaqueKeyFactory" as KeyFactory
participant "NestContainer" as Container
participant "Module" as Module

Scanner -> Compiler: compile(AppModule)
activate Compiler

Compiler -> Compiler: extractMetadata(AppModule)
note right: Extract @Module() decorator metadata

Compiler -> KeyFactory: createModuleKey(moduleId)
activate KeyFactory
KeyFactory --> Compiler: opaqueToken
deactivate KeyFactory

Compiler --> Scanner: { type, dynamicMetadata, token }
deactivate Compiler

Scanner -> Container: addModule(AppModule, scope)
activate Container

Container -> Module: new Module(metatype, container)
activate Module

Module -> Module: addModuleRef()
note right: Add self-reference provider

Module -> Module: addCoreProviders()
note right: Add ModuleRef, Reflector

Module --> Container: moduleRef
deactivate Module

Container -> Container: modules.set(token, moduleRef)
Container --> Scanner: token
deactivate Container

loop for each import
    Scanner -> Scanner: scanForModules(importedModule, scope, registry)
end

Scanner -> Scanner: scanModulesForDependencies()
note right: Second pass for providers, controllers, exports

@enduml
```

### 6.2 Dynamic Module Structure

```plantuml
@startuml Class_Dynamic_Module
skinparam classAttributeIconSize 0

title Dynamic Module Configuration

interface DynamicModule {
    +module: Type
    +imports?: Array<Type | DynamicModule | ForwardReference>
    +controllers?: Type[]
    +providers?: Provider[]
    +exports?: Array<DynamicModule | Provider | string | symbol>
    +global?: boolean
}

interface ModuleMetadata {
    +imports?: Array<Type | DynamicModule | ForwardReference>
    +controllers?: Type[]
    +providers?: Provider[]
    +exports?: Array<Provider | string | symbol>
}

class ConfigModule <<DynamicModule>> {
    +{static} forRoot(options: ConfigOptions): DynamicModule
    +{static} forRootAsync(options: ConfigAsyncOptions): DynamicModule
    +{static} forFeature(options: ConfigOptions): DynamicModule
}

interface ConfigOptions {
    +isGlobal?: boolean
    +envFilePath?: string | string[]
    +ignoreEnvFile?: boolean
    +load?: ConfigFactory[]
}

interface ConfigAsyncOptions {
    +imports?: ModuleMetadata['imports']
    +useClass?: Type<ConfigOptionsFactory>
    +useExisting?: Type<ConfigOptionsFactory>
    +useFactory?: (...args: any[]) => ConfigOptions
    +inject?: any[]
}

ConfigModule ..> DynamicModule : returns
ConfigModule --> ConfigOptions : uses
ConfigModule --> ConfigAsyncOptions : uses

note bottom of ConfigModule
  Dynamic modules allow runtime
  configuration through static
  factory methods (forRoot, forFeature)
end note

@enduml
```

---

## 7. Error Handling Architecture

### 7.1 Exception Hierarchy

```plantuml
@startuml Class_Exception_Hierarchy
skinparam classAttributeIconSize 0

title Exception Class Hierarchy

class Error {
    +message: string
    +name: string
    +stack?: string
}

abstract class HttpException extends Error {
    -response: string | object
    -status: number
    +getResponse(): string | object
    +getStatus(): number
    +initMessage(): void
}

class BadRequestException extends HttpException {
    +constructor(message?: string | object)
}

class UnauthorizedException extends HttpException {
    +constructor(message?: string | object)
}

class ForbiddenException extends HttpException {
    +constructor(message?: string | object)
}

class NotFoundException extends HttpException {
    +constructor(message?: string | object)
}

class ConflictException extends HttpException {
    +constructor(message?: string | object)
}

class InternalServerErrorException extends HttpException {
    +constructor(message?: string | object)
}

class BadGatewayException extends HttpException {
    +constructor(message?: string | object)
}

class ServiceUnavailableException extends HttpException {
    +constructor(message?: string | object)
}

note right of HttpException
  Base class for all HTTP exceptions
  with status code and response body
end note

@enduml
```

### 7.2 Exception Filter Chain

```plantuml
@startuml Sequence_Exception_Handling
title Exception Handling Flow

participant "Controller" as Controller
participant "ExceptionsHandler" as Handler
participant "ExternalExceptionsHandler" as External
participant "BaseExceptionFilter" as BaseFilter
participant "CustomFilter" as Custom
participant "HttpAdapter" as Adapter

Controller -> Handler: throw HttpException
activate Handler

Handler -> Handler: invokeCustomFilters(exception, host)

alt Custom filter exists
    Handler -> Custom: catch(exception, host)
    activate Custom
    Custom -> Adapter: reply(response, body, statusCode)
    deactivate Custom
else No custom filter
    Handler -> BaseFilter: catch(exception, host)
    activate BaseFilter
    
    alt Is HttpException
        BaseFilter -> BaseFilter: handleHttpException(exception, host)
        BaseFilter -> Adapter: reply(response, body, statusCode)
    else Is unknown exception
        BaseFilter -> BaseFilter: handleUnknownException(exception, host)
        BaseFilter -> Adapter: reply(response, "Internal Server Error", 500)
    end
    deactivate BaseFilter
end

deactivate Handler

@enduml
```

---

## 8. Metadata Reflection System

### 8.1 Decorator Metadata Flow

```plantuml
@startuml Sequence_Metadata_Reflection
title Decorator Metadata Registration and Reading

participant "Developer Code" as Dev
participant "@Controller()" as ControllerDec
participant "@Get()" as GetDec
participant "@Injectable()" as InjectableDec
participant "Reflect.metadata" as Reflect
participant "MetadataScanner" as Scanner
participant "DependenciesScanner" as DepScanner

== Decoration Phase (Compile Time) ==

Dev -> ControllerDec: @Controller('users')
activate ControllerDec
ControllerDec -> Reflect: defineMetadata(PATH_METADATA, 'users', target)
ControllerDec -> Reflect: defineMetadata(CONTROLLER_WATERMARK, true, target)
deactivate ControllerDec

Dev -> GetDec: @Get(':id')
activate GetDec
GetDec -> Reflect: defineMetadata(PATH_METADATA, ':id', target, propertyKey)
GetDec -> Reflect: defineMetadata(METHOD_METADATA, RequestMethod.GET, target, propertyKey)
deactivate GetDec

Dev -> InjectableDec: @Injectable()
activate InjectableDec
InjectableDec -> Reflect: defineMetadata(INJECTABLE_WATERMARK, true, target)
deactivate InjectableDec

== Scanning Phase (Runtime) ==

DepScanner -> Scanner: scanForPaths(controllerInstance)
activate Scanner

Scanner -> Reflect: getMetadata(PATH_METADATA, target)
Reflect --> Scanner: 'users'

Scanner -> Reflect: getMetadata(METHOD_METADATA, target, 'findOne')
Reflect --> Scanner: RequestMethod.GET

Scanner -> Reflect: getMetadata('design:paramtypes', target)
Reflect --> Scanner: [UserService, String]

Scanner --> DepScanner: RoutePathMetadata[]
deactivate Scanner

@enduml
```

### 8.2 Custom Decorator Creation

```plantuml
@startuml Class_Custom_Decorator
skinparam classAttributeIconSize 0

title Custom Decorator Pattern

class SetMetadata <<Decorator Factory>> {
    +{static} (key: string, value: any): CustomDecorator
}

interface CustomDecorator {
    +KEY: string
}

class Roles <<Custom Decorator>> {
    +{static} (...roles: string[]): CustomDecorator
}

class RolesGuard implements CanActivate {
    -reflector: Reflector
    +canActivate(context: ExecutionContext): boolean
}

class Reflector {
    +get<T>(metadataKey: string, target: Type | Function): T
    +getAll<T>(metadataKey: string, targets: Type[]): T[]
    +getAllAndMerge<T>(metadataKey: string, targets: Type[]): T[]
    +getAllAndOverride<T>(metadataKey: string, targets: Type[]): T
}

SetMetadata ..> CustomDecorator : creates
Roles --> SetMetadata : uses
RolesGuard --> Reflector : uses

note bottom of Roles
  @Roles('admin', 'user')
  @SetMetadata('roles', ['admin', 'user'])
end note

note bottom of RolesGuard
  const roles = this.reflector.get<string[]>(
    'roles',
    context.getHandler()
  );
end note

@enduml
```

---

## 9. Communication Diagram

### 9.1 Application Bootstrap Communication

```plantuml
@startuml Communication_Bootstrap
title Application Bootstrap Object Communication

object "NestFactory" as Factory
object "NestContainer" as Container
object "DependenciesScanner" as Scanner
object "InstanceLoader" as Loader
object "Injector" as Injector
object "NestApplication" as App
object "RoutesResolver" as Routes
object "MiddlewareModule" as Middleware
object "GraphInspector" as Inspector

Factory --> Container : 1: create
Factory --> Scanner : 2: create
Factory --> Inspector : 3: create
Factory --> Scanner : 4: scan(module)
Scanner --> Container : 5: addModule
Scanner --> Container : 6: addProvider
Scanner --> Container : 7: addController
Factory --> Loader : 8: createInstances
Loader --> Injector : 9: loadInstance
Injector --> Container : 10: resolve
Factory --> App : 11: create
App --> Routes : 12: resolve
App --> Middleware : 13: configure
Routes --> Container : 14: getControllers
Routes --> Inspector : 15: inspect

@enduml
```

---

## 10. State Machine - Application Lifecycle

```plantuml
@startuml State_Application_Lifecycle
title NestJS Application Lifecycle States

[*] --> Initializing : NestFactory.create()

state Initializing {
    [*] --> CreatingContainer
    CreatingContainer --> ScanningModules : container ready
    ScanningModules --> CompilingModules : modules scanned
    CompilingModules --> LoadingDependencies : modules compiled
    LoadingDependencies --> ResolvingRoutes : dependencies loaded
    ResolvingRoutes --> ConfiguringMiddleware : routes resolved
    ConfiguringMiddleware --> [*] : middleware configured
}

Initializing --> Ready : init() complete

state Ready {
    [*] --> Idle
    Idle --> Listening : listen()
    Listening --> ProcessingRequest : request received
    ProcessingRequest --> Listening : response sent
}

Ready --> ShuttingDown : close() / SIGTERM

state ShuttingDown {
    [*] --> BeforeShutdown
    BeforeShutdown: beforeApplicationShutdown()
    BeforeShutdown --> OnShutdown : complete
    OnShutdown: onApplicationShutdown()
    OnShutdown --> ModuleDestroy : complete
    ModuleDestroy: onModuleDestroy()
    ModuleDestroy --> Cleanup : complete
    Cleanup --> [*] : resources released
}

ShuttingDown --> [*] : shutdown complete

@enduml
```

---

## 11. Algorithms

### 11.1 Module Distance Calculation

```
ALGORITHM CalculateModulesDistance
  INPUT: modules: Map<string, Module>, rootModule: Module
  OUTPUT: void (modules are mutated with distance values)
  
  1. SET rootModule.distance = 0
  2. SET queue = [rootModule]
  3. SET visited = new Set([rootModule.token])
  
  4. WHILE queue is not empty:
     a. SET current = queue.shift()
     b. FOR EACH importedModule IN current.imports:
        i.  IF importedModule.token NOT IN visited:
            - ADD importedModule.token TO visited
            - SET importedModule.distance = current.distance + 1
            - PUSH importedModule TO queue
  
  5. RETURN
```

### 11.2 Dependency Resolution Algorithm

```
ALGORITHM ResolveDependency
  INPUT: wrapper: InstanceWrapper, moduleRef: Module, contextId: ContextId
  OUTPUT: resolvedInstance: T
  
  1. IF wrapper.isResolved(contextId):
     RETURN wrapper.getInstanceByContextId(contextId)
  
  2. SET dependencies = getConstructorDependencies(wrapper.metatype)
  
  3. FOR EACH dependency IN dependencies:
     a. SET depWrapper = lookupComponent(dependency, moduleRef)
     
     b. IF depWrapper IS NULL:
        i.  SET depWrapper = lookupComponentInImports(moduleRef, dependency)
        
     c. IF depWrapper IS NULL:
        THROW UnknownDependenciesException
     
     d. IF NOT depWrapper.isResolved(contextId):
        CALL ResolveDependency(depWrapper, depWrapper.host, contextId)
  
  4. SET resolvedDeps = dependencies.map(d => d.instance)
  
  5. SET instance = instantiateClass(resolvedDeps, wrapper.metatype)
  
  6. CALL wrapper.setInstanceByContextId(contextId, instance)
  
  7. RETURN instance
```

### 11.3 Guard Chain Execution

```
ALGORITHM TryActivateGuards
  INPUT: guards: CanActivate[], context: ExecutionContext
  OUTPUT: boolean
  
  1. FOR i = 0 TO guards.length - 1:
     a. SET guard = guards[i]
     b. SET result = guard.canActivate(context)
     
     c. IF result IS Promise:
        SET result = AWAIT result
        
     d. IF result IS Observable:
        SET result = AWAIT firstValueFrom(result)
     
     e. IF result IS FALSE:
        THROW ForbiddenException
  
  2. RETURN TRUE
```

---

## 12. Quality Attributes

### 12.1 Design Principles Applied

| Principle | Implementation |
|-----------|----------------|
| **Single Responsibility** | Each class has one reason to change (Injector resolves, Scanner scans, Router routes) |
| **Open/Closed** | Framework extensible via decorators, middleware, guards without modification |
| **Liskov Substitution** | HTTP adapters are interchangeable (Express/Fastify) |
| **Interface Segregation** | Small focused interfaces (CanActivate, PipeTransform, NestInterceptor) |
| **Dependency Inversion** | High-level modules depend on abstractions (AbstractHttpAdapter) |

### 12.2 Extensibility Points

1. **Custom Decorators** - Extend metadata system
2. **Custom Providers** - Factory, Value, Class, Existing providers
3. **Custom Middleware** - Request preprocessing
4. **Custom Guards** - Authorization logic
5. **Custom Interceptors** - Cross-cutting concerns
6. **Custom Pipes** - Validation/Transformation
7. **Custom Exception Filters** - Error handling
8. **Custom HTTP Adapters** - Platform support

---

## 13. References

- [NestJS Official Documentation](https://docs.nestjs.com/)
- [NestJS GitHub Repository](https://github.com/nestjs/nest)
- [TypeScript Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html)
- [Reflect Metadata](https://github.com/rbuckton/reflect-metadata)
- [Dependency Injection Pattern](https://martinfowler.com/articles/injection.html)
