# NestJS Development Commands

## Essential Commands

### Building
```bash
# Build all packages (development)
npm run build

# Build all packages with clean first (production)
npm run build:prod

# Watch mode for development
npm run build:dev

# Clean build artifacts
npm run clean
```

### Testing
```bash
# Run unit tests
npm run test

# Run unit tests in watch mode
npm run test:dev

# Run unit tests with coverage
npm run test:cov

# Run integration tests (requires Docker)
npm run test:integration

# Start/stop Docker containers for integration tests
npm run test:docker:up
npm run test:docker:down
```

### Linting & Formatting
```bash
# Run all linters
npm run lint

# Fix linting issues
npm run lint:fix

# Format code with Prettier
npm run format

# Lint specific targets
npm run lint:packages    # Lint package source files
npm run lint:spec        # Lint spec files
npm run lint:integration # Lint integration tests
```

### Development Setup
```bash
# Install dependencies
npm ci --legacy-peer-deps

# Prepare development environment (builds and sets up samples)
sh scripts/prepare.sh

# Run integration test suite
sh scripts/run-integration.sh
```

### Publishing (Maintainers)
```bash
# Publish release
npm run publish

# Publish beta
npm run publish:beta

# Publish next
npm run publish:next

# Publish RC
npm run publish:rc
```

### Samples
```bash
# Build and test all samples
npm run build:samples

# Move packages to sample directories
npm run move:samples
```

## Git/System Commands (Linux)
```bash
# Common git commands
git checkout -b <branch-name> master   # Create feature branch
git commit -a                          # Commit all changes
git push origin <branch-name>          # Push changes
git rebase master -i                   # Interactive rebase
git push -f                            # Force push after rebase

# File system
ls -la                                 # List files
cd <directory>                         # Change directory
find . -name "*.ts"                    # Find files
grep -r "pattern" .                    # Search in files
```

## Quick Development Workflow
1. Clone and install: `npm ci --legacy-peer-deps`
2. Set up environment: `sh scripts/prepare.sh`
3. Make changes to packages
4. Build: `npm run build`
5. Test: `npm run test`
6. Lint: `npm run lint`
7. Format: `npm run format`
