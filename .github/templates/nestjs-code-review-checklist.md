# NestJS Core Framework - Code Review Checklist

## Pre-Review Validation
- [ ] PR has descriptive title following convention: `type(scope): description`
  - Examples: `feat(core): add lazy module loading`, `fix(common): resolve metadata issue`
- [ ] PR description links to issue or provides clear context
- [ ] CI/CD pipeline passes (all tests, linting)
- [ ] Correct target branch (usually `master`)

---

## Functional Requirements

### Core Functionality
- [ ] Feature works as intended per specifications
- [ ] All acceptance criteria met
- [ ] Edge cases handled appropriately
- [ ] Error messages are clear and actionable
- [ ] No regressions in existing functionality

### Backward Compatibility (Critical for Framework)
- [ ] Public API changes are backward compatible OR
- [ ] Breaking changes documented with migration guide
- [ ] Deprecated APIs marked with `@deprecated` JSDoc
- [ ] Deprecation warnings logged at runtime when appropriate
- [ ] Version increments follow SemVer (major for breaking, minor for features, patch for fixes)

---

## NestJS Architecture Compliance

### Module System
- [ ] New modules follow standard structure:
  ```typescript
  @Module({
    imports: [...],
    controllers: [...],
    providers: [...],
    exports: [...]
  })
  ```
- [ ] Module dependencies are acyclic (no circular imports)
- [ ] Dynamic modules use `.forRoot()` / `.forFeature()` pattern correctly
- [ ] Global modules use `@Global()` decorator only when necessary

### Dependency Injection
- [ ] All services use `@Injectable()` decorator
- [ ] Constructor injection preferred over property injection
- [ ] Provider scope appropriate for use case:
  - `DEFAULT` (singleton) for stateless services
  - `REQUEST` only when request data needed
  - `TRANSIENT` only for truly per-instance state
- [ ] Custom providers use correct syntax:
  - `useClass` for alternative implementations
  - `useValue` for constants/config
  - `useFactory` for dynamic/async initialization
- [ ] Token-based injection uses `@Inject('TOKEN')` correctly
- [ ] Optional dependencies use `@Optional()` decorator

### Decorator Implementation
- [ ] Decorators attach metadata using `Reflect.defineMetadata()`
- [ ] Metadata keys are defined as constants (avoid string literals)
- [ ] Decorator parameters are strongly typed
- [ ] `reflect-metadata` imported before any decorators used
- [ ] Decorator factories return proper decorator signature:
  - `ClassDecorator`, `MethodDecorator`, `PropertyDecorator`, `ParameterDecorator`

### Request Pipeline
- [ ] Middleware, Guards, Interceptors, Pipes, Filters applied correctly
- [ ] Execution order understood and documented
- [ ] Exception filters catch appropriate error types
- [ ] Interceptors properly transform requests/responses
- [ ] Guards return boolean or throw exceptions

---

## Code Quality Standards

### TypeScript Best Practices
- [ ] No use of `any` type (use `unknown` or specific types)
- [ ] Interfaces used for contracts, types for unions/intersections
- [ ] Generics used where appropriate for type safety
- [ ] Enums used for fixed sets of values
- [ ] Return types explicitly declared on public methods
- [ ] Async functions return `Promise<T>`
- [ ] Proper use of `readonly` for immutable properties
- [ ] No unused imports or variables

### SOLID Principles
- [ ] **Single Responsibility**: Each class has one reason to change
- [ ] **Open/Closed**: Extensible without modification
- [ ] **Liskov Substitution**: Subtypes properly substitute base types
- [ ] **Interface Segregation**: No fat interfaces
- [ ] **Dependency Inversion**: Depend on abstractions, not concretions

### Clean Code
- [ ] Functions are small and focused (< 30 lines preferred)
- [ ] Descriptive names (no single letters except loops)
- [ ] No magic numbers (use named constants)
- [ ] Comments explain "why", not "what"
- [ ] No commented-out code
- [ ] No TODO/FIXME without issue reference

### Naming Conventions
- [ ] Classes: PascalCase (`CatsService`, `UserController`)
- [ ] Interfaces: PascalCase with descriptive name (`UserRepository`, not `IUserRepository`)
- [ ] Methods/Functions: camelCase (`findById`, `createUser`)
- [ ] Constants: UPPER_SNAKE_CASE (`MAX_RETRIES`, `DEFAULT_PORT`)
- [ ] Private fields: prefix with `_` when needed for clarity
- [ ] Decorators: PascalCase (`@Injectable`, `@Controller`)

---

## Testing Quality (Mocha + Chai)

### Unit Tests
- [ ] Test file exists: `[source-file].spec.ts` alongside source
- [ ] Test coverage > 80% for new/modified code
- [ ] All public methods tested
- [ ] Edge cases covered (null, undefined, empty arrays, etc.)
- [ ] Error cases tested (exceptions thrown/caught)
- [ ] Tests use Chai assertions:
  ```typescript
  expect(result).to.equal(expected);
  expect(value).to.exist;
  expect(array).to.have.lengthOf(3);
  expect(fn).to.throw(ErrorClass);
  ```
- [ ] No Jest patterns (avoid `toBe`, `toEqual`, etc.)
- [ ] Async tests use `async/await` properly
- [ ] Test doubles properly mocked/stubbed
- [ ] Tests are deterministic (no flaky tests)

### Integration Tests (when applicable)
- [ ] Integration tests in `integration/[feature-name]/`
- [ ] Tests use `Test.createTestingModule()` for DI
- [ ] HTTP tests use `supertest` or platform-specific client
- [ ] Database/external services mocked or Dockerized
- [ ] Tests clean up after themselves (no side effects)

### Test Structure
- [ ] Clear AAA pattern (Arrange, Act, Assert)
- [ ] One assertion concept per test (but multiple expects OK)
- [ ] Test names describe what is being tested:
  ```typescript
  it('should throw NotFoundException when user does not exist', ...)
  ```
- [ ] `before`/`after` hooks used for setup/teardown
- [ ] `beforeEach`/`afterEach` for per-test isolation

---

## Platform Agnosticism

### Adapter Independence
- [ ] Core code doesn't import Express or Fastify directly
- [ ] HTTP abstractions used (`HttpAdapter`, `HttpServer`)
- [ ] Request/Response objects not leaked to business logic
- [ ] Works with both Express and Fastify adapters
- [ ] WebSocket code compatible with Socket.io and ws

### Metadata System
- [ ] No platform-specific metadata keys
- [ ] Metadata reading works regardless of underlying platform

---

## Performance & Scalability

### Efficiency
- [ ] No N+1 query problems (if database code)
- [ ] Appropriate use of caching
- [ ] Lazy initialization where beneficial
- [ ] No memory leaks (event listeners cleaned up)
- [ ] Async operations don't block event loop
- [ ] Efficient algorithms (consider time/space complexity)

### Bundle Size
- [ ] New dependencies justified (evaluate alternatives)
- [ ] Tree-shaking supported (ES modules)
- [ ] Heavy dependencies lazy-loaded if possible
- [ ] No duplicate dependencies across packages

---

## Security

### Input Validation
- [ ] User input validated before processing
- [ ] DTOs use `class-validator` decorators (in application code)
- [ ] Injection attacks prevented (parameterized queries)
- [ ] Path traversal vulnerabilities avoided
- [ ] File uploads validated (type, size)

### Authentication & Authorization
- [ ] Guards properly enforce auth checks
- [ ] Sensitive operations require proper permissions
- [ ] No hardcoded secrets or credentials
- [ ] JWT tokens validated correctly (if applicable)

### Data Protection
- [ ] Sensitive data not logged
- [ ] Passwords never stored in plaintext
- [ ] Secrets managed via environment variables
- [ ] HTTPS enforced in production recommendations

---

## Documentation

### Code Documentation
- [ ] Public APIs have TSDoc comments:
  ```typescript
  /**
   * Creates a new user.
   * @param dto - The user creation data
   * @returns The created user entity
   * @throws {ConflictException} If email already exists
   */
  async create(dto: CreateUserDto): Promise<User>
  ```
- [ ] Complex algorithms explained with comments
- [ ] Decorator parameters documented
- [ ] Examples provided for non-obvious usage

### External Documentation
- [ ] README updated if needed
- [ ] Migration guide for breaking changes
- [ ] Examples in `sample/` directory updated
- [ ] API reference generated via TypeDoc

---

## Linting & Formatting

### ESLint Compliance
- [ ] `npm run lint` passes without errors
- [ ] No disabled ESLint rules without justification
- [ ] TypeScript strict mode compatible

### Prettier Formatting
- [ ] Code formatted with Prettier
- [ ] `.prettierignore` respected
- [ ] No manual formatting overrides

---

## Commit & PR Standards

### Commit Messages
- [ ] Follow Conventional Commits:
  - `feat(core):` - New features
  - `fix(common):` - Bug fixes
  - `chore(deps):` - Maintenance
  - `docs(readme):` - Documentation
  - `test(injector):` - Tests
  - `perf(router):` - Performance
  - `refactor(module):` - Code restructuring
- [ ] Commit message describes "why", not just "what"
- [ ] Commits are atomic (one logical change per commit)

### PR Description
- [ ] Links to related issue(s)
- [ ] Describes what changed and why
- [ ] Lists breaking changes (if any)
- [ ] Includes migration instructions (if needed)
- [ ] Screenshots/examples for visual changes

---

## Monorepo Considerations

### Package Structure
- [ ] Changes in correct package (`common`, `core`, `microservices`, etc.)
- [ ] No cross-package imports that violate boundaries
- [ ] Package dependencies updated in `package.json`
- [ ] Lerna versioning respected

### Build System
- [ ] `npm run build` succeeds
- [ ] TypeScript compilation clean (no errors)
- [ ] Source maps generated correctly
- [ ] Distribution files in correct locations

---

## Final Checks

### Before Approval
- [ ] All CI checks pass
- [ ] Code review comments addressed
- [ ] Requested changes implemented
- [ ] Follow-up issues created for future work (if needed)

### Merge Requirements
- [ ] At least one approval from maintainer
- [ ] All conversations resolved
- [ ] Squash commits if needed for clean history
- [ ] Merge to correct branch

---

## Risk Assessment

| Risk Area | Level | Notes |
|-----------|-------|-------|
| Backward Compatibility | High/Medium/Low | [Any concerns?] |
| Performance Impact | High/Medium/Low | [Benchmarks needed?] |
| Security | High/Medium/Low | [Security audit needed?] |
| Maintainability | High/Medium/Low | [Complexity concerns?] |

---

## Reviewer Notes

**Overall Assessment**: ✅ Approved / ⚠️ Changes Requested / ❓ Questions

**Strengths**:
- [What was done well?]

**Concerns**:
- [What needs improvement?]

**Follow-up Actions**:
- [ ] [Any follow-up tasks?]
