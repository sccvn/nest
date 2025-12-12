# GitHub Copilot Custom Agents - Onboarding Guide

## Overview

This folder contains custom AI agent definitions for GitHub Copilot that provide specialized expertise for different roles and domains within the NestJS development lifecycle.

## Agent Naming Convention

Agents follow the pattern: `<role>-<domain>.agent.md`

### Roles
| Role | Description | Experience Level |
|------|-------------|------------------|
| `junior` | Entry-level, 0-2 years experience | Learning, guided tasks |
| `senior` | 5+ years experience | Independent, complex tasks |
| `principal` | 10+ years, technical leadership | Architecture, strategy |
| `lead` | Team leadership, mentoring | Coordination, reviews |

### Domains
| Domain | Expertise Area |
|--------|----------------|
| `solution-architect` | End-to-end system design |
| `software-architect` | Software patterns and structure |
| `business-analyst` | Requirements and process modeling |
| `software-engineer` | Development and implementation |
| `automation-tester` | Test automation and QA |
| `manual-tester` | Manual testing and exploration |
| `technical-writer` | Documentation |
| `devops-engineer` | CI/CD and infrastructure |

## Available Agents

### Architecture & Design
- `senior-solution-architect.agent.md` - System architecture extraction and design
- `senior-software-architect.agent.md` - Software patterns and low-level design

### Analysis & Planning
- `senior-business-analyst.agent.md` - Requirements analysis and BDD planning

### Development
- `senior-software-engineer.agent.md` - Implementation with TDD

### Quality & Testing
- `lead-code-reviewer.agent.md` - Code quality and security review
- `senior-automation-tester.agent.md` - Automated testing and load testing

## How to Create a New Agent

1. Copy the template structure from an existing agent
2. Define clear role and domain boundaries
3. Specify input/output formats
4. List tools and capabilities
5. Include examples where helpful

## Agent Selection

Agents are selected by the orchestrator (see `/instructions/`) based on:
- Task type and complexity
- Required expertise
- Current workflow phase
- Dependencies between tasks

## Integration with Instructions

Agents work together through instruction files that define:
- Multi-agent coordination models (HIVE, TEAM, etc.)
- Communication topologies (Hierarchical, Mesh, Star)
- Task delegation patterns
