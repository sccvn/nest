# PlantUML Diagrams Quick Reference

## 📊 All Diagrams Index

This document provides quick access to all PlantUML diagrams in the v1 documentation.

---

## 🏗️ Architecture Diagrams

### C4 Model Diagrams

#### 1. System Context Diagram
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#21-c4-context-diagram)  
**Purpose**: Shows NestJS system boundaries and external actors  
**Key Elements**: HTTP Clients, Microservices, Database, External APIs

#### 2. Container Diagram
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#22-c4-container-diagram)  
**Purpose**: Shows major runtime containers  
**Key Elements**: @nestjs/core, Platform Adapters, HTTP Server, Microservice Transports

#### 3. Component Diagram (@nestjs/core)
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#23-c4-component-diagram)  
**Purpose**: Internal structure of core package  
**Key Elements**: IoC Container, Router, Middleware, Guards, Interceptors, Pipes

---

## 📦 Class Diagrams

### IoC Container System

#### 4. IoC Container Architecture
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#31-ioc-container-system)  
**Classes**: NestContainer, ModulesContainer, Module, InstanceWrapper, Injector  
**Relationships**: Container → Modules → Providers → Instances

#### 5. Instance Wrapper Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#12-singleton-pattern)  
**Focus**: Scoped instance management with WeakMap

### Router System

#### 6. Router Architecture
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#32-router-system)  
**Classes**: RoutesResolver, RouterExplorer, RouterExecutionContext, RouterProxy  
**Flow**: Route resolution → Handler creation → Execution

#### 7. Request Handler Creation
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#32-router-system)  
**Focus**: How enhancers wrap handlers

### Exception System

#### 8. Exception Hierarchy
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#33-exception-hierarchy)  
**Classes**: HttpException, NotFoundException, BadRequestException, etc.  
**Pattern**: Standard HTTP status codes as exceptions

### Dynamic Modules

#### 9. Dynamic Module Pattern
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#34-dynamic-module-pattern)  
**Interfaces**: DynamicModule, ModuleMetadata  
**Methods**: forRoot(), forRootAsync(), forFeature()

### Decorators

#### 10. Custom Decorator Pattern
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#35-custom-decorator-pattern)  
**Purpose**: Creating reusable decorators  
**Example**: @User() parameter decorator

---

## 🔄 Sequence Diagrams

### Dependency Injection

#### 11. DI Resolution Flow
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#41-dependency-injection-resolution)  
**Actors**: Client, Injector, NestContainer, Module, InstanceWrapper  
**Flow**: Request provider → Lookup → Resolve dependencies → Instantiate → Cache

### Request Processing

#### 12. Request Handler Pipeline
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#42-request-handler-pipeline)  
**Layers**: HTTP Server → Router → Middleware → Guards → Interceptors → Pipes → Handler  
**Timing**: Shows pre/post-processing phases

### Error Handling

#### 13. Exception Handling Flow
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#43-exception-handling)  
**Actors**: Handler, RouterProxy, ExceptionsHandler, ExceptionFilter  
**Flow**: Exception thrown → Caught → Filtered → Response

### Module System

#### 14. Module Compilation
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#44-module-compilation)  
**Process**: Scan modules → Compile metadata → Generate tokens → Register providers

### Metadata

#### 15. Metadata Reflection
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#45-metadata-reflection)  
**Flow**: Decorator applied → Metadata stored → Runtime reflection → Usage

---

## 🔁 State Machine Diagrams

### Provider Lifecycle

#### 16. Provider Scope State Machine
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#51-provider-scope-lifecycle)  
**States**: Prototype, Initialized, Singleton, Request-Scoped, Transient  
**Transitions**: Based on scope configuration

### Application Lifecycle

#### 17. Application Lifecycle States
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#52-application-lifecycle)  
**States**: Initialization → Module Init → Bootstrap → Running → Shutdown  
**Hooks**: OnModuleInit, OnApplicationBootstrap, OnApplicationShutdown

---

## 📈 Activity Diagrams

#### 18. Request Processing Flow
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#61-request-processing-flow)  
**Activities**: Route matching, middleware execution, guard checks, pipe transformation  
**Decision Points**: Route found?, Guard passed?, Validation passed?

---

## 📡 Communication Diagrams

#### 19. Application Bootstrap
**File**: [LLD-NestJS-Core-Architecture.md](LLD-NestJS-Core-Architecture.md#71-application-bootstrap)  
**Components**: NestFactory, DependenciesScanner, ModuleCompiler, Injector  
**Messages**: Numbered interaction sequence

---

## 🎨 Design Pattern Diagrams

### Creational Patterns

#### 20. Factory Pattern Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#11-factory-pattern)  
**Class**: NestFactoryStatic  
**Methods**: create(), createMicroservice(), createApplicationContext()

#### 21. Singleton Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#12-singleton-pattern)  
**Implementation**: InstanceWrapper with STATIC_CONTEXT

#### 22. Builder Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#13-builder-pattern)  
**Classes**: MiddlewareBuilder, MiddlewareConfigProxy

### Structural Patterns

#### 23. Adapter Pattern Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#21-adapter-pattern)  
**Classes**: AbstractHttpAdapter, ExpressAdapter, FastifyAdapter  
**Purpose**: Platform abstraction

#### 24. Decorator Pattern (GoF)
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#22-decorator-pattern-gang-of-four)  
**Implementation**: Handler wrapping with Guards, Interceptors, Pipes

#### 25. Proxy Pattern Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#23-proxy-pattern)  
**Classes**: RouterProxy, ExceptionsHandler, ExceptionsZone

#### 26. Facade Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#24-facade-pattern)  
**Class**: NestApplication  
**Simplifies**: Middleware, Routes, Microservices, WebSockets

#### 27. Composite Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#25-composite-pattern)  
**Structure**: Module tree with imports

### Behavioral Patterns

#### 28. Chain of Responsibility Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#31-chain-of-responsibility-pattern)  
**Chains**: Middleware, Guards, Interceptors  
**Consumer**: GuardsConsumer with tryActivate()

#### 29. Strategy Pattern Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#32-strategy-pattern)  
**Interface**: ModuleOpaqueKeyFactory  
**Strategies**: ByReference, DeepHashed

#### 30. Observer Pattern Details
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#33-observer-pattern)  
**Subject**: NestApplicationContext  
**Observers**: Services with lifecycle hooks

#### 31. Template Method Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#34-template-method-pattern)  
**Abstract**: ContextCreator  
**Concrete**: GuardsContextCreator, PipesContextCreator, InterceptorsContextCreator

#### 32. Command Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#35-command-pattern)  
**Commands**: GetReplFn, DebugReplFn, SelectReplFn, ResolveReplFn

#### 33. Visitor Pattern
**File**: [Design-Patterns-Catalog.md](Design-Patterns-Catalog.md#36-visitor-pattern)  
**Visitor**: MetadataScanner  
**Usage**: PathsExplorer, DependenciesScanner

---

## 🧮 Algorithm Diagrams

### Data Structure Diagrams

#### 34. Module Graph Structure
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#11-module-graph)  
**Type**: Directed Acyclic Graph (DAG)  
**Nodes**: Modules, Edges: Imports

#### 35. Provider Registry Structure
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#12-provider-registry)  
**Type**: Hash Map  
**Keys**: InjectionToken, Values: InstanceWrapper

#### 36. Instance Cache Structure
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#13-instance-cache-scoped-instances)  
**Type**: WeakMap  
**Keys**: ContextId, Values: InstancePerContext

### Algorithm Activity Diagrams

#### 37. Module Distance Calculation (BFS)
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#21-module-distance-calculation)  
**Algorithm**: Breadth-First Search  
**Complexity**: O(V+E)

#### 38. Dependency Resolution (Topological Sort)
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#22-dependency-resolution-topological-sort)  
**Algorithm**: DFS with cycle detection  
**Complexity**: O(V+E)

#### 39. Provider Lookup Algorithm
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#23-provider-lookup-algorithm)  
**Search Order**: Local → Imports → Global  
**Complexity**: O(M×P)

#### 40. Guard Chain Execution
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#24-guard-chain-execution)  
**Pattern**: Short-circuit evaluation  
**Complexity**: O(n)

#### 41. Interceptor Chain Building
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#25-interceptor-chain-rxjs-pipeline)  
**Pattern**: Inside-out wrapping  
**Complexity**: O(n)

#### 42. Route Matching Algorithm
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#26-route-matching-algorithm)  
**Types**: Static, Parameterized, Wildcard, Regex  
**Complexity**: O(R)

#### 43. Module Compilation (Dynamic Modules)
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#27-module-compilation-dynamic-modules)  
**Process**: Extract metadata → Generate token → Create factory  
**Complexity**: O(P)

### Data Flow Diagrams

#### 44. Request Processing Data Flow
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#41-request-processing-data-flow)  
**Flow**: HTTP → Router → Pipeline → Handler → Response

#### 45. Dependency Injection Data Flow
**File**: [Algorithms-DataStructures.md](Algorithms-DataStructures.md#42-dependency-injection-data-flow)  
**Flow**: Scanner → Compiler → Container → Injector → Cache

---

## 🔧 Diagram Rendering Guide

### Method 1: VS Code (Recommended)

1. **Install Extension**:
   ```
   Name: PlantUML
   Id: jebbs.plantuml
   ```

2. **Preview Diagram**:
   - Open markdown file
   - Press `Alt+D` (or `Cmd+D` on Mac)
   - Or right-click → "Preview Current Diagram"

3. **Export**:
   - Right-click diagram → "Export Current Diagram"
   - Choose format: PNG, SVG, PDF

### Method 2: Online Editor

1. Go to [PlantUML Online](http://www.plantuml.com/plantuml)
2. Copy PlantUML code block from markdown
3. Paste into editor
4. Download rendered image

### Method 3: Command Line

```bash
# Install PlantUML
## Ubuntu/Debian
sudo apt-get install plantuml

## macOS
brew install plantuml

## Windows (via Chocolatey)
choco install plantuml

# Extract diagrams from markdown
grep -A 50 "@startuml" LLD-NestJS-Core-Architecture.md > temp.puml

# Render single diagram
plantuml diagram.puml

# Render all diagrams in directory
plantuml -tpng *.puml

# Generate SVG instead of PNG
plantuml -tsvg diagram.puml
```

### Method 4: Automated Extraction Script

Save as `extract-diagrams.sh`:
```bash
#!/bin/bash

# Extract all PlantUML diagrams from markdown files
for md_file in *.md; do
    echo "Processing $md_file..."
    
    # Extract diagram blocks
    awk '/@startuml/,/@enduml/' "$md_file" > temp.puml
    
    if [ -s temp.puml ]; then
        # Generate PNG
        plantuml temp.puml
        
        # Rename with source file name
        base_name="${md_file%.md}"
        counter=1
        
        for png in *.png; do
            if [ -f "$png" ]; then
                mv "$png" "${base_name}_diagram_${counter}.png"
                ((counter++))
            fi
        done
    fi
    
    rm -f temp.puml
done

echo "All diagrams extracted!"
```

Usage:
```bash
chmod +x extract-diagrams.sh
./extract-diagrams.sh
```

---

## 📊 Diagram Categories

### By Type
| Type | Count | Use Case |
|------|-------|----------|
| C4 Diagrams | 3 | System architecture overview |
| Class Diagrams | 10 | Object-oriented design |
| Sequence Diagrams | 5 | Interaction flows |
| State Machine | 2 | Lifecycle states |
| Activity Diagrams | 4 | Process flows |
| Communication | 1 | Component interaction |
| Data Structure | 3 | Internal structures |
| Algorithm Flows | 7 | Algorithm visualization |
| Data Flow | 2 | Data movement |

**Total**: 37+ diagrams

### By Purpose
| Purpose | Diagrams |
|---------|----------|
| **Understanding Architecture** | 1-3, 11-15, 19, 44-45 |
| **Learning Patterns** | 20-33 |
| **Analyzing Algorithms** | 34-43 |
| **Debugging Issues** | 6-8, 11, 13 |
| **Performance Optimization** | 37-43 |
| **Implementation Reference** | 4-10, 16-18 |

---

## 🎯 Quick Access by Task

### Task: Onboard New Developer
**Diagrams**: 1, 2, 3, 12, 18  
**Order**: Context → Container → Component → Request Flow → Activity

### Task: Debug DI Issue
**Diagrams**: 4, 5, 11, 38  
**Order**: Container Structure → Instance Wrapper → DI Sequence → Resolution Algorithm

### Task: Optimize Performance
**Diagrams**: 34-36, 37-43, 44-45  
**Order**: Data Structures → Algorithms → Data Flows

### Task: Implement Custom Feature
**Diagrams**: 20-33 (Pattern matching your feature)  
**Order**: Find similar pattern → Study implementation → Apply

### Task: Understand Request Lifecycle
**Diagrams**: 6, 12, 18, 44  
**Order**: Router → Sequence → Activity → Data Flow

---

## 📖 Diagram Legends

### Class Diagram Symbols
- `+` : Public
- `-` : Private
- `#` : Protected
- `{static}` : Static member
- `{abstract}` : Abstract member
- `<<interface>>` : Interface stereotype
- `<<abstract>>` : Abstract class stereotype

### Sequence Diagram Symbols
- `→` : Synchronous call
- `-->` : Response/return
- `⇢` : Asynchronous call
- `alt` : Alternative paths
- `loop` : Iteration
- `opt` : Optional

### Activity Diagram Symbols
- `( )` : Start/End
- `[ ]` : Action
- `◇` : Decision
- `⬢` : Merge
- `⬚` : Fork/Join

### State Machine Symbols
- `[*]` : Start/End state
- `─→` : Transition
- `state Name` : State definition

---

## 💡 Tips for Using Diagrams

### For Learning
1. Start with high-level (C4) diagrams
2. Drill down to class diagrams
3. Study sequence diagrams for flows
4. Reference algorithms for deep understanding

### For Development
1. Keep relevant class diagrams open
2. Use sequence diagrams for tracing
3. Reference patterns when implementing
4. Check algorithms for complexity

### For Documentation
1. Embed diagrams in design docs
2. Export as PNG for presentations
3. Link to source markdown for updates
4. Version diagrams with code changes

### For Debugging
1. Print relevant sequence diagrams
2. Trace execution path on paper
3. Compare actual vs expected flow
4. Identify deviation points

---

## 🔄 Updating Diagrams

When NestJS code changes:

1. **Identify Impact**:
   ```bash
   git diff master..HEAD packages/core/
   ```

2. **Find Affected Diagrams**:
   - Class structure changed → Update class diagrams
   - New pattern added → Add to catalog
   - Algorithm modified → Update pseudocode

3. **Update PlantUML**:
   - Edit markdown file
   - Modify `@startuml` block
   - Preview changes
   - Verify rendering

4. **Document Changes**:
   - Add change note in markdown
   - Update version in README
   - Commit with descriptive message

---

## 📚 Additional Resources

### PlantUML References
- [Official Documentation](https://plantuml.com/)
- [Cheat Sheet](https://ogom.github.io/draw_uml/plantuml/)
- [Real World Examples](https://real-world-plantuml.com/)

### UML Standards
- [UML 2.5 Specification](https://www.omg.org/spec/UML/2.5/)
- [C4 Model Guide](https://c4model.com/)

### Tools
- [PlantUML VS Code Extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
- [PlantUML Online Server](http://www.plantuml.com/plantuml)
- [Diagram Converter](https://github.com/gaphor/gaphor)

---

**Diagram Count**: 45+  
**Coverage**: Complete NestJS core architecture  
**Last Updated**: December 12, 2025
