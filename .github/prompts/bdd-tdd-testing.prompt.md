# BDD/TDD Testing Methodology Expertise - NestJS Core Framework

## Overview
This prompt provides comprehensive expertise in Behavior-Driven Development (BDD) and Test-Driven Development (TDD) methodologies for creating high-quality, well-tested software, **specifically adapted for NestJS core framework development**.

**Critical Context**: 
- This project uses **Mocha + Chai**, NOT Jest
- Test structure: Unit tests (`.spec.ts`) alongside source, E2E tests in `integration/`
- Coverage target: > 80%
- Test assertions: `expect().to.equal()`, `expect().to.be.true` (Chai syntax)
- Async patterns: `async/await` with Mocha's `it()`, `before()`, `after()` hooks

## Behavior-Driven Development (BDD)

### Gherkin Syntax

#### Feature File Structure
```gherkin
@tag1 @tag2
Feature: Feature Name
  As a <role>
  I want <capability>
  So that <benefit>

  Background:
    Given common precondition for all scenarios

  @happy-path
  Scenario: Descriptive scenario name
    Given <precondition>
    And <another precondition>
    When <action>
    And <another action>
    Then <expected result>
    And <another expected result>
    But <negative expectation>

  @edge-case
  Scenario Outline: Scenario with multiple data sets
    Given <parameterized precondition with "<param1>">
    When <action with "<param2>">
    Then <expected result with "<expected>">

    Examples:
      | param1   | param2   | expected   |
      | value1a  | value1b  | expected1  |
      | value2a  | value2b  | expected2  |
      | value3a  | value3b  | expected3  |

  @error-handling
  Scenario: Error scenario
    Given <precondition for error>
    When <action that causes error>
    Then <error should be handled>
    And <user should see error message>
```

### BDD Scenario Patterns

#### CRUD Operations
```gherkin
Feature: User Management
  As an admin
  I want to manage users
  So that I can control system access

  Background:
    Given I am logged in as admin
    And the following users exist:
      | email           | name      | role    |
      | john@test.com   | John Doe  | user    |
      | jane@test.com   | Jane Doe  | admin   |

  # CREATE
  Scenario: Create a new user
    When I create a user with:
      | email          | name       | role |
      | new@test.com   | New User   | user |
    Then the user should be created
    And I should receive a confirmation email

  Scenario: Fail to create user with duplicate email
    When I create a user with email "john@test.com"
    Then I should see error "Email already exists"
    And no user should be created

  # READ
  Scenario: View user details
    When I view user with email "john@test.com"
    Then I should see user details:
      | field | value      |
      | name  | John Doe   |
      | role  | user       |

  Scenario: List all users
    When I request all users
    Then I should see 2 users
    And the list should be sorted by name

  # UPDATE
  Scenario: Update user role
    When I update user "john@test.com" role to "admin"
    Then the user role should be "admin"
    And an audit log should be created

  # DELETE
  Scenario: Delete a user
    When I delete user "john@test.com"
    Then the user should be soft-deleted
    And the user should not appear in user list
```

#### Authentication Scenarios
```gherkin
Feature: User Authentication
  As a user
  I want to authenticate securely
  So that I can access protected resources

  Scenario: Successful login
    Given I have a registered account
    When I login with valid credentials
    Then I should receive an access token
    And the token should expire in 1 hour

  Scenario: Failed login with wrong password
    Given I have a registered account
    When I login with wrong password
    Then I should see error "Invalid credentials"
    And no token should be issued

  Scenario: Account lockout after failed attempts
    Given I have a registered account
    When I fail to login 5 times
    Then my account should be locked
    And I should see error "Account locked. Try again in 30 minutes"

  Scenario: Password reset
    Given I have a registered account
    When I request password reset
    Then I should receive a reset email
    And the reset link should expire in 24 hours

  @security
  Scenario: Token refresh
    Given I have a valid access token
    And the token is about to expire
    When I refresh my token
    Then I should receive a new access token
    And the old token should be invalidated
```

#### API Testing Scenarios
```gherkin
Feature: REST API Endpoints
  As an API consumer
  I want well-designed endpoints
  So that I can integrate reliably

  @api @cats
  Scenario: GET /cats - List all cats
    Given the following cats exist:
      | name | age | breed   |
      | Tom  | 3   | Persian |
      | Sam  | 5   | Siamese |
    When I send GET request to "/cats"
    Then the response status should be 200
    And the response should contain 2 cats
    And each cat should have fields: id, name, age, breed

  @api @cats
  Scenario: GET /cats with pagination
    Given 25 cats exist
    When I send GET request to "/cats?page=2&limit=10"
    Then the response status should be 200
    And the response should contain 10 cats
    And the response should include pagination metadata:
      | field      | value |
      | totalItems | 25    |
      | totalPages | 3     |
      | currentPage| 2     |

  @api @cats
  Scenario: POST /cats - Create cat with valid data
    When I send POST request to "/cats" with:
      """json
      {
        "name": "Whiskers",
        "age": 2,
        "breed": "Maine Coon"
      }
      """
    Then the response status should be 201
    And the response should contain the created cat
    And the cat should have an id

  @api @cats @validation
  Scenario Outline: POST /cats - Validation errors
    When I send POST request to "/cats" with:
      """json
      {
        "name": "<name>",
        "age": <age>,
        "breed": "<breed>"
      }
      """
    Then the response status should be 400
    And the error message should contain "<error>"

    Examples:
      | name | age | breed   | error                    |
      |      | 2   | Persian | name should not be empty |
      | Tom  | -1  | Persian | age must be positive     |
      | Tom  | 2   |         | breed should not be empty|
```

## Test-Driven Development (TDD)

### TDD Cycle: Red-Green-Refactor

```
┌─────────────────────────────────────────────────────────────┐
│                    TDD Cycle                                 │
│                                                             │
│     ┌─────────┐                                             │
│     │   RED   │ ◄─────────────────────────────────┐        │
│     │  Write  │                                    │        │
│     │ Failing │                                    │        │
│     │  Test   │                                    │        │
│     └────┬────┘                                    │        │
│          │                                         │        │
│          ▼                                         │        │
│     ┌─────────┐                                    │        │
│     │  GREEN  │                                    │        │
│     │  Write  │                                    │        │
│     │ Minimal │                                    │        │
│     │  Code   │                                    │        │
│     └────┬────┘                                    │        │
│          │                                         │        │
│          ▼                                         │        │
│     ┌─────────┐                                    │        │
│     │REFACTOR │────────────────────────────────────┘        │
│     │ Improve │                                             │
│     │  Code   │                                             │
│     └─────────┘                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### TDD Example: Building a Calculator

#### Step 1: RED - Write Failing Test
```typescript
// calculator.spec.ts
describe('Calculator', () => {
  describe('add', () => {
    it('should add two positive numbers', () => {
      const calculator = new Calculator();
      
      const result = calculator.add(2, 3);
      
      expect(result).toBe(5);
    });
  });
});

// This test will FAIL because Calculator doesn't exist yet
```

#### Step 2: GREEN - Write Minimal Code
```typescript
// calculator.ts
export class Calculator {
  add(a: number, b: number): number {
    return a + b; // Minimal implementation to pass test
  }
}

// Test now PASSES
```

#### Step 3: RED - Add More Tests
```typescript
describe('Calculator', () => {
  let calculator: Calculator;

  beforeEach(() => {
    calculator = new Calculator();
  });

  describe('add', () => {
    it('should add two positive numbers', () => {
      expect(calculator.add(2, 3)).toBe(5);
    });

    it('should add negative numbers', () => {
      expect(calculator.add(-2, -3)).toBe(-5);
    });

    it('should add zero', () => {
      expect(calculator.add(5, 0)).toBe(5);
    });

    it('should handle decimal numbers', () => {
      expect(calculator.add(0.1, 0.2)).toBeCloseTo(0.3);
    });
  });

  describe('subtract', () => {
    it('should subtract two numbers', () => {
      expect(calculator.subtract(5, 3)).toBe(2);
    });
  });
});
```

#### Step 4: GREEN - Implement All Features
```typescript
export class Calculator {
  add(a: number, b: number): number {
    return a + b;
  }

  subtract(a: number, b: number): number {
    return a - b;
  }
}
```

#### Step 5: REFACTOR - Improve Code
```typescript
export class Calculator {
  /**
   * Adds two numbers together.
   * @param a - First operand
   * @param b - Second operand
   * @returns The sum of a and b
   */
  add(a: number, b: number): number {
    this.validateInputs(a, b);
    return a + b;
  }

  /**
   * Subtracts the second number from the first.
   * @param a - First operand
   * @param b - Second operand
   * @returns The difference of a and b
   */
  subtract(a: number, b: number): number {
    this.validateInputs(a, b);
    return a - b;
  }

  private validateInputs(...numbers: number[]): void {
    for (const n of numbers) {
      if (!Number.isFinite(n)) {
        throw new Error('Invalid number input');
      }
    }
  }
}
```

### Test Patterns

#### Arrange-Act-Assert (AAA)
```typescript
describe('UserService', () => {
  it('should create user with hashed password', async () => {
    // ARRANGE - Set up test data and mocks
    const dto = { email: 'test@test.com', password: 'password123' };
    const hashedPassword = 'hashed_password';
    mockHashService.hash.mockResolvedValue(hashedPassword);

    // ACT - Execute the code under test
    const result = await userService.create(dto);

    // ASSERT - Verify the results
    expect(result.email).toBe(dto.email);
    expect(result.password).toBe(hashedPassword);
    expect(mockHashService.hash).toHaveBeenCalledWith(dto.password);
  });
});
```

#### Given-When-Then
```typescript
describe('Order', () => {
  describe('when applying discount', () => {
    it('should reduce total by discount percentage', () => {
      // Given
      const order = new Order();
      order.addItem({ price: 100, quantity: 2 }); // Total: 200

      // When
      order.applyDiscount(10); // 10% discount

      // Then
      expect(order.total).toBe(180);
    });
  });
});
```

### Mock Patterns

#### Mock Dependencies
```typescript
describe('OrderService', () => {
  let orderService: OrderService;
  let mockOrderRepository: jest.Mocked<IOrderRepository>;
  let mockPaymentService: jest.Mocked<IPaymentService>;

  beforeEach(() => {
    mockOrderRepository = {
      save: jest.fn(),
      findById: jest.fn(),
      update: jest.fn(),
    };

    mockPaymentService = {
      process: jest.fn(),
      refund: jest.fn(),
    };

    orderService = new OrderService(mockOrderRepository, mockPaymentService);
  });

  it('should process payment and save order', async () => {
    // Arrange
    const orderDto = createOrderDto();
    const savedOrder = createOrder({ status: 'CONFIRMED' });
    
    mockPaymentService.process.mockResolvedValue({ success: true });
    mockOrderRepository.save.mockResolvedValue(savedOrder);

    // Act
    const result = await orderService.create(orderDto);

    // Assert
    expect(mockPaymentService.process).toHaveBeenCalledWith(
      expect.objectContaining({ amount: orderDto.total })
    );
    expect(mockOrderRepository.save).toHaveBeenCalled();
    expect(result.status).toBe('CONFIRMED');
  });
});
```

#### Stub External Services
```typescript
// Create stub for external API
const createPaymentGatewayStub = () => ({
  charge: jest.fn().mockImplementation(async (amount: number) => {
    if (amount > 10000) {
      throw new Error('Amount exceeds limit');
    }
    return { transactionId: 'tx_123', status: 'success' };
  }),
  
  refund: jest.fn().mockResolvedValue({ 
    refundId: 'ref_123', 
    status: 'refunded' 
  }),
});
```

### Test Coverage Goals

```
Coverage Targets:
├── Statements: >= 80%
├── Branches: >= 75%
├── Functions: >= 90%
└── Lines: >= 80%

Priority:
1. Business Logic (Service Layer): 95%+
2. Controllers/Handlers: 90%+
3. Utilities/Helpers: 85%+
4. DTOs/Models: 70%+
```

## Integration: BDD + TDD

### Combined Workflow

```
1. Business Analyst writes BDD scenarios (Gherkin)
         ↓
2. Developer writes step definitions (empty)
         ↓
3. TDD Cycle for each step:
   a. Write failing unit test
   b. Implement minimal code
   c. Refactor
         ↓
4. Integration tests based on BDD scenarios
         ↓
5. E2E tests verify full scenario
```

### Example: Implementing BDD Scenario with TDD

```gherkin
# cats.feature
Scenario: Create a new cat
  Given I am authenticated as admin
  When I create a cat with name "Tom" and age 3
  Then the cat should be saved
  And I should receive the cat details
```

```typescript
// Step 1: Step Definitions
defineFeature(feature, (test) => {
  test('Create a new cat', ({ given, when, then, and }) => {
    let authToken: string;
    let response: any;
    
    given('I am authenticated as admin', async () => {
      authToken = await getAdminToken();
    });
    
    when(/^I create a cat with name "(.*)" and age (\d+)$/, async (name, age) => {
      response = await createCat({ name, age: parseInt(age) }, authToken);
    });
    
    then('the cat should be saved', () => {
      expect(response.status).toBe(201);
    });
    
    and('I should receive the cat details', () => {
      expect(response.body).toHaveProperty('id');
      expect(response.body.name).toBe('Tom');
      expect(response.body.age).toBe(3);
    });
  });
});

// Step 2: TDD for CatsService.create()
describe('CatsService', () => {
  describe('create', () => {
    it('should create and return a new cat', async () => {
      // ... TDD implementation
    });
  });
});
```

## Best Practices

### BDD Best Practices
1. Write scenarios in business language
2. One scenario = one behavior
3. Keep scenarios independent
4. Use Background for common setup
5. Avoid technical implementation details

### TDD Best Practices
1. Write ONE failing test at a time
2. Write MINIMAL code to pass
3. Refactor only when tests pass
4. Test behavior, not implementation
5. Keep tests fast and isolated

### Test Quality Checklist
- [ ] Tests are independent (no shared state)
- [ ] Tests are deterministic (same result every run)
- [ ] Tests are fast (< 100ms for unit tests)
- [ ] Tests are readable (clear intent)
- [ ] Tests cover edge cases
- [ ] Tests cover error scenarios
- [ ] Tests use meaningful assertions

## References
- [Cucumber Gherkin Reference](https://cucumber.io/docs/gherkin/reference/)
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [TDD by Example - Kent Beck](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)
- [BDD in Action - John Ferguson Smart](https://www.manning.com/books/bdd-in-action)
