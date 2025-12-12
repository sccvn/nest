# NestJS Architecture Guide

## Core Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    NestJS Application                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Module    │  │   Module    │  │   Module    │  ...    │
│  │  (Feature)  │  │  (Feature)  │  │   (Shared)  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
├─────────────────────────────────────────────────────────────┤
│                     NestFactory                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 NestApplication                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │  Container  │  │   Router    │  │   Config    │ │   │
│  │  │ (Injector)  │  │             │  │             │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│                   HTTP Adapter Layer                         │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  Express Adapter│  │  Fastify Adapter│                  │
│  └─────────────────┘  └─────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

## Request Lifecycle

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Controller → Service → Interceptors (after) → Exception Filters → Response
```

## Module Structure

```typescript
@Module({
  imports: [...],      // Other modules to import
  controllers: [...],  // Request handlers
  providers: [...],    // Services, repositories, etc.
  exports: [...],      // Providers available to other modules
})
export class FeatureModule {}
```

## Key Classes & Their Locations

### Entry Points
| Class | Location | Purpose |
|-------|----------|---------|
| `NestFactory` | `packages/core/nest-factory.ts` | Creates NestApplication |
| `NestFactoryStatic` | `packages/core/nest-factory.ts` | Factory implementation |

### Core Classes
| Class | Location | Purpose |
|-------|----------|---------|
| `NestApplication` | `packages/core/nest-application.ts` | Main application class |
| `NestApplicationContext` | `packages/core/nest-application-context.ts` | Base application context |
| `ApplicationConfig` | `packages/core/application-config.ts` | Global configuration |
| `NestContainer` | `packages/core/injector/container.ts` | DI container |
| `Scanner` | `packages/core/scanner.ts` | Module scanning |
| `MetadataScanner` | `packages/core/metadata-scanner.ts` | Metadata analysis |

### DI System (`packages/core/injector/`)
| Class | Purpose |
|-------|---------|
| `Container` | Main DI container |
| `Injector` | Dependency resolution |
| `Module` | Module wrapper |
| `InstanceWrapper` | Provider wrapper |
| `InstanceLoader` | Instance creation |

### Routing (`packages/core/router/`)
| Class | Purpose |
|-------|---------|
| `RoutesResolver` | Route registration |
| `RouterExplorer` | Route discovery |
| `RouterExecutionContext` | Request handling context |
| `RoutePathFactory` | Path generation |

## Decorators Architecture

### Controller Decorators (`packages/common/decorators/`)
```
decorators/
├── core/
│   ├── controller.decorator.ts    # @Controller
│   ├── injectable.decorator.ts    # @Injectable
│   ├── module.decorator.ts        # @Module
│   └── ...
├── http/
│   ├── route-params.decorator.ts  # @Body, @Param, @Query
│   ├── request-mapping.decorator.ts # @Get, @Post, @Put, @Delete
│   └── ...
└── modules/
    ├── global.decorator.ts        # @Global
    └── ...
```

## Lifecycle Hooks

```typescript
// Module lifecycle
interface OnModuleInit { onModuleInit(): any; }
interface OnModuleDestroy { onModuleDestroy(): any; }

// Application lifecycle  
interface OnApplicationBootstrap { onApplicationBootstrap(): any; }
interface OnApplicationShutdown { onApplicationShutdown(signal?: string): any; }
interface BeforeApplicationShutdown { beforeApplicationShutdown(signal?: string): any; }
```

## Error Handling Architecture

```
packages/core/exceptions/
├── base-exception-filter.ts       # Default exception handler
├── exceptions-handler.ts          # Exception processing
└── external-exceptions-handler.ts # External context handling

packages/common/exceptions/
├── http.exception.ts              # Base HTTP exception
├── bad-request.exception.ts       # 400
├── unauthorized.exception.ts      # 401
├── forbidden.exception.ts         # 403
├── not-found.exception.ts         # 404
└── ...                            # Other HTTP status exceptions
```

## Extension Points

1. **Custom Decorators**: Create metadata using `SetMetadata()`
2. **Custom Pipes**: Implement `PipeTransform`
3. **Custom Guards**: Implement `CanActivate`
4. **Custom Interceptors**: Implement `NestInterceptor`
5. **Custom Filters**: Use `@Catch()` decorator
6. **Custom Middleware**: Implement `NestMiddleware`
7. **Custom Adapters**: Extend `AbstractHttpAdapter`
