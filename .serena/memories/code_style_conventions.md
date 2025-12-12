# NestJS Code Style and Conventions

## General Style
- **Style Guide**: Follows [Google's JavaScript Style Guide](https://google.github.io/styleguide/jsguide.html)
- **Line Length**: Wrap all code at **100 characters**
- **Formatter**: Prettier (run with `npm run format`)
- **Linter**: ESLint with TypeScript support

## TypeScript Conventions

### Type System
- `strictNullChecks: true` - Always handle null/undefined
- `noImplicitAny: false` - Explicit `any` is allowed
- `experimentalDecorators: true` - Decorators are heavily used
- `emitDecoratorMetadata: true` - Required for DI

### Naming Conventions
| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `NestApplication`, `CatsController` |
| Interfaces | PascalCase with `I` prefix (optional) | `ControllerOptions`, `INestApplication` |
| Methods | camelCase | `findAll()`, `useGlobalPipes()` |
| Variables | camelCase | `defaultPath`, `scopeOptions` |
| Constants | SCREAMING_SNAKE_CASE | `CONTROLLER_WATERMARK`, `PATH_METADATA` |
| Files | kebab-case | `nest-factory.ts`, `controller.decorator.ts` |
| Decorators | PascalCase | `@Controller()`, `@Injectable()` |

### File Organization
- One class per file (generally)
- File name matches the class name (kebab-case)
- Decorators in `decorators/` subdirectory
- Interfaces in `interfaces/` subdirectory
- Unit tests in `test/` subdirectory with `.spec.ts` suffix

## Documentation

### JSDoc Comments
- Use `@publicApi` tag for public API documentation
- Include `@see` links to documentation
- Document all overloaded function signatures separately

```typescript
/**
 * Decorator that marks a class as a Nest controller.
 * @param {string} prefix - Route path prefix
 * @see [Controllers](https://docs.nestjs.com/controllers)
 * @publicApi
 */
export function Controller(prefix: string): ClassDecorator;
```

## Testing Conventions
- Test framework: **Mocha** + **Chai** + **Sinon**
- Test files: `*.spec.ts` in `test/` directory
- Use `describe()` blocks for grouping
- Use `it()` for individual tests
- Use `expect()` from Chai for assertions

```typescript
import { expect } from 'chai';
import * as sinon from 'sinon';

describe('ClassName', () => {
  describe('methodName', () => {
    it('should do something', () => {
      expect(result).to.equal(expected);
    });
  });
});
```

## Decorator Patterns
- Decorators return a ClassDecorator, MethodDecorator, or ParameterDecorator
- Use `Reflect.defineMetadata()` to attach metadata
- Constants for metadata keys stored in `constants.ts`

```typescript
export function Controller(prefix?: string): ClassDecorator {
  return (target: object) => {
    Reflect.defineMetadata(CONTROLLER_WATERMARK, true, target);
    Reflect.defineMetadata(PATH_METADATA, prefix || '/', target);
  };
}
```

## Import Order
1. External npm packages
2. Internal packages (`@nestjs/*`)
3. Relative imports (../../)

## ESLint Configuration
Key rules (from `eslint.config.mjs`):
- `@typescript-eslint/no-explicit-any: 'off'` - `any` is allowed
- `@typescript-eslint/no-unused-vars: 'off'` - Unused vars allowed
- Uses Prettier plugin for formatting
