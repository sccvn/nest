# Architecture Patterns Expertise - NestJS Core Framework Focus

## Overview
This prompt provides comprehensive expertise in software architecture patterns, design patterns, and architectural decision-making for building robust, scalable systems, **with specific focus on the NestJS core framework architecture**.

**Special Context**: This is applied to the **NestJS monorepo**, which implements:
- **Modular Architecture**: Package-based boundaries (`common`, `core`, `microservices`, etc.)
- **Platform Adapter Pattern**: Abstraction over Express, Fastify, Socket.io, ws
- **Dependency Injection Container**: Hierarchical module-scoped IoC
- **Decorator Pattern**: Metadata-driven declarative programming
- **Monorepo Structure**: Lerna-managed independent packages with shared configurations

## Architectural Styles

### Layered Architecture
```
┌─────────────────────────────────────┐
│       Presentation Layer            │  Controllers, DTOs, Views
├─────────────────────────────────────┤
│       Application Layer             │  Services, Use Cases
├─────────────────────────────────────┤
│         Domain Layer                │  Entities, Value Objects, Domain Services
├─────────────────────────────────────┤
│      Infrastructure Layer           │  Repositories, External Services
└─────────────────────────────────────┘
```

#### NestJS Implementation
```typescript
// Presentation Layer - Controller
@Controller('orders')
export class OrdersController {
  constructor(private readonly ordersService: OrdersService) {}

  @Post()
  async create(@Body() dto: CreateOrderDto): Promise<OrderResponseDto> {
    const order = await this.ordersService.create(dto);
    return plainToClass(OrderResponseDto, order);
  }
}

// Application Layer - Service
@Injectable()
export class OrdersService {
  constructor(
    private readonly orderRepository: IOrderRepository,
    private readonly inventoryService: IInventoryService,
  ) {}

  async create(dto: CreateOrderDto): Promise<Order> {
    const order = Order.create(dto);
    await this.inventoryService.reserve(order.items);
    return this.orderRepository.save(order);
  }
}

// Domain Layer - Entity
export class Order {
  private constructor(
    public readonly id: string,
    public readonly items: OrderItem[],
    public readonly status: OrderStatus,
  ) {}

  static create(dto: CreateOrderDto): Order {
    return new Order(uuid(), dto.items.map(OrderItem.create), OrderStatus.PENDING);
  }

  confirm(): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new InvalidOrderStateException();
    }
    this.status = OrderStatus.CONFIRMED;
  }
}

// Infrastructure Layer - Repository
@Injectable()
export class TypeOrmOrderRepository implements IOrderRepository {
  constructor(
    @InjectRepository(OrderEntity)
    private readonly repo: Repository<OrderEntity>,
  ) {}

  async save(order: Order): Promise<Order> {
    const entity = this.toEntity(order);
    const saved = await this.repo.save(entity);
    return this.toDomain(saved);
  }
}
```

### Hexagonal Architecture (Ports & Adapters)

```
              ┌──────────────────────────────────────┐
              │                                      │
   Driving    │    ┌────────────────────────┐       │    Driven
   Adapters   │    │                        │       │    Adapters
              │    │      Application       │       │
┌──────────┐  │    │         Core           │       │  ┌──────────┐
│   REST   │──┼───►│    ┌────────────┐     │◄──────┼──│ Database │
│   API    │  │    │    │   Domain   │     │       │  │          │
└──────────┘  │    │    │   Model    │     │       │  └──────────┘
              │    │    └────────────┘     │       │
┌──────────┐  │    │                        │       │  ┌──────────┐
│   gRPC   │──┼───►│       Use Cases        │◄──────┼──│ Message  │
│          │  │    │                        │       │  │  Queue   │
└──────────┘  │    └────────────────────────┘       │  └──────────┘
              │             ▲       ▲                │
              │     Ports   │       │   Ports       │
              └──────────────────────────────────────┘
```

#### Implementation Structure
```
src/
├── domain/                    # Core domain (no external dependencies)
│   ├── entities/
│   │   └── order.entity.ts
│   ├── value-objects/
│   │   └── money.vo.ts
│   ├── services/
│   │   └── order-calculator.service.ts
│   └── events/
│       └── order-created.event.ts
│
├── application/               # Use cases and ports
│   ├── ports/
│   │   ├── in/               # Driving ports (interfaces for use cases)
│   │   │   └── create-order.port.ts
│   │   └── out/              # Driven ports (interfaces for adapters)
│   │       ├── order-repository.port.ts
│   │       └── payment-service.port.ts
│   └── use-cases/
│       └── create-order.use-case.ts
│
├── infrastructure/            # Adapters
│   ├── adapters/
│   │   ├── in/               # Driving adapters
│   │   │   ├── rest/
│   │   │   │   └── orders.controller.ts
│   │   │   └── grpc/
│   │   │       └── orders.grpc.controller.ts
│   │   └── out/              # Driven adapters
│   │       ├── persistence/
│   │       │   └── typeorm-order.repository.ts
│   │       └── external/
│   │           └── stripe-payment.service.ts
│   └── config/
│       └── database.config.ts
```

### Microservices Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
└─────────────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
    │  User   │   │  Order  │   │ Product │   │ Payment │
    │ Service │   │ Service │   │ Service │   │ Service │
    └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
    │  User   │   │  Order  │   │ Product │   │ Payment │
    │   DB    │   │   DB    │   │   DB    │   │   DB    │
    └─────────┘   └─────────┘   └─────────┘   └─────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  Message Broker │
              │   (RabbitMQ)    │
              └─────────────────┘
```

#### Service Communication Patterns

```typescript
// Synchronous - REST/HTTP
@Injectable()
export class OrderService {
  constructor(private readonly httpService: HttpService) {}

  async getProductDetails(productId: string): Promise<Product> {
    const { data } = await this.httpService.axiosRef.get(
      `http://product-service/products/${productId}`
    );
    return data;
  }
}

// Synchronous - gRPC
@Injectable()
export class OrderService {
  @Client({
    transport: Transport.GRPC,
    options: { package: 'product', protoPath: 'product.proto' },
  })
  private client: ClientGrpc;
  private productService: ProductServiceClient;

  onModuleInit() {
    this.productService = this.client.getService<ProductServiceClient>('ProductService');
  }

  async getProduct(id: string): Promise<Product> {
    return firstValueFrom(this.productService.findOne({ id }));
  }
}

// Asynchronous - Message Queue
@Injectable()
export class OrderService {
  constructor(
    @Inject('RABBITMQ_SERVICE')
    private readonly client: ClientProxy,
  ) {}

  async createOrder(order: Order): Promise<void> {
    // Publish event
    this.client.emit('order_created', new OrderCreatedEvent(order));
  }
}

@Controller()
export class PaymentHandler {
  @EventPattern('order_created')
  async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    // Process payment
  }
}
```

### Event-Driven Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Order     │    │  Inventory  │    │  Shipping   │
│   Service   │    │   Service   │    │   Service   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       │ OrderCreated     │ StockReserved    │
       │     Event        │     Event        │
       ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────┐
│                    Event Bus                        │
│                  (Kafka/RabbitMQ)                   │
└─────────────────────────────────────────────────────┘
```

#### Event Sourcing
```typescript
// Event Store
interface Event {
  id: string;
  aggregateId: string;
  type: string;
  data: any;
  timestamp: Date;
  version: number;
}

// Aggregate with Event Sourcing
class OrderAggregate {
  private events: Event[] = [];
  private state: OrderState;

  apply(event: Event): void {
    switch (event.type) {
      case 'OrderCreated':
        this.state = { ...this.state, status: 'PENDING', items: event.data.items };
        break;
      case 'OrderConfirmed':
        this.state = { ...this.state, status: 'CONFIRMED' };
        break;
      case 'OrderShipped':
        this.state = { ...this.state, status: 'SHIPPED', trackingNumber: event.data.trackingNumber };
        break;
    }
    this.events.push(event);
  }

  static rehydrate(events: Event[]): OrderAggregate {
    const aggregate = new OrderAggregate();
    events.forEach(event => aggregate.apply(event));
    return aggregate;
  }
}
```

### CQRS (Command Query Responsibility Segregation)

```
┌──────────────────────────────────────────────────────────────────┐
│                          Client                                  │
└──────────────────────────────────────────────────────────────────┘
                    │                           │
            Commands│                           │Queries
                    ▼                           ▼
          ┌─────────────────┐         ┌─────────────────┐
          │ Command Handler │         │  Query Handler  │
          └────────┬────────┘         └────────┬────────┘
                   │                           │
                   ▼                           ▼
          ┌─────────────────┐         ┌─────────────────┐
          │   Write Model   │         │   Read Model    │
          │   (Normalized)  │─────────│ (Denormalized)  │
          └─────────────────┘  Sync   └─────────────────┘
```

#### NestJS CQRS Implementation
```typescript
// Command
export class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItemDto[],
  ) {}
}

// Command Handler
@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(
    private readonly orderRepository: IOrderRepository,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: CreateOrderCommand): Promise<string> {
    const order = Order.create(command.userId, command.items);
    await this.orderRepository.save(order);
    
    this.eventBus.publish(new OrderCreatedEvent(order.id, order.items));
    
    return order.id;
  }
}

// Query
export class GetOrderQuery {
  constructor(public readonly orderId: string) {}
}

// Query Handler
@QueryHandler(GetOrderQuery)
export class GetOrderHandler implements IQueryHandler<GetOrderQuery> {
  constructor(private readonly orderReadRepository: IOrderReadRepository) {}

  async execute(query: GetOrderQuery): Promise<OrderReadModel> {
    return this.orderReadRepository.findById(query.orderId);
  }
}

// Event Handler (Sync read model)
@EventsHandler(OrderCreatedEvent)
export class OrderCreatedHandler implements IEventHandler<OrderCreatedEvent> {
  constructor(private readonly orderReadRepository: IOrderReadRepository) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    const readModel = new OrderReadModel(
      event.orderId,
      event.items,
      'PENDING',
      new Date(),
    );
    await this.orderReadRepository.save(readModel);
  }
}
```

## Design Patterns

### Creational Patterns

#### Factory Pattern
```typescript
// Abstract Factory
interface DatabaseConnection {
  connect(): Promise<void>;
  query(sql: string): Promise<any>;
}

interface DatabaseFactory {
  createConnection(): DatabaseConnection;
}

class PostgresFactory implements DatabaseFactory {
  createConnection(): DatabaseConnection {
    return new PostgresConnection();
  }
}

class MongoFactory implements DatabaseFactory {
  createConnection(): DatabaseConnection {
    return new MongoConnection();
  }
}

// Usage with DI
@Module({
  providers: [
    {
      provide: 'DATABASE_FACTORY',
      useFactory: () => {
        const dbType = process.env.DB_TYPE;
        return dbType === 'postgres' ? new PostgresFactory() : new MongoFactory();
      },
    },
  ],
})
export class DatabaseModule {}
```

#### Builder Pattern
```typescript
class OrderBuilder {
  private order: Partial<Order> = {};

  withCustomer(customerId: string): OrderBuilder {
    this.order.customerId = customerId;
    return this;
  }

  withItems(items: OrderItem[]): OrderBuilder {
    this.order.items = items;
    return this;
  }

  withShipping(address: Address): OrderBuilder {
    this.order.shippingAddress = address;
    return this;
  }

  withDiscount(discount: Discount): OrderBuilder {
    this.order.discount = discount;
    return this;
  }

  build(): Order {
    this.validate();
    return new Order(this.order as OrderProps);
  }

  private validate(): void {
    if (!this.order.customerId) throw new Error('Customer required');
    if (!this.order.items?.length) throw new Error('Items required');
  }
}

// Usage
const order = new OrderBuilder()
  .withCustomer('cust-123')
  .withItems([item1, item2])
  .withShipping(address)
  .withDiscount(discount)
  .build();
```

### Structural Patterns

#### Adapter Pattern
```typescript
// External payment API (Stripe)
interface StripePayment {
  createCharge(amount: number, currency: string, source: string): Promise<StripeCharge>;
}

// Our internal payment interface
interface PaymentGateway {
  processPayment(payment: PaymentRequest): Promise<PaymentResult>;
}

// Adapter
class StripeAdapter implements PaymentGateway {
  constructor(private readonly stripe: StripePayment) {}

  async processPayment(payment: PaymentRequest): Promise<PaymentResult> {
    const charge = await this.stripe.createCharge(
      payment.amount,
      payment.currency,
      payment.sourceToken,
    );
    
    return {
      transactionId: charge.id,
      status: charge.status === 'succeeded' ? 'SUCCESS' : 'FAILED',
      amount: payment.amount,
    };
  }
}
```

#### Decorator Pattern
```typescript
// Base interface
interface DataService {
  getData(key: string): Promise<any>;
}

// Base implementation
class DatabaseService implements DataService {
  async getData(key: string): Promise<any> {
    // Fetch from database
    return this.db.find(key);
  }
}

// Caching decorator
class CachingDecorator implements DataService {
  constructor(
    private readonly wrapped: DataService,
    private readonly cache: CacheService,
  ) {}

  async getData(key: string): Promise<any> {
    const cached = await this.cache.get(key);
    if (cached) return cached;

    const data = await this.wrapped.getData(key);
    await this.cache.set(key, data);
    return data;
  }
}

// Logging decorator
class LoggingDecorator implements DataService {
  constructor(
    private readonly wrapped: DataService,
    private readonly logger: Logger,
  ) {}

  async getData(key: string): Promise<any> {
    this.logger.log(`Getting data for key: ${key}`);
    const result = await this.wrapped.getData(key);
    this.logger.log(`Got data for key: ${key}`);
    return result;
  }
}

// Usage - compose decorators
const service = new LoggingDecorator(
  new CachingDecorator(
    new DatabaseService(),
    cacheService,
  ),
  logger,
);
```

### Behavioral Patterns

#### Strategy Pattern
```typescript
// Strategy interface
interface PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number;
}

// Concrete strategies
class RegularPricing implements PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number {
    return basePrice * quantity;
  }
}

class BulkPricing implements PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number {
    const discount = quantity >= 100 ? 0.2 : quantity >= 50 ? 0.1 : 0;
    return basePrice * quantity * (1 - discount);
  }
}

class SeasonalPricing implements PricingStrategy {
  calculatePrice(basePrice: number, quantity: number): number {
    const isHoliday = this.checkHolidaySeason();
    const multiplier = isHoliday ? 1.25 : 1;
    return basePrice * quantity * multiplier;
  }
}

// Context
class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  calculate(basePrice: number, quantity: number): number {
    return this.strategy.calculatePrice(basePrice, quantity);
  }
}
```

#### Observer Pattern (with RxJS)
```typescript
// Subject - Observable
@Injectable()
export class OrderEventService {
  private orderCreated$ = new Subject<OrderCreatedEvent>();
  private orderUpdated$ = new Subject<OrderUpdatedEvent>();

  emitOrderCreated(event: OrderCreatedEvent): void {
    this.orderCreated$.next(event);
  }

  onOrderCreated(): Observable<OrderCreatedEvent> {
    return this.orderCreated$.asObservable();
  }
}

// Observer - Subscribe
@Injectable()
export class NotificationService implements OnModuleInit {
  constructor(private readonly orderEvents: OrderEventService) {}

  onModuleInit(): void {
    this.orderEvents.onOrderCreated().pipe(
      filter(event => event.order.total > 1000),
      map(event => ({
        type: 'HIGH_VALUE_ORDER',
        orderId: event.order.id,
      })),
    ).subscribe(notification => {
      this.sendNotification(notification);
    });
  }
}
```

## Architecture Decision Records (ADR)

### ADR Template
```markdown
# ADR-001: Use PostgreSQL for Primary Database

## Status
Accepted

## Context
We need to choose a primary database for the e-commerce platform.
Requirements:
- ACID compliance for financial transactions
- Support for complex queries
- Scalability to millions of records
- Strong ecosystem and tooling

## Decision
Use PostgreSQL as the primary database.

## Consequences
### Positive
- Strong ACID compliance
- Excellent query performance with proper indexing
- Rich feature set (JSON, full-text search, etc.)
- Large community and extensive documentation

### Negative
- Horizontal scaling is more complex than NoSQL alternatives
- Requires careful index management for performance

### Risks
- Single point of failure without proper replication setup
- Mitigation: Implement read replicas and automated failover

## Alternatives Considered
1. MySQL - Less feature-rich
2. MongoDB - Lacks ACID guarantees needed for transactions
3. CockroachDB - Less mature ecosystem
```

## Best Practices

### Architecture Selection Criteria
1. **Scalability requirements** - Monolith vs Microservices
2. **Team size and expertise** - Simpler is better for small teams
3. **Business domain complexity** - DDD for complex domains
4. **Performance requirements** - CQRS for read-heavy systems
5. **Consistency requirements** - Event Sourcing for audit trails

### Common Anti-Patterns to Avoid
1. **Big Ball of Mud** - No clear structure
2. **Distributed Monolith** - Microservices without proper boundaries
3. **Anemic Domain Model** - Logic in services, not entities
4. **Gold Plating** - Over-engineering for future needs
5. **Premature Optimization** - Optimizing before measuring

## References
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hexagonal Architecture - Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Domain-Driven Design - Eric Evans](https://domainlanguage.com/ddd/)
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
