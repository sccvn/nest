# Code Quality and Security Review Expertise - NestJS Core Framework

## Overview
This prompt provides comprehensive expertise in code quality assessment, security review, and compliance checking using industry-standard tools and best practices, **specifically tailored for the NestJS core framework codebase**.

**Framework-Specific Considerations**:
- **Backward Compatibility**: Public APIs must maintain compatibility or provide deprecation paths
- **Bundle Size**: Dependencies impact framework consumers, evaluate carefully
- **Platform Agnostic**: Code must work with Express, Fastify, and future adapters
- **Metadata System**: Proper use of `reflect-metadata` for decorator implementation
- **Testing**: Mocha/Chai patterns, >80% coverage requirement
- **Linting**: ESLint with NestJS-specific rules
- **Commit Convention**: Conventional Commits (feat, fix, chore, docs, test, perf, refactor)

## Code Quality Standards

### SOLID Principles

#### Single Responsibility Principle (SRP)
```typescript
// ❌ BAD: Multiple responsibilities
class UserService {
  async createUser(dto: CreateUserDto) { /* ... */ }
  async sendEmail(to: string, subject: string) { /* ... */ }
  async generateReport(userId: string) { /* ... */ }
}

// ✅ GOOD: Single responsibility
class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly eventBus: EventBus,
  ) {}

  async createUser(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepository.create(dto);
    this.eventBus.publish(new UserCreatedEvent(user));
    return user;
  }
}

class EmailService {
  async sendEmail(to: string, subject: string, body: string) { /* ... */ }
}

class ReportService {
  async generateUserReport(userId: string) { /* ... */ }
}
```

#### Open/Closed Principle (OCP)
```typescript
// ❌ BAD: Modifying existing code for new payment types
class PaymentProcessor {
  process(type: string, amount: number) {
    if (type === 'credit') { /* ... */ }
    else if (type === 'debit') { /* ... */ }
    else if (type === 'crypto') { /* ... */ } // New type requires modification
  }
}

// ✅ GOOD: Open for extension, closed for modification
interface PaymentStrategy {
  process(amount: number): Promise<PaymentResult>;
}

class CreditCardPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> { /* ... */ }
}

class CryptoPayment implements PaymentStrategy {
  async process(amount: number): Promise<PaymentResult> { /* ... */ }
}

class PaymentProcessor {
  constructor(private readonly strategies: Map<string, PaymentStrategy>) {}

  async process(type: string, amount: number): Promise<PaymentResult> {
    const strategy = this.strategies.get(type);
    if (!strategy) throw new Error(`Unknown payment type: ${type}`);
    return strategy.process(amount);
  }
}
```

#### Liskov Substitution Principle (LSP)
```typescript
// ❌ BAD: Square changes Rectangle behavior
class Rectangle {
  constructor(protected width: number, protected height: number) {}
  
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  setWidth(w: number) { this.width = this.height = w; } // Violates LSP
  setHeight(h: number) { this.width = this.height = h; }
}

// ✅ GOOD: Separate abstractions
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private side: number) {}
  area(): number { return this.side * this.side; }
}
```

#### Interface Segregation Principle (ISP)
```typescript
// ❌ BAD: Fat interface
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Robot implements Worker {
  work() { /* ... */ }
  eat() { throw new Error('Robots do not eat'); } // Forced to implement
  sleep() { throw new Error('Robots do not sleep'); }
}

// ✅ GOOD: Segregated interfaces
interface Workable {
  work(): void;
}

interface Feedable {
  eat(): void;
}

interface Restable {
  sleep(): void;
}

class Human implements Workable, Feedable, Restable {
  work() { /* ... */ }
  eat() { /* ... */ }
  sleep() { /* ... */ }
}

class Robot implements Workable {
  work() { /* ... */ }
}
```

#### Dependency Inversion Principle (DIP)
```typescript
// ❌ BAD: High-level depends on low-level
class UserService {
  private database = new MySQLDatabase();
  
  async getUser(id: string) {
    return this.database.query(`SELECT * FROM users WHERE id = ?`, [id]);
  }
}

// ✅ GOOD: Both depend on abstractions
interface IUserRepository {
  findById(id: string): Promise<User | null>;
}

class MySQLUserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> { /* ... */ }
}

class UserService {
  constructor(private readonly userRepository: IUserRepository) {}
  
  async getUser(id: string): Promise<User | null> {
    return this.userRepository.findById(id);
  }
}
```

### Clean Code Practices

#### Naming Conventions
```typescript
// Variables: camelCase, meaningful names
const userEmail = 'user@example.com';  // ✅
const e = 'user@example.com';           // ❌

// Functions: verb + noun, describes action
async function getUserById(id: string) { /* ... */ }  // ✅
async function data(id: string) { /* ... */ }          // ❌

// Classes: PascalCase, noun
class UserRepository { /* ... */ }  // ✅
class DoStuff { /* ... */ }         // ❌

// Constants: SCREAMING_SNAKE_CASE
const MAX_RETRY_ATTEMPTS = 3;  // ✅
const maxRetry = 3;            // ❌

// Interfaces: PascalCase, often with 'I' prefix or descriptive
interface IUserService { /* ... */ }
interface UserService { /* ... */ }   // Also acceptable in TS

// Boolean: is/has/can/should prefix
const isActive = true;
const hasPermission = false;
const canEdit = true;
```

#### Function Design
```typescript
// ✅ GOOD: Small, single purpose
async function createUser(dto: CreateUserDto): Promise<User> {
  validateUserDto(dto);
  const hashedPassword = await hashPassword(dto.password);
  const user = await saveUser({ ...dto, password: hashedPassword });
  await sendWelcomeEmail(user.email);
  return user;
}

// ❌ BAD: Too many responsibilities, too long
async function handleUserRegistration(req, res) {
  // 200 lines of mixed concerns...
}

// ✅ GOOD: Limited parameters (max 3, use object for more)
function createOrder(userId: string, items: OrderItem[]): Order { }
function createOrder(options: CreateOrderOptions): Order { }

// ❌ BAD: Too many parameters
function createOrder(userId, items, discount, shipping, tax, notes) { }
```

#### Error Handling
```typescript
// ✅ GOOD: Specific exceptions
class UserNotFoundException extends NotFoundException {
  constructor(userId: string) {
    super(`User with ID ${userId} not found`);
  }
}

class InvalidCredentialsException extends UnauthorizedException {
  constructor() {
    super('Invalid email or password');
  }
}

// ✅ GOOD: Proper error handling
async function getUser(id: string): Promise<User> {
  try {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new UserNotFoundException(id);
    }
    return user;
  } catch (error) {
    if (error instanceof UserNotFoundException) {
      throw error;
    }
    this.logger.error(`Failed to get user ${id}`, error);
    throw new InternalServerErrorException('Failed to retrieve user');
  }
}

// ❌ BAD: Swallowing errors
async function getUser(id: string): Promise<User | null> {
  try {
    return await this.userRepository.findById(id);
  } catch (error) {
    return null; // Lost error information
  }
}
```

## Security Review

### OWASP Top 10 Checklist

#### 1. Injection (SQL, NoSQL, Command)
```typescript
// ❌ VULNERABLE: SQL Injection
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// ✅ SECURE: Parameterized query
const user = await this.userRepository.findOne({ where: { id: userId } });

// ❌ VULNERABLE: Command Injection
exec(`git clone ${userInput}`);

// ✅ SECURE: Sanitize and validate
import { execFile } from 'child_process';
if (isValidGitUrl(userInput)) {
  execFile('git', ['clone', userInput]);
}
```

#### 2. Broken Authentication
```typescript
// ✅ SECURE: Strong password policy
class PasswordValidator {
  validate(password: string): ValidationResult {
    const errors: string[] = [];
    if (password.length < 12) errors.push('Minimum 12 characters');
    if (!/[A-Z]/.test(password)) errors.push('At least one uppercase');
    if (!/[a-z]/.test(password)) errors.push('At least one lowercase');
    if (!/[0-9]/.test(password)) errors.push('At least one number');
    if (!/[^A-Za-z0-9]/.test(password)) errors.push('At least one special character');
    return { valid: errors.length === 0, errors };
  }
}

// ✅ SECURE: Rate limiting for login
@UseGuards(ThrottlerGuard)
@Throttle(5, 60) // 5 attempts per 60 seconds
@Post('login')
async login(@Body() dto: LoginDto) { /* ... */ }

// ✅ SECURE: Account lockout
if (user.failedLoginAttempts >= 5) {
  throw new AccountLockedException();
}
```

#### 3. Sensitive Data Exposure
```typescript
// ❌ BAD: Exposing sensitive data
return user; // Includes passwordHash, tokens, etc.

// ✅ GOOD: DTO transformation
return plainToClass(UserResponseDto, user, {
  excludeExtraneousValues: true,
});

// ✅ GOOD: Explicit field selection
class UserResponseDto {
  @Expose() id: string;
  @Expose() email: string;
  @Expose() name: string;
  // passwordHash is NOT exposed
}

// ✅ GOOD: Encrypt sensitive data at rest
@Column({ transformer: new EncryptionTransformer(key) })
ssn: string;
```

#### 4. XML External Entities (XXE)
```typescript
// ✅ SECURE: Disable external entities
import { parseStringPromise } from 'xml2js';

const options = {
  explicitArray: false,
  ignoreAttrs: true,
  // Disable external entities
  xmldec: { version: '1.0', encoding: 'UTF-8', standalone: true },
};

const result = await parseStringPromise(xml, options);
```

#### 5. Broken Access Control
```typescript
// ❌ VULNERABLE: No authorization check
@Get(':id')
async getOrder(@Param('id') id: string) {
  return this.orderService.findById(id);
}

// ✅ SECURE: Authorization check
@Get(':id')
@UseGuards(JwtAuthGuard, OrderOwnerGuard)
async getOrder(
  @Param('id') id: string,
  @CurrentUser() user: User,
) {
  const order = await this.orderService.findById(id);
  if (order.userId !== user.id && !user.isAdmin) {
    throw new ForbiddenException();
  }
  return order;
}

// ✅ SECURE: Role-based access
@Roles('admin')
@UseGuards(RolesGuard)
@Delete(':id')
async deleteUser(@Param('id') id: string) { /* ... */ }
```

#### 6. Security Misconfiguration
```typescript
// ✅ SECURE: Helmet middleware
import helmet from 'helmet';
app.use(helmet());

// ✅ SECURE: CORS configuration
app.enableCors({
  origin: ['https://example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
});

// ✅ SECURE: Remove sensitive headers
app.use((req, res, next) => {
  res.removeHeader('X-Powered-By');
  next();
});
```

#### 7. Cross-Site Scripting (XSS)
```typescript
// ✅ SECURE: Output encoding
import { escape } from 'html-escaper';

const safeOutput = escape(userInput);

// ✅ SECURE: Content Security Policy
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'unsafe-inline'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "data:", "https:"],
  },
}));

// ✅ SECURE: Input validation
class CreateCommentDto {
  @IsString()
  @MaxLength(1000)
  @Transform(({ value }) => sanitizeHtml(value))
  content: string;
}
```

#### 8. Insecure Deserialization
```typescript
// ✅ SECURE: Validate and transform input
class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsEnum(Role)
  @IsOptional()
  role?: Role = Role.USER; // Default to least privilege
}

// ✅ SECURE: Whitelist properties
app.useGlobalPipes(new ValidationPipe({
  whitelist: true, // Strip non-whitelisted properties
  forbidNonWhitelisted: true, // Throw error for non-whitelisted
  transform: true,
}));
```

#### 9. Using Components with Known Vulnerabilities
```bash
# Check dependencies for vulnerabilities
npm audit

# Fix vulnerabilities
npm audit fix

# Use Snyk for deeper analysis
snyk test
snyk monitor
```

#### 10. Insufficient Logging & Monitoring
```typescript
// ✅ GOOD: Comprehensive logging
@Injectable()
export class AuditInterceptor implements NestInterceptor {
  constructor(private readonly logger: Logger) {}

  intercept(context: ExecutionContext, next: CallHandler) {
    const request = context.switchToHttp().getRequest();
    const { method, url, user, ip } = request;
    
    const startTime = Date.now();
    
    return next.handle().pipe(
      tap({
        next: () => {
          this.logger.log({
            type: 'API_ACCESS',
            method,
            url,
            userId: user?.id,
            ip,
            duration: Date.now() - startTime,
            status: 'SUCCESS',
          });
        },
        error: (error) => {
          this.logger.error({
            type: 'API_ERROR',
            method,
            url,
            userId: user?.id,
            ip,
            duration: Date.now() - startTime,
            error: error.message,
            stack: error.stack,
          });
        },
      }),
    );
  }
}
```

## SonarQube Integration

### Quality Gates
```yaml
# sonar-project.properties
sonar.projectKey=nestjs-project
sonar.projectName=NestJS Project
sonar.sources=src
sonar.tests=test
sonar.typescript.lcov.reportPaths=coverage/lcov.info

# Quality Gates (configure in SonarQube)
sonar.qualitygate.conditions:
  - metric: coverage
    op: LT
    value: 80
  - metric: duplicated_lines_density
    op: GT
    value: 3
  - metric: reliability_rating
    op: GT
    value: 1  # A
  - metric: security_rating
    op: GT
    value: 1  # A
  - metric: sqale_rating
    op: GT
    value: 1  # A
```

### Common SonarQube Rules

| Rule | Severity | Description |
|------|----------|-------------|
| `typescript:S1854` | Major | Remove unused variables |
| `typescript:S4144` | Major | Avoid duplicate function implementations |
| `typescript:S3776` | Critical | Cognitive complexity too high |
| `typescript:S2068` | Blocker | Hard-coded credentials |
| `typescript:S5693` | Critical | Denial of Service (ReDoS) |

## Snyk Integration

### Configuration
```yaml
# .snyk
version: v1.19.0
ignore: {}
patch: {}
```

### Commands
```bash
# Test for vulnerabilities
snyk test

# Test code for security issues
snyk code test

# Monitor project
snyk monitor

# Fix vulnerabilities
snyk fix

# Test container images
snyk container test my-image:latest
```

### CI/CD Integration
```yaml
# GitHub Actions
- name: Run Snyk Security Scan
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
```

## Code Review Checklist

### General
- [ ] Code follows project style guide
- [ ] No commented-out code
- [ ] No console.log statements
- [ ] Proper error handling
- [ ] Meaningful variable/function names
- [ ] Functions are small and focused
- [ ] No code duplication

### Security
- [ ] Input validation on all endpoints
- [ ] Output encoding where needed
- [ ] Authentication required for protected routes
- [ ] Authorization checks in place
- [ ] No hardcoded secrets
- [ ] SQL/NoSQL injection prevented
- [ ] XSS prevented

### Performance
- [ ] No N+1 queries
- [ ] Proper indexing considered
- [ ] Pagination for list endpoints
- [ ] Caching where appropriate
- [ ] Async operations don't block

### Testing
- [ ] Unit tests for new code
- [ ] Edge cases covered
- [ ] Error scenarios tested
- [ ] Coverage meets threshold

## References
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [SonarQube Rules](https://rules.sonarsource.com/typescript)
- [Snyk Vulnerability DB](https://snyk.io/vuln)
- [Clean Code by Robert C. Martin](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
