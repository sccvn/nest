# Algorithms & Data Structures - NestJS Framework

## Overview

This document details the key algorithms and data structures used in the NestJS core framework, with complexity analysis and pseudocode representations.

---

## 1. Data Structures

### 1.1 Module Graph

**Location**: `packages/core/injector/modules-container.ts`

**Structure**: Directed Acyclic Graph (DAG) of modules

```plantuml
@startuml DS_ModuleGraph
skinparam classAttributeIconSize 0

class ModulesContainer {
    -modules: Map<string, Module>
    +get(token: string): Module
    +set(token: string, module: Module): void
    +values(): IterableIterator<Module>
    +forEach(callback): void
}

class Module {
    -_token: string
    -_imports: Set<Module>
    -_exports: Set<InjectionToken>
    -distance: number
    +imports: Set<Module>
    +exports: Set<InjectionToken>
}

ModulesContainer "1" o-- "*" Module : contains
Module "*" --> "*" Module : imports

note bottom of ModulesContainer
  Graph Properties:
  - Directed: imports have direction
  - Acyclic: circular imports detected
  - Weighted: distance from root
end note

@enduml
```

**Complexity**:
| Operation | Time Complexity |
|-----------|-----------------|
| Add Module | O(1) |
| Get Module | O(1) |
| Find All Imports | O(V + E) |
| Detect Cycles | O(V + E) |

---

### 1.2 Provider Registry

**Location**: `packages/core/injector/module.ts`

**Structure**: Multiple hash maps for different provider types

```plantuml
@startuml DS_ProviderRegistry
skinparam classAttributeIconSize 0

class Module {
    -_providers: Map<InjectionToken, InstanceWrapper>
    -_controllers: Map<InjectionToken, InstanceWrapper>
    -_injectables: Map<InjectionToken, InstanceWrapper>
    -_middlewares: Map<InjectionToken, InstanceWrapper>
    --
    +getProviderByKey(token): InstanceWrapper
    +hasProvider(token): boolean
    +addProvider(provider): string
}

class InstanceWrapper<T> {
    +token: InjectionToken
    +metatype: Type<T>
    +inject: InjectorDependency[]
    -values: WeakMap<ContextId, InstancePerContext<T>>
    -transientMap: WeakMap<ContextId, WeakMap<InstanceWrapper, T>>
}

note right of Module
  Hash Map Properties:
  - O(1) average lookup
  - Token as key (string, symbol, or class)
  - Supports custom injection tokens
end note

@enduml
```

**Complexity**:
| Operation | Time Complexity |
|-----------|-----------------|
| Add Provider | O(1) average |
| Get Provider | O(1) average |
| Has Provider | O(1) average |
| List All Providers | O(n) |

---

### 1.3 Instance Cache (Scoped Instances)

**Location**: `packages/core/injector/instance-wrapper.ts`

**Structure**: WeakMap for context-based instance storage

```plantuml
@startuml DS_InstanceCache
skinparam classAttributeIconSize 0

class InstanceWrapper<T> {
    -values: WeakMap<ContextId, InstancePerContext<T>>
    -transientMap: WeakMap<ContextId, WeakMap<InstanceWrapper, T>>
    --
    +getInstanceByContextId(contextId, inquirerId?): InstancePerContext<T>
    +setInstanceByContextId(contextId, value, inquirerId?): void
    +isDependencyTreeStatic(): boolean
}

class ContextId {
    +id: number
    +{static} create(): ContextId
}

class InstancePerContext<T> {
    +instance: T
    +isResolved: boolean
    +donePromise?: Promise<void>
}

InstanceWrapper --> ContextId : key
InstanceWrapper --> InstancePerContext : value

note right of InstanceWrapper
  WeakMap Benefits:
  - Automatic garbage collection
  - Context isolation
  - Memory-efficient for request scope
end note

@enduml
```

**Scoping Strategy**:
```
STATIC_CONTEXT (id=1) → Singleton instances
REQUEST_CONTEXT → Per-request instances (auto-GC when request ends)
TRANSIENT_CONTEXT → Per-injection instances (requires inquirerId)
```

---

## 2. Core Algorithms

### 2.1 Module Distance Calculation

**Location**: `packages/core/scanner.ts`

**Purpose**: Calculate topological depth of each module from root

```plantuml
@startuml Algo_ModuleDistance
start

:Initialize: Set all modules distance = UNREACHABLE;
:Set root module distance = 0;
:Create queue with root module;

while (Queue not empty?) is (yes)
    :Dequeue current module;
    
    partition "Process Imports" {
        :Get all imported modules;
        
        while (More imports?) is (yes)
            :Get next imported module;
            
            if (importedModule.distance > currentModule.distance + 1?) then (yes)
                :Update: importedModule.distance = currentModule.distance + 1;
                :Enqueue imported module;
            endif
        endwhile (no)
    }
endwhile (no)

:Return module distances;
stop

note right
  BFS traversal ensures:
  - Minimum distance calculated
  - Global modules get distance based on first encounter
  - Used for lifecycle hook ordering
end note

@enduml
```

**Pseudocode**:
```
ALGORITHM CalculateModuleDistance(rootModule)
    INPUT: rootModule - the application's entry module
    OUTPUT: All modules have correct distance values

    FOR each module IN allModules DO
        module.distance = UNREACHABLE
    END FOR
    
    rootModule.distance = 0
    queue = new Queue()
    queue.enqueue(rootModule)
    
    WHILE queue is not empty DO
        current = queue.dequeue()
        
        FOR each importedModule IN current.imports DO
            IF importedModule.distance > current.distance + 1 THEN
                importedModule.distance = current.distance + 1
                queue.enqueue(importedModule)
            END IF
        END FOR
    END WHILE
```

**Complexity**: O(V + E) where V = modules, E = import relationships

---

### 2.2 Dependency Resolution (Topological Sort)

**Location**: `packages/core/injector/injector.ts`

**Purpose**: Resolve provider dependencies in correct order

```plantuml
@startuml Algo_DependencyResolution
start

:Input: InstanceWrapper to resolve;

if (Already resolved?) then (yes)
    :Return cached instance;
    stop
endif

if (Is resolving? (circular)) then (yes)
    if (Has forward reference?) then (yes)
        :Create lazy proxy;
    else (no)
        :Throw CircularDependencyException;
        stop
    endif
endif

:Mark as "resolving";
:Get constructor dependencies from metadata;

partition "Resolve Dependencies" {
    :dependencies = [];
    
    while (More params?) is (yes)
        :Get param token;
        
        if (Token is undefined?) then (yes)
            :Throw UndefinedDependencyException;
            stop
        endif
        
        :Find provider wrapper;
        
        if (Not found in module?) then (yes)
            :Search in imported modules;
            
            if (Still not found?) then (yes)
                if (@Optional()?) then (yes)
                    :Use undefined;
                else (no)
                    :Throw UnknownDependenciesException;
                    stop
                endif
            endif
        endif
        
        :Recursively resolve dependency;
        :Add to dependencies[];
    endwhile (no)
}

:Instantiate class with dependencies;
:Mark as "resolved";
:Cache instance by contextId;
:Return instance;
stop

@enduml
```

**Pseudocode**:
```
ALGORITHM ResolveDependencies(wrapper, module, contextId)
    INPUT: wrapper - InstanceWrapper to resolve
           module - containing Module
           contextId - scope context
    OUTPUT: Resolved instance

    IF wrapper.isResolved(contextId) THEN
        RETURN wrapper.getInstance(contextId)
    END IF
    
    IF wrapper.isResolving THEN
        IF hasForwardReference(wrapper) THEN
            RETURN createLazyProxy(wrapper)
        ELSE
            THROW CircularDependencyException
        END IF
    END IF
    
    wrapper.markAsResolving()
    
    dependencies = []
    params = Reflect.getMetadata('design:paramtypes', wrapper.metatype)
    injectTokens = Reflect.getMetadata(INJECT_METADATA, wrapper.metatype)
    
    FOR i = 0 TO params.length - 1 DO
        token = injectTokens[i] OR params[i]
        
        IF token IS undefined THEN
            THROW UndefinedDependencyException(i)
        END IF
        
        dependencyWrapper = lookupProvider(token, module)
        
        IF dependencyWrapper IS null THEN
            IF hasOptionalDecorator(i) THEN
                dependencies[i] = undefined
            ELSE
                THROW UnknownDependenciesException(token)
            END IF
        ELSE
            dependencies[i] = ResolveDependencies(dependencyWrapper, module, contextId)
        END IF
    END FOR
    
    instance = new wrapper.metatype(...dependencies)
    wrapper.setInstance(instance, contextId)
    wrapper.markAsResolved()
    
    RETURN instance
```

**Complexity**: O(V + E) for DAG traversal, where V = providers, E = dependencies

---

### 2.3 Provider Lookup Algorithm

**Location**: `packages/core/injector/injector.ts`

**Purpose**: Find provider across module hierarchy

```plantuml
@startuml Algo_ProviderLookup
start

:Input: token, searchModule, requestingWrapper;

partition "Search Local Module" {
    :Search in module.providers;
    
    if (Found?) then (yes)
        :Return wrapper;
        stop
    endif
}

partition "Search Imported Modules" {
    :Get all imports of searchModule;
    
    while (More imports?) is (yes)
        :Get next importedModule;
        
        if (Token in importedModule.exports?) then (yes)
            :Search in importedModule.providers;
            
            if (Found?) then (yes)
                :Return wrapper;
                stop
            endif
        endif
    endwhile (no)
}

partition "Search Global Modules" {
    :Get all global modules;
    
    while (More global modules?) is (yes)
        :Get next globalModule;
        
        if (Token in globalModule.exports?) then (yes)
            :Search in globalModule.providers;
            
            if (Found?) then (yes)
                :Return wrapper;
                stop
            endif
        endif
    endwhile (no)
}

:Return null (not found);
stop

@enduml
```

**Pseudocode**:
```
ALGORITHM LookupProvider(token, module, wrapper)
    INPUT: token - injection token
           module - starting Module
           wrapper - requesting InstanceWrapper (for error context)
    OUTPUT: InstanceWrapper or null

    // Step 1: Search local module
    IF module.providers.has(token) THEN
        RETURN module.providers.get(token)
    END IF
    
    // Step 2: Search imported modules (respect exports)
    FOR each importedModule IN module.imports DO
        IF importedModule.exports.has(token) THEN
            IF importedModule.providers.has(token) THEN
                RETURN importedModule.providers.get(token)
            END IF
            // May need recursive search for re-exported providers
            result = LookupInExportedModules(token, importedModule)
            IF result IS NOT null THEN
                RETURN result
            END IF
        END IF
    END FOR
    
    // Step 3: Search global modules
    FOR each globalModule IN container.globalModules DO
        IF globalModule.exports.has(token) THEN
            IF globalModule.providers.has(token) THEN
                RETURN globalModule.providers.get(token)
            END IF
        END IF
    END FOR
    
    RETURN null
```

**Complexity**: O(M × P) worst case, where M = modules, P = average providers per module

---

### 2.4 Guard Chain Execution

**Location**: `packages/core/guards/guards-consumer.ts`

**Purpose**: Execute guard chain and short-circuit on failure

```plantuml
@startuml Algo_GuardChain
start

:Input: guards[], ExecutionContext;

:result = true;

while (More guards? AND result = true) is (yes)
    :Get next guard;
    :Call guard.canActivate(context);
    
    if (Returns Promise?) then (yes)
        :result = await promise;
    else if (Returns Observable?) then (yes)
        :result = await firstValueFrom(observable);
    else (sync)
        :result = guard result;
    endif
    
    if (result = false?) then (yes)
        :Throw ForbiddenException;
        stop
    endif
endwhile (no)

:Return true;
stop

note right
  Short-circuit evaluation:
  - First false/rejection stops chain
  - Supports sync, Promise, Observable returns
  - Guards executed in order of specificity
end note

@enduml
```

**Pseudocode**:
```
ALGORITHM ExecuteGuardChain(guards, context)
    INPUT: guards - array of CanActivate instances
           context - ExecutionContext
    OUTPUT: true if all guards pass, throws otherwise

    FOR each guard IN guards DO
        result = guard.canActivate(context)
        
        IF result IS Promise THEN
            result = AWAIT result
        ELSE IF result IS Observable THEN
            result = AWAIT firstValueFrom(result)
        END IF
        
        IF result ≠ true THEN
            THROW ForbiddenException
        END IF
    END FOR
    
    RETURN true
```

**Complexity**: O(n) where n = number of guards

---

### 2.5 Interceptor Chain (RxJS Pipeline)

**Location**: `packages/core/interceptors/interceptors-consumer.ts`

**Purpose**: Build and execute interceptor chain using RxJS

```plantuml
@startuml Algo_InterceptorChain
start

:Input: interceptors[], context, next (handler);

if (No interceptors?) then (yes)
    :Return next.handle();
    stop
endif

:Create nextFn pointing to actual handler;
:Reverse iterate interceptors;

partition "Build Chain" {
    while (More interceptors?) is (yes)
        :Get interceptor (from end);
        :Wrap: currentFn = () => interceptor.intercept(context, { handle: nextFn });
        :nextFn = currentFn;
    endwhile (no)
}

:Execute first interceptor (starts chain);
:Return resulting Observable;
stop

note right
  Chain builds inside-out:
  
  InterceptorA.intercept(ctx, {
    handle: () => InterceptorB.intercept(ctx, {
      handle: () => handler()
    })
  })
end note

@enduml
```

**Pseudocode**:
```
ALGORITHM BuildInterceptorChain(interceptors, context, handler)
    INPUT: interceptors - array of NestInterceptor
           context - ExecutionContext
           handler - final request handler
    OUTPUT: Observable<any>

    IF interceptors.length = 0 THEN
        RETURN defer(() => transformValue(handler()))
    END IF
    
    // Build chain from last to first
    nextFn = () => defer(() => transformValue(handler()))
    
    FOR i = interceptors.length - 1 DOWNTO 0 DO
        interceptor = interceptors[i]
        currentNextFn = nextFn
        
        nextFn = () => interceptor.intercept(context, {
            handle: () => currentNextFn()
        })
    END FOR
    
    RETURN nextFn()
```

**Complexity**: O(n) to build chain, execution depends on interceptor logic

---

### 2.6 Route Matching Algorithm

**Location**: `packages/core/router/router-explorer.ts`

**Purpose**: Match incoming request to handler

```plantuml
@startuml Algo_RouteMatching
start

:Input: request (method, path);

:Normalize path (remove trailing slash);
:Get global prefix;
:Build full path with versioning;

partition "Find Matching Route" {
    :Get routes for HTTP method;
    
    while (More routes?) is (yes)
        :Get next route definition;
        
        if (Is parameterized route?) then (yes)
            :Convert to regex pattern;
            :Extract param names;
        endif
        
        if (Path matches pattern?) then (yes)
            if (Has parameters?) then (yes)
                :Extract param values;
                :Attach to request.params;
            endif
            :Return matched handler;
            stop
        endif
    endwhile (no)
}

:Return 404 handler;
stop

@enduml
```

**Route Types**:
- Static: `/users` - exact match
- Parameterized: `/users/:id` - pattern match
- Wildcard: `/users/*` - prefix match
- Regex: `/users/:id(\\d+)` - custom pattern

**Complexity**: O(R) where R = number of routes for the HTTP method

---

### 2.7 Module Compilation (Dynamic Modules)

**Location**: `packages/core/injector/module-compiler.ts`

**Purpose**: Compile module class into factory with resolved dynamic metadata

```plantuml
@startuml Algo_ModuleCompilation
start

:Input: module (Type or DynamicModule);

if (Is DynamicModule?) then (yes)
    :Extract module class from 'module' property;
    :Extract dynamic metadata (providers, imports, etc.);
else (no)
    :Use class directly;
    :dynamicMetadata = {};
endif

:Get static metadata from @Module() decorator;
:Merge static and dynamic metadata;

partition "Generate Token" {
    if (Using DeepHashedKeyFactory?) then (yes)
        :Hash module class + dynamic metadata;
        :token = hash result;
    else (ByReference)
        :token = module class reference;
    endif
}

:Create ModuleFactory;
:Return { type, dynamicMetadata, token };
stop

note right
  Dynamic modules allow:
  - Runtime configuration
  - Async factories (forRootAsync)
  - Different instances per import
end note

@enduml
```

**Pseudocode**:
```
ALGORITHM CompileModule(moduleClass)
    INPUT: moduleClass - Type<any> or DynamicModule
    OUTPUT: ModuleFactory

    IF isDynamicModule(moduleClass) THEN
        type = moduleClass.module
        dynamicMetadata = {
            imports: moduleClass.imports,
            exports: moduleClass.exports,
            providers: moduleClass.providers,
            controllers: moduleClass.controllers,
        }
    ELSE
        type = moduleClass
        dynamicMetadata = {}
    END IF
    
    staticMetadata = Reflect.getMetadata(MODULE_METADATA, type)
    mergedMetadata = merge(staticMetadata, dynamicMetadata)
    
    token = moduleOpaqueKeyFactory.create(
        getModuleId(type),
        type.name,
        dynamicMetadata
    )
    
    RETURN { type, dynamicMetadata, token }
```

---

## 3. Complexity Summary

| Algorithm | Time | Space | Location |
|-----------|------|-------|----------|
| Module Distance | O(V+E) | O(V) | scanner.ts |
| Dependency Resolution | O(V+E) | O(V) | injector.ts |
| Provider Lookup | O(M×P) | O(1) | injector.ts |
| Guard Chain | O(n) | O(1) | guards-consumer.ts |
| Interceptor Chain | O(n) | O(n) | interceptors-consumer.ts |
| Route Matching | O(R) | O(1) | router-explorer.ts |
| Module Compilation | O(P) | O(P) | module-compiler.ts |

**Legend**:
- V = vertices (modules/providers)
- E = edges (imports/dependencies)
- M = number of modules
- P = average providers per module
- n = number of enhancers
- R = number of routes

---

## 4. Data Flow Diagrams

### 4.1 Request Processing Data Flow

```plantuml
@startuml DF_RequestProcessing
!define RECTANGLE class

skinparam rectangle {
    BackgroundColor<<datastore>> LightBlue
    BackgroundColor<<process>> LightGreen
    BackgroundColor<<external>> LightGray
}

rectangle "HTTP Server" as http <<external>>
rectangle "Route Matcher" as router <<process>>
rectangle "Middleware Chain" as middleware <<process>>
rectangle "Guard Chain" as guards <<process>>
rectangle "Interceptor Chain (pre)" as interceptors_pre <<process>>
rectangle "Pipe Chain" as pipes <<process>>
rectangle "Handler" as handler <<process>>
rectangle "Interceptor Chain (post)" as interceptors_post <<process>>
rectangle "Exception Filter" as exception <<process>>

rectangle "Route Table" as routeTable <<datastore>>
rectangle "Provider Registry" as providers <<datastore>>
rectangle "Metadata Store" as metadata <<datastore>>

http --> router : Request
router --> routeTable : Lookup
router --> middleware : Matched Route
middleware --> guards : Modified Request
guards --> metadata : Get Guards
guards --> interceptors_pre : Authorized Request
interceptors_pre --> providers : Get Interceptors
interceptors_pre --> pipes : Observable Chain
pipes --> metadata : Get Pipes
pipes --> handler : Transformed Args
handler --> providers : Resolve Dependencies
handler --> interceptors_post : Result
interceptors_post --> http : Response
exception --> http : Error Response

note right of routeTable
  Contains:
  - Path patterns
  - HTTP methods
  - Handler references
end note

@enduml
```

### 4.2 Dependency Injection Data Flow

```plantuml
@startuml DF_DependencyInjection
!define RECTANGLE class

skinparam rectangle {
    BackgroundColor<<datastore>> LightBlue
    BackgroundColor<<process>> LightGreen
}

rectangle "Module Scanner" as scanner <<process>>
rectangle "Module Compiler" as compiler <<process>>
rectangle "Container" as container <<datastore>>
rectangle "Injector" as injector <<process>>
rectangle "Instance Loader" as loader <<process>>
rectangle "Metadata Reader" as metadata <<process>>

rectangle "Module Registry" as modules <<datastore>>
rectangle "Provider Registry" as providers <<datastore>>
rectangle "Instance Cache" as cache <<datastore>>

scanner --> compiler : Module Classes
compiler --> container : Compiled Modules
container --> modules : Store
container --> providers : Register Providers

injector --> providers : Lookup Provider
injector --> metadata : Read @Inject
injector --> cache : Check Cache
injector --> loader : Load Instance

loader --> cache : Store Instance

note bottom of cache
  Cache Structure:
  - STATIC_CONTEXT → Singletons
  - REQUEST_CONTEXT → Per-request
  - TRANSIENT_CONTEXT → Per-injection
end note

@enduml
```

---

## 5. Performance Considerations

### 5.1 Caching Strategies

1. **Instance Caching**: WeakMap by ContextId prevents re-resolution
2. **Metadata Caching**: Decorator metadata cached by Reflect
3. **Route Caching**: Compiled routes stored at bootstrap

### 5.2 Lazy Loading

1. **Lazy Modules**: Modules can be loaded on-demand via `LazyModuleLoader`
2. **Forward References**: `forwardRef()` enables lazy dependency resolution
3. **Deferred Instantiation**: Transient providers only instantiated when requested

### 5.3 Memory Management

1. **WeakMap for Request Scope**: Auto-GC when request completes
2. **Module Tree Pruning**: Unused imports can be tree-shaken
3. **Singleton Pattern**: Reduces memory footprint for stateless services

---

## 6. References

- [Introduction to Algorithms (CLRS)](https://mitpress.mit.edu/books/introduction-algorithms-third-edition)
- [NestJS Fundamentals](https://docs.nestjs.com/fundamentals)
- [TypeScript Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html)
