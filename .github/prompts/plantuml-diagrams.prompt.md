---
name: plantuml-diagrams
description: Provides expertise in creating UML diagrams using PlantUML syntax, including C4 model diagrams, sequence diagrams, class diagrams, ERD diagrams, state machines, and more.
model: Claude Opus 4.5 (Preview) (copilot)
agent: senior-software-architect
---
# PlantUML Diagrams Expertise

## Overview
This prompt provides comprehensive expertise in creating UML diagrams using PlantUML syntax, including C4 model diagrams, sequence diagrams, class diagrams, ERD diagrams, state machines, and more.

## C4 Model Diagrams

### C4 Context Diagram
```plantuml
@startuml C4_Context
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

LAYOUT_WITH_LEGEND()

title System Context Diagram - E-Commerce Platform

Person(customer, "Customer", "A user who wants to buy products")
Person(admin, "Admin", "System administrator")

System(ecommerce, "E-Commerce Platform", "Allows customers to browse and purchase products")

System_Ext(payment, "Payment Gateway", "Processes payments")
System_Ext(shipping, "Shipping Service", "Handles delivery")
System_Ext(email, "Email Service", "Sends notifications")

Rel(customer, ecommerce, "Uses", "HTTPS")
Rel(admin, ecommerce, "Manages", "HTTPS")
Rel(ecommerce, payment, "Processes payments", "HTTPS/REST")
Rel(ecommerce, shipping, "Creates shipments", "HTTPS/REST")
Rel(ecommerce, email, "Sends emails", "SMTP")

@enduml
```

### C4 Container Diagram
```plantuml
@startuml C4_Container
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title Container Diagram - E-Commerce Platform

Person(customer, "Customer", "A user who wants to buy products")

System_Boundary(ecommerce, "E-Commerce Platform") {
    Container(spa, "Web Application", "Angular", "Delivers the web front-end")
    Container(mobile, "Mobile App", "React Native", "Mobile front-end")
    Container(api, "API Gateway", "NestJS", "Routes and validates requests")
    Container(users, "Users Service", "NestJS", "Manages user accounts")
    Container(catalog, "Catalog Service", "NestJS", "Product catalog management")
    Container(orders, "Orders Service", "NestJS", "Order processing")
    Container(cart, "Cart Service", "NestJS", "Shopping cart management")
    ContainerDb(db, "Database", "PostgreSQL", "Stores application data")
    ContainerQueue(queue, "Message Queue", "RabbitMQ", "Async messaging")
    Container(cache, "Cache", "Redis", "Caching layer")
}

System_Ext(payment, "Payment Gateway", "External payment processor")

Rel(customer, spa, "Uses", "HTTPS")
Rel(customer, mobile, "Uses", "HTTPS")
Rel(spa, api, "Calls", "HTTPS/REST")
Rel(mobile, api, "Calls", "HTTPS/REST")
Rel(api, users, "Routes to", "gRPC")
Rel(api, catalog, "Routes to", "gRPC")
Rel(api, orders, "Routes to", "gRPC")
Rel(api, cart, "Routes to", "gRPC")
Rel(users, db, "Reads/Writes", "TCP")
Rel(catalog, db, "Reads/Writes", "TCP")
Rel(orders, db, "Reads/Writes", "TCP")
Rel(orders, queue, "Publishes", "AMQP")
Rel(cart, cache, "Reads/Writes", "TCP")
Rel(orders, payment, "Processes", "HTTPS")

@enduml
```

### C4 Component Diagram
```plantuml
@startuml C4_Component
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Component Diagram - Orders Service

Container_Boundary(orders, "Orders Service") {
    Component(controller, "Orders Controller", "NestJS Controller", "REST API endpoints")
    Component(service, "Orders Service", "NestJS Service", "Business logic")
    Component(repository, "Orders Repository", "TypeORM Repository", "Data access")
    Component(validator, "Order Validator", "Class Validator", "Input validation")
    Component(mapper, "Order Mapper", "Class Transformer", "DTO mapping")
    Component(events, "Event Publisher", "Event Emitter", "Domain events")
    Component(saga, "Order Saga", "NestJS CQRS", "Order workflow")
}

ContainerDb(db, "PostgreSQL", "Database")
ContainerQueue(queue, "RabbitMQ", "Message Queue")
Container(inventory, "Inventory Service", "Stock management")
Container(payment, "Payment Service", "Payment processing")

Rel(controller, validator, "Validates with")
Rel(controller, service, "Uses")
Rel(service, repository, "Uses")
Rel(service, mapper, "Maps with")
Rel(service, events, "Publishes")
Rel(repository, db, "Reads/Writes")
Rel(events, queue, "Sends to")
Rel(saga, inventory, "Reserves stock")
Rel(saga, payment, "Processes payment")

@enduml
```

## Sequence Diagrams

### Basic Sequence Diagram
```plantuml
@startuml Sequence_CreateOrder
title Create Order Flow

actor Customer
participant "API Gateway" as API
participant "Orders Service" as Orders
participant "Inventory Service" as Inventory
participant "Payment Service" as Payment
database "Database" as DB
queue "Message Queue" as MQ

Customer -> API: POST /orders
activate API

API -> Orders: createOrder(orderDto)
activate Orders

Orders -> DB: Begin Transaction
activate DB

Orders -> Inventory: checkStock(items)
activate Inventory
Inventory --> Orders: StockAvailable
deactivate Inventory

Orders -> DB: Save Order (PENDING)
DB --> Orders: Order Created

Orders -> Payment: processPayment(order)
activate Payment

alt Payment Successful
    Payment --> Orders: PaymentConfirmed
    deactivate Payment
    
    Orders -> DB: Update Order (CONFIRMED)
    Orders -> Inventory: reserveStock(items)
    activate Inventory
    Inventory --> Orders: StockReserved
    deactivate Inventory
    
    Orders -> MQ: publish(OrderCreatedEvent)
    
    Orders -> DB: Commit Transaction
    DB --> Orders: Committed
    deactivate DB
    
    Orders --> API: OrderResponse
    deactivate Orders
    
    API --> Customer: 201 Created
    deactivate API
    
else Payment Failed
    Payment --> Orders: PaymentFailed
    deactivate Payment
    
    Orders -> DB: Rollback Transaction
    DB --> Orders: Rolled Back
    deactivate DB
    
    Orders --> API: PaymentException
    deactivate Orders
    
    API --> Customer: 402 Payment Required
    deactivate API
end

@enduml
```

### Async Message Flow
```plantuml
@startuml Sequence_AsyncFlow
title Async Order Processing

participant "Orders Service" as Orders
queue "Message Queue" as MQ
participant "Notification Service" as Notif
participant "Shipping Service" as Ship
participant "Analytics Service" as Analytics

Orders -> MQ: publish(OrderCreatedEvent)
activate MQ

MQ --> Notif: OrderCreatedEvent
activate Notif
Notif -> Notif: Send Confirmation Email
Notif --> MQ: ack
deactivate Notif

MQ --> Ship: OrderCreatedEvent
activate Ship
Ship -> Ship: Create Shipment
Ship -> MQ: publish(ShipmentCreatedEvent)
Ship --> MQ: ack
deactivate Ship

MQ --> Analytics: OrderCreatedEvent
activate Analytics
Analytics -> Analytics: Update Metrics
Analytics --> MQ: ack
deactivate Analytics

deactivate MQ

@enduml
```

## Class Diagrams

### Domain Model
```plantuml
@startuml Class_DomainModel
skinparam classAttributeIconSize 0

title Domain Model - E-Commerce

package "User Domain" {
    class User {
        -id: UUID
        -email: string
        -passwordHash: string
        -profile: UserProfile
        -roles: Role[]
        +authenticate(password): boolean
        +hasRole(role): boolean
    }
    
    class UserProfile {
        -firstName: string
        -lastName: string
        -phone: string
        -addresses: Address[]
    }
    
    class Address {
        -street: string
        -city: string
        -country: string
        -zipCode: string
        +format(): string
    }
    
    enum Role {
        CUSTOMER
        ADMIN
        VENDOR
    }
}

package "Order Domain" {
    class Order <<AggregateRoot>> {
        -id: UUID
        -userId: UUID
        -items: OrderItem[]
        -status: OrderStatus
        -total: Money
        -createdAt: DateTime
        +addItem(product, quantity): void
        +removeItem(productId): void
        +calculateTotal(): Money
        +confirm(): void
        +cancel(): void
    }
    
    class OrderItem <<ValueObject>> {
        -productId: UUID
        -productName: string
        -quantity: number
        -unitPrice: Money
        +subtotal(): Money
    }
    
    enum OrderStatus {
        PENDING
        CONFIRMED
        PROCESSING
        SHIPPED
        DELIVERED
        CANCELLED
    }
    
    class Money <<ValueObject>> {
        -amount: Decimal
        -currency: Currency
        +add(money): Money
        +subtract(money): Money
        +multiply(factor): Money
    }
}

package "Product Domain" {
    class Product <<AggregateRoot>> {
        -id: UUID
        -name: string
        -description: string
        -price: Money
        -category: Category
        -inventory: number
        +updatePrice(price): void
        +adjustInventory(delta): void
        +isAvailable(): boolean
    }
    
    class Category {
        -id: UUID
        -name: string
        -parentId: UUID
    }
}

User "1" -- "*" Order: places >
Order "1" *-- "*" OrderItem: contains >
OrderItem "*" --> "1" Product: references >
User "1" *-- "1" UserProfile
UserProfile "1" *-- "*" Address
User "*" -- "*" Role

@enduml
```

### Service Layer
```plantuml
@startuml Class_ServiceLayer
skinparam classAttributeIconSize 0

title Service Layer Architecture

interface IOrderService {
    +create(dto: CreateOrderDto): Promise<Order>
    +findById(id: string): Promise<Order>
    +findByUser(userId: string): Promise<Order[]>
    +updateStatus(id: string, status: OrderStatus): Promise<Order>
    +cancel(id: string): Promise<void>
}

interface IOrderRepository {
    +save(order: Order): Promise<Order>
    +findById(id: string): Promise<Order | null>
    +findByUserId(userId: string): Promise<Order[]>
    +update(order: Order): Promise<Order>
}

class OrderService implements IOrderService {
    -repository: IOrderRepository
    -inventoryService: IInventoryService
    -paymentService: IPaymentService
    -eventBus: EventBus
    +create(dto: CreateOrderDto): Promise<Order>
    +findById(id: string): Promise<Order>
    -validateOrder(dto: CreateOrderDto): void
    -publishEvent(event: DomainEvent): void
}

class OrderRepository implements IOrderRepository {
    -entityManager: EntityManager
    +save(order: Order): Promise<Order>
    +findById(id: string): Promise<Order | null>
}

class OrderController {
    -orderService: IOrderService
    +create(dto: CreateOrderDto): Promise<OrderDto>
    +findOne(id: string): Promise<OrderDto>
    +findAll(query: QueryDto): Promise<OrderDto[]>
}

OrderController --> IOrderService
OrderService --> IOrderRepository
OrderService --> IInventoryService
OrderService --> IPaymentService
OrderRepository ..|> IOrderRepository

@enduml
```

## ERD Diagrams

### Database Schema
```plantuml
@startuml ERD_Schema
!define primary_key(x) <b>x</b>
!define foreign_key(x) <i>x</i>
!define not_null(x) <u>x</u>

skinparam linetype ortho

title Database Schema - E-Commerce

entity "users" as users {
    primary_key(id) : UUID <<PK>>
    --
    not_null(email) : VARCHAR(255) <<UNIQUE>>
    not_null(password_hash) : VARCHAR(255)
    not_null(created_at) : TIMESTAMP
    updated_at : TIMESTAMP
    deleted_at : TIMESTAMP
}

entity "user_profiles" as profiles {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(user_id) : UUID <<FK>>
    first_name : VARCHAR(100)
    last_name : VARCHAR(100)
    phone : VARCHAR(20)
}

entity "addresses" as addresses {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(user_id) : UUID <<FK>>
    not_null(street) : VARCHAR(255)
    not_null(city) : VARCHAR(100)
    not_null(country) : VARCHAR(100)
    not_null(zip_code) : VARCHAR(20)
    is_default : BOOLEAN
}

entity "products" as products {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(category_id) : UUID <<FK>>
    not_null(name) : VARCHAR(255)
    description : TEXT
    not_null(price_amount) : DECIMAL(10,2)
    not_null(price_currency) : VARCHAR(3)
    not_null(inventory) : INTEGER
    not_null(created_at) : TIMESTAMP
}

entity "categories" as categories {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(parent_id) : UUID <<FK>>
    not_null(name) : VARCHAR(100)
    slug : VARCHAR(100) <<UNIQUE>>
}

entity "orders" as orders {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(user_id) : UUID <<FK>>
    foreign_key(shipping_address_id) : UUID <<FK>>
    not_null(status) : VARCHAR(20)
    not_null(total_amount) : DECIMAL(10,2)
    not_null(total_currency) : VARCHAR(3)
    not_null(created_at) : TIMESTAMP
    updated_at : TIMESTAMP
}

entity "order_items" as order_items {
    primary_key(id) : UUID <<PK>>
    --
    foreign_key(order_id) : UUID <<FK>>
    foreign_key(product_id) : UUID <<FK>>
    not_null(product_name) : VARCHAR(255)
    not_null(quantity) : INTEGER
    not_null(unit_price_amount) : DECIMAL(10,2)
    not_null(unit_price_currency) : VARCHAR(3)
}

users ||--o| profiles : "has"
users ||--o{ addresses : "has"
users ||--o{ orders : "places"
orders ||--|{ order_items : "contains"
order_items }o--|| products : "references"
products }o--|| categories : "belongs to"
categories ||--o{ categories : "parent"
orders }o--o| addresses : "ships to"

@enduml
```

## State Machine Diagrams

### Order State Machine
```plantuml
@startuml State_Order
title Order State Machine

[*] --> Draft : create

state Draft {
    [*] --> Empty
    Empty --> HasItems : addItem
    HasItems --> HasItems : addItem / removeItem
    HasItems --> Empty : removeLastItem
}

Draft --> Pending : submit
Pending --> Confirmed : confirmPayment
Pending --> Cancelled : cancel / paymentFailed

state Processing {
    [*] --> Preparing
    Preparing --> Packed : pack
    Packed --> ReadyToShip : verify
}

Confirmed --> Processing : startProcessing
Processing --> Shipped : ship

state Shipped {
    [*] --> InTransit
    InTransit --> OutForDelivery : outForDelivery
    OutForDelivery --> Delivered : deliver
    InTransit --> Failed : deliveryFailed
    OutForDelivery --> Failed : deliveryFailed
}

Shipped --> Delivered
Failed --> InTransit : retry

Delivered --> [*]
Cancelled --> [*]

note right of Pending
  Payment timeout: 30 minutes
  Auto-cancel if not paid
end note

note right of Processing
  Inventory reserved
  Cannot be cancelled
end note

@enduml
```

## Communication Diagrams

### Object Communication
```plantuml
@startuml Communication_Order
title Order Creation Communication

object "Customer" as customer
object "OrderController" as controller
object "OrderService" as service
object "OrderRepository" as repo
object "InventoryService" as inventory
object "PaymentService" as payment
object "EventBus" as events

customer --> controller : 1: createOrder(dto)
controller --> service : 2: create(dto)
service --> inventory : 3: checkAvailability(items)
inventory --> service : 4: availabilityResult
service --> repo : 5: save(order)
repo --> service : 6: savedOrder
service --> payment : 7: process(order)
payment --> service : 8: paymentResult
service --> inventory : 9: reserve(items)
service --> events : 10: publish(OrderCreatedEvent)
service --> controller : 11: order
controller --> customer : 12: response

@enduml
```

## Activity Diagrams

### Order Processing Flow
```plantuml
@startuml Activity_OrderProcessing
title Order Processing Activity

start

:Receive Order;

fork
    :Validate Order Details;
fork again
    :Check Customer Credit;
end fork

if (Valid and Credit OK?) then (yes)
    :Reserve Inventory;
    
    if (Inventory Available?) then (yes)
        :Process Payment;
        
        if (Payment Successful?) then (yes)
            :Confirm Order;
            
            fork
                :Send Confirmation Email;
            fork again
                :Update Analytics;
            fork again
                :Create Shipment;
            end fork
            
            :Complete;
            stop
        else (no)
            :Release Inventory;
            :Notify Payment Failed;
            stop
        endif
    else (no)
        :Notify Out of Stock;
        stop
    endif
else (no)
    :Reject Order;
    :Notify Customer;
    stop
endif

@enduml
```

## Best Practices

### Diagram Organization
1. Use consistent naming conventions
2. Include legends and titles
3. Use appropriate colors for different element types
4. Keep diagrams focused (one concept per diagram)
5. Use notes for important clarifications

### C4 Model Guidelines
1. Context → Container → Component → Code (zoom levels)
2. Each diagram should stand alone
3. Include all relevant systems/containers
4. Show data flow directions
5. Use standard C4 notation

### Sequence Diagram Tips
1. Show only relevant interactions
2. Use activation bars for clarity
3. Include alt/opt/loop fragments
4. Number messages for complex flows
5. Show async messages with open arrowheads

## References
- [PlantUML Documentation](https://plantuml.com/guide)
- [C4 Model](https://c4model.com)
- [C4-PlantUML](https://github.com/plantuml-stdlib/C4-PlantUML)
