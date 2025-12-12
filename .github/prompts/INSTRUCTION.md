# GitHub Copilot Custom Prompts - Onboarding Guide

## Overview

This folder contains domain-specific expertise prompts that provide deep knowledge in particular technologies, frameworks, methodologies, or practices. Prompts are used by agents to enhance their capabilities.

## Prompt Naming Convention

Prompts follow the pattern: `<domain>-<expertise>.prompt.md`

### Domain Categories
| Category | Examples |
|----------|----------|
| Programming Languages | `typescript`, `javascript`, `java`, `golang`, `rust` |
| Frameworks | `nestjs`, `express`, `fastify`, `angular`, `react` |
| Testing | `jest`, `mocha`, `k6`, `jmeter`, `cypress` |
| Architecture | `microservices`, `ddd`, `cqrs`, `event-sourcing` |
| DevOps | `docker`, `kubernetes`, `helm`, `terraform` |
| Documentation | `plantuml`, `mermaid`, `openapi`, `asyncapi` |
| Methodologies | `bdd`, `tdd`, `solid`, `clean-architecture` |
| Security | `owasp`, `snyk`, `sonarqube` |

## Available Prompts

### TypeScript & NestJS
- `typescript-nestjs.prompt.md` - NestJS framework expertise

### Architecture & Design
- `plantuml-diagrams.prompt.md` - UML diagram generation
- `architecture-patterns.prompt.md` - Design patterns and architecture

### Testing & Quality
- `bdd-tdd-testing.prompt.md` - BDD/TDD methodologies
- `code-quality-review.prompt.md` - Code quality and security

### Performance
- `load-testing-k6.prompt.md` - Performance and load testing

## How Prompts Are Used

1. **Agent Integration**: Agents reference prompts for domain knowledge
2. **Context Enhancement**: Prompts provide best practices and patterns
3. **Output Standards**: Define expected formats and quality criteria
4. **Tool Guidance**: Specify how to use specific tools effectively

## Creating New Prompts

1. Identify the domain and expertise area
2. Document core concepts and principles
3. Include practical examples and patterns
4. Specify output formats and templates
5. Add best practices and anti-patterns
6. Include tool-specific guidance

## Prompt Structure Template

```markdown
# <Domain> - <Expertise> Prompt

## Overview
Brief description of the expertise area

## Core Concepts
Key principles and fundamentals

## Patterns & Practices
Common patterns with examples

## Tools & Commands
Specific tools and their usage

## Output Templates
Expected output formats

## Best Practices
Do's and Don'ts

## References
Links to documentation
```
