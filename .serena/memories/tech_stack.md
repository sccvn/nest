# NestJS Tech Stack

## Core Technologies

### Runtime & Language
| Technology | Version | Purpose |
|------------|---------|---------|
| Node.js | >= 20 | Runtime environment |
| TypeScript | 5.9.x | Primary language |
| ES Target | ES2021 | Compilation target |

### Build System
| Tool | Purpose |
|------|---------|
| TypeScript Compiler (tsc) | Main compilation |
| Lerna | Monorepo management |
| Gulp | Build tasks, file operations |
| Husky | Git hooks |
| lint-staged | Pre-commit linting |

### Testing
| Tool | Purpose |
|------|---------|
| Mocha | Test framework |
| Chai | Assertion library |
| Sinon | Mocking/stubbing |
| NYC | Code coverage |
| Supertest | HTTP integration testing |
| Docker Compose | Integration test infrastructure |

### Code Quality
| Tool | Purpose |
|------|---------|
| ESLint | Linting |
| Prettier | Code formatting |
| typescript-eslint | TypeScript-specific linting |

## Framework Dependencies

### Core Runtime Dependencies
```json
{
  "rxjs": "7.8.x",           // Reactive programming
  "reflect-metadata": "0.2.x", // Metadata reflection
  "iterare": "1.2.x",        // Iteration utilities
  "path-to-regexp": "8.x",   // Route path matching
  "tslib": "2.8.x",          // TypeScript helpers
  "uid": "2.0.x",            // Unique ID generation
  "uuid": "13.x",            // UUID generation
  "fast-safe-stringify": "2.1.x" // Safe JSON stringify
}
```

### HTTP Platforms
```json
{
  "express": "5.x",          // Default HTTP platform
  "fastify": "5.x",          // Alternative HTTP platform
  "cors": "2.8.x"            // CORS middleware
}
```

### WebSockets
```json
{
  "socket.io": "4.8.x",      // Socket.io adapter
  "ws": "8.x"                // WS adapter
}
```

### Validation
```json
{
  "class-transformer": "0.5.x",
  "class-validator": "0.14.x"
}
```

### Microservices (Optional)
```json
{
  "@grpc/grpc-js": "1.x",    // gRPC
  "kafkajs": "2.x",          // Kafka
  "amqplib": "0.10.x",       // RabbitMQ
  "nats": "2.x",             // NATS
  "mqtt": "5.x",             // MQTT
  "ioredis": "5.x"           // Redis
}
```

### Database Integrations (in samples/integration)
```json
{
  "typeorm": "0.3.x",        // SQL ORM
  "mongoose": "9.x",         // MongoDB ODM
  "mysql2": "3.x",           // MySQL driver
  "redis": "5.x"             // Redis client
}
```

### GraphQL (in samples/integration)
```json
{
  "graphql": "16.x",
  "@apollo/server": "5.x",
  "@nestjs/graphql": "13.x",
  "@nestjs/apollo": "13.x"
}
```

## Architecture Patterns

### Design Patterns Used
- **Dependency Injection**: Core pattern, similar to Angular
- **Decorator Pattern**: Extensive use of decorators
- **Module Pattern**: Modular architecture
- **Factory Pattern**: NestFactory for application creation
- **Adapter Pattern**: Platform adapters (Express, Fastify)
- **Interceptor Pattern**: Request/response transformation
- **Guard Pattern**: Route protection
- **Pipe Pattern**: Data transformation and validation
- **Filter Pattern**: Exception handling

### Key Concepts
1. **Modules**: Organizational units (`@Module`)
2. **Controllers**: Handle HTTP requests (`@Controller`)
3. **Providers/Services**: Business logic (`@Injectable`)
4. **Middleware**: Request processing pipeline
5. **Guards**: Authorization (`@UseGuards`)
6. **Interceptors**: Cross-cutting concerns (`@UseInterceptors`)
7. **Pipes**: Transformation/validation (`@UsePipes`)
8. **Exception Filters**: Error handling (`@Catch`)
