# NestJS Repository Structure

## Root Level Structure
```
nest/
├── packages/           # Core NestJS packages (monorepo)
├── integration/        # Integration tests
├── sample/             # Example applications (35+)
├── benchmarks/         # Performance benchmarks
├── scripts/            # Build and automation scripts
├── tools/              # Build tools (gulp tasks, benchmarks)
├── hooks/              # Mocha test hooks
```

## Packages Directory (`packages/`)
The monorepo contains the following packages (all published as `@nestjs/*`):

| Package | Directory | Description |
|---------|-----------|-------------|
| `@nestjs/common` | `packages/common/` | Common utilities, decorators, pipes, guards, interfaces |
| `@nestjs/core` | `packages/core/` | Core framework, dependency injection, nest-factory |
| `@nestjs/microservices` | `packages/microservices/` | Microservices support (Redis, MQTT, NATS, etc.) |
| `@nestjs/platform-express` | `packages/platform-express/` | Express HTTP adapter |
| `@nestjs/platform-fastify` | `packages/platform-fastify/` | Fastify HTTP adapter |
| `@nestjs/platform-socket.io` | `packages/platform-socket.io/` | Socket.io WebSocket adapter |
| `@nestjs/platform-ws` | `packages/platform-ws/` | WS WebSocket adapter |
| `@nestjs/testing` | `packages/testing/` | Testing utilities |
| `@nestjs/websockets` | `packages/websockets/` | WebSocket support |

## Core Package Structure (`packages/core/`)
```
packages/core/
├── adapters/          # HTTP adapters abstraction
├── discovery/         # Module discovery service
├── errors/            # Error classes
├── exceptions/        # Exception filters
├── guards/            # Guards context
├── helpers/           # Utility helpers
├── hooks/             # Lifecycle hooks
├── injector/          # Dependency injection container
├── inspector/         # Graph inspector
├── interceptors/      # Interceptors context
├── interfaces/        # TypeScript interfaces
├── middleware/        # Middleware support
├── pipes/             # Pipes context
├── repl/              # REPL functionality
├── router/            # HTTP routing
├── services/          # Core services (Reflector)
├── test/              # Unit tests
├── nest-factory.ts    # Main entry point - NestFactory
├── nest-application.ts # NestApplication class
└── index.ts           # Package exports
```

## Common Package Structure (`packages/common/`)
```
packages/common/
├── decorators/        # All decorators (@Controller, @Injectable, @Get, etc.)
├── enums/             # Enumerations (HttpStatus, RequestMethod, etc.)
├── exceptions/        # HTTP exceptions (HttpException, etc.)
├── file-stream/       # File streaming utilities
├── interfaces/        # TypeScript interfaces
├── module-utils/      # Module utilities
├── pipes/             # Built-in pipes (ValidationPipe, etc.)
├── serializer/        # Serialization
├── services/          # Services (Logger, Console)
├── test/              # Unit tests
├── utils/             # Utility functions
├── constants.ts       # Framework constants
└── index.ts           # Package exports
```

## Sample Applications (`sample/`)
35+ example applications demonstrating various features:
- 01-cats-app: Basic REST API
- 02-gateways: WebSocket gateways
- 03-microservices: Microservices patterns
- 04-grpc: gRPC integration
- 05-sql-typeorm: TypeORM database
- 06-mongoose: MongoDB with Mongoose
- 11-swagger: OpenAPI documentation
- 12-graphql-schema-first: GraphQL (schema first)
- 23-graphql-code-first: GraphQL (code first)
- And many more...

## Integration Tests (`integration/`)
E2E tests for various features with Docker support:
- cors, discovery, graphql-*, hello-world, hooks
- injector, inspector, lazy-modules, microservices
- mongoose, nest-application, repl, scopes
- send-files, testing-module-override, typeorm
- versioning, websockets
