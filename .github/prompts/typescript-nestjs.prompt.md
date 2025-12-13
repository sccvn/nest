# TypeScript NestJS Core Framework Development Expertise

## Overview
This prompt provides deep expertise in **NestJS core framework development** (v11.1.6+), including architectural patterns, dependency injection internals, decorator system, testing strategies, and best practices for contributing to the framework itself or building framework-grade modules.

**Target Audience**: Framework contributors, library authors, and advanced NestJS developers working on the core codebase.

**Key Focus Areas**:
- NestJS core architecture and internals
- Dependency injection container implementation
- Decorator metadata system
- Module compilation and resolution
- Platform adapter patterns
- Framework testing strategies (Mocha/Chai)
- Monorepo management (Lerna)

## Core Concepts

### Module System
```typescript
// Feature module structure
@Module({
  imports: [
    TypeOrmModule.forFeature([Entity]),
    CommonModule,
  ],
  controllers: [FeatureController],
  providers: [
    FeatureService,
    FeatureRepository,
    { provide: 'CONFIG', useValue: config },
  ],
  exports: [FeatureService],
})
export class FeatureModule {}
```

### Dependency Injection Scopes
```typescript
// DEFAULT - Singleton (shared across app)
@Injectable()
export class SingletonService {}

// REQUEST - New instance per request
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}

// TRANSIENT - New instance each injection
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {}
```

### Custom Decorators
```typescript
// Parameter decorator
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);

// Method decorator with metadata
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// Class decorator
export const ApiController = (prefix: string): ClassDecorator => {
  return applyDecorators(
    Controller(prefix),
    ApiTags(prefix),
    UseGuards(AuthGuard),
  );
};
```

## Design Patterns

### Repository Pattern
```typescript
// Repository abstraction
export interface IRepository<T> {
  findById(id: string): Promise<T | null>;
  findAll(options?: FindOptions): Promise<T[]>;
  create(data: Partial<T>): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Implementation
@Injectable()
export class UserRepository implements IRepository<User> {
  constructor(
    @InjectRepository(User)
    private readonly repo: Repository<User>,
  ) {}

  async findById(id: string): Promise<User | null> {
    return this.repo.findOne({ where: { id } });
  }
  // ... other methods
}
```

### CQRS Pattern
```typescript
// Command
export class CreateUserCommand {
  constructor(
    public readonly email: string,
    public readonly name: string,
  ) {}
}

// Command Handler
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(private readonly userService: UserService) {}

  async execute(command: CreateUserCommand): Promise<User> {
    return this.userService.create(command);
  }
}

// Query
export class GetUserQuery {
  constructor(public readonly id: string) {}
}

// Query Handler
@QueryHandler(GetUserQuery)
export class GetUserHandler implements IQueryHandler<GetUserQuery> {
  constructor(private readonly userService: UserService) {}

  async execute(query: GetUserQuery): Promise<User> {
    return this.userService.findOne(query.id);
  }
}
```

### Event-Driven Architecture
```typescript
// Event
export class UserCreatedEvent {
  constructor(public readonly user: User) {}
}

// Event Handler
@EventsHandler(UserCreatedEvent)
export class UserCreatedHandler implements IEventHandler<UserCreatedEvent> {
  constructor(private readonly emailService: EmailService) {}

  handle(event: UserCreatedEvent) {
    this.emailService.sendWelcome(event.user.email);
  }
}

// Publishing events
@Injectable()
export class UserService {
  constructor(private readonly eventBus: EventBus) {}

  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.create(dto);
    this.eventBus.publish(new UserCreatedEvent(user));
    return user;
  }
}
```

## API Development

### RESTful Controller
```typescript
@ApiTags('users')
@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @ApiOperation({ summary: 'Create user' })
  @ApiResponse({ status: 201, type: UserDto })
  @ApiResponse({ status: 400, description: 'Validation failed' })
  async create(@Body() dto: CreateUserDto): Promise<UserDto> {
    const user = await this.usersService.create(dto);
    return plainToClass(UserDto, user);
  }

  @Get()
  @ApiOperation({ summary: 'List users' })
  @ApiQuery({ name: 'page', required: false })
  @ApiQuery({ name: 'limit', required: false })
  async findAll(
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number,
    @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
  ): Promise<PaginatedResponse<UserDto>> {
    return this.usersService.findAll({ page, limit });
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: 'string' })
  async findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserDto> {
    const user = await this.usersService.findOne(id);
    if (!user) throw new NotFoundException();
    return plainToClass(UserDto, user);
  }

  @Put(':id')
  @Roles('admin')
  @UseGuards(RolesGuard)
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateUserDto,
  ): Promise<UserDto> {
    return this.usersService.update(id, dto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    await this.usersService.remove(id);
  }
}
```

### Validation
```typescript
// DTO with validation
export class CreateUserDto {
  @ApiProperty({ example: 'john@example.com' })
  @IsEmail()
  @IsNotEmpty()
  email: string;

  @ApiProperty({ example: 'John Doe' })
  @IsString()
  @Length(2, 100)
  name: string;

  @ApiProperty({ example: 'password123' })
  @IsString()
  @MinLength(8)
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and number',
  })
  password: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  roles?: string[];
}

// Custom validator
@ValidatorConstraint({ async: true })
@Injectable()
export class IsUniqueEmailConstraint implements ValidatorConstraintInterface {
  constructor(private readonly userService: UserService) {}

  async validate(email: string): Promise<boolean> {
    const user = await this.userService.findByEmail(email);
    return !user;
  }

  defaultMessage(): string {
    return 'Email already exists';
  }
}

export function IsUniqueEmail(options?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName,
      options,
      constraints: [],
      validator: IsUniqueEmailConstraint,
    });
  };
}
```

## Error Handling

### Custom Exceptions
```typescript
export class BusinessException extends HttpException {
  constructor(
    public readonly code: string,
    message: string,
    status: HttpStatus = HttpStatus.BAD_REQUEST,
  ) {
    super({ code, message }, status);
  }
}

export class UserNotFoundException extends BusinessException {
  constructor(userId: string) {
    super('USER_NOT_FOUND', `User with ID ${userId} not found`, HttpStatus.NOT_FOUND);
  }
}
```

### Exception Filter
```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  private readonly logger = new Logger(AllExceptionsFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const { status, body } = this.getErrorResponse(exception);

    this.logger.error(
      `${request.method} ${request.url} - ${status}`,
      exception instanceof Error ? exception.stack : undefined,
    );

    response.status(status).json({
      ...body,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }

  private getErrorResponse(exception: unknown): { status: number; body: object } {
    if (exception instanceof HttpException) {
      return {
        status: exception.getStatus(),
        body: exception.getResponse() as object,
      };
    }

    return {
      status: HttpStatus.INTERNAL_SERVER_ERROR,
      body: { message: 'Internal server error' },
    };
  }
}
```

## Testing Patterns

### Unit Test Setup
```typescript
describe('UsersService', () => {
  let service: UsersService;
  let repository: MockType<Repository<User>>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useFactory: repositoryMockFactory,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get(getRepositoryToken(User));
  });

  describe('findOne', () => {
    it('should return user when found', async () => {
      const user = createMockUser();
      repository.findOne.mockResolvedValue(user);

      const result = await service.findOne('1');

      expect(result).toEqual(user);
      expect(repository.findOne).toHaveBeenCalledWith({ where: { id: '1' } });
    });

    it('should throw NotFoundException when user not found', async () => {
      repository.findOne.mockResolvedValue(null);

      await expect(service.findOne('1')).rejects.toThrow(NotFoundException);
    });
  });
});
```

### E2E Test Setup
```typescript
describe('UsersController (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    })
      .overrideProvider(EmailService)
      .useValue(mockEmailService)
      .compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ transform: true }));
    await app.init();

    // Get auth token
    const response = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'test@test.com', password: 'password' });
    authToken = response.body.accessToken;
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /users', () => {
    it('should create user', () => {
      return request(app.getHttpServer())
        .post('/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ email: 'new@test.com', name: 'New User', password: 'Password1' })
        .expect(201)
        .expect((res) => {
          expect(res.body.email).toBe('new@test.com');
          expect(res.body.id).toBeDefined();
        });
    });
  });
});
```

## Configuration

### Environment Configuration
```typescript
// config/configuration.ts
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    username: process.env.DB_USERNAME,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
  },
  jwt: {
    secret: process.env.JWT_SECRET,
    expiresIn: process.env.JWT_EXPIRES_IN || '1h',
  },
});

// config/validation.ts
import * as Joi from 'joi';

export const validationSchema = Joi.object({
  NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
  PORT: Joi.number().default(3000),
  DB_HOST: Joi.string().required(),
  DB_PORT: Joi.number().default(5432),
  DB_USERNAME: Joi.string().required(),
  DB_PASSWORD: Joi.string().required(),
  DB_NAME: Joi.string().required(),
  JWT_SECRET: Joi.string().required(),
});

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
      validationSchema,
    }),
  ],
})
export class AppModule {}
```

## Best Practices

### Do's
- Use DTOs for all input/output
- Validate all inputs with class-validator
- Use dependency injection for testability
- Implement proper error handling
- Write comprehensive tests
- Use async/await consistently
- Follow single responsibility principle

### Don'ts
- Don't expose entities directly in APIs
- Don't use `any` type
- Don't skip validation
- Don't catch and swallow errors
- Don't hardcode configuration
- Don't use synchronous operations for I/O
- Don't skip tests for edge cases

## References
- [NestJS Documentation](https://docs.nestjs.com)
- [TypeORM Documentation](https://typeorm.io)
- [Class Validator](https://github.com/typestack/class-validator)
- [Class Transformer](https://github.com/typestack/class-transformer)
