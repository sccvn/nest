# Task Completion Checklist

## Before Submitting Changes

### 1. Code Quality
- [ ] Follow Google's JavaScript Style Guide
- [ ] Code wrapped at 100 characters
- [ ] Run formatter: `npm run format`
- [ ] Run linter: `npm run lint` (or `npm run lint:fix`)

### 2. Testing
- [ ] All features/bug fixes must have unit tests
- [ ] Run unit tests: `npm run test`
- [ ] Tests pass with no failures
- [ ] For integration changes, run: `npm run test:integration`

### 3. Build Verification
- [ ] Build succeeds: `npm run build`
- [ ] No TypeScript errors

### 4. Commit Message Format
Follow Angular commit convention:
```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style (formatting, semicolons)
- `refactor`: Code refactoring
- `perf`: Performance improvement
- `test`: Adding/correcting tests
- `build`: Build system changes
- `ci`: CI configuration changes
- `chore`: Other changes
- `sample`: Sample app changes

**Scopes:**
- `common`, `core`, `microservices`, `express`, `fastify`
- `socket.io`, `ws`, `testing`, `websockets`
- `sample/#` for example apps

**Examples:**
```
feat(core): add new lifecycle hook
fix(common): resolve injection scope issue
docs(changelog): update change log to v11
```

### 5. Pull Request Guidelines
- [ ] Search for existing PRs first
- [ ] Create feature branch from master
- [ ] Include appropriate test cases
- [ ] Follow coding rules
- [ ] Use descriptive commit messages
- [ ] Rebase on master before final push

## Quick Validation Commands
```bash
# Full validation sequence
npm run build && npm run test && npm run lint

# Quick check (lint only)
npm run lint:ci

# Format check
npm run format
```

## Integration Test Requirements
For changes affecting:
- Microservices: Docker required
- Database integrations: Specific containers needed
- WebSockets: Browser testing may be needed

Run full integration suite:
```bash
npm run test:docker:up
npm run test:integration
npm run test:docker:down
```
