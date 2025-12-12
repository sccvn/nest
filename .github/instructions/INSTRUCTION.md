# GitHub Copilot Custom Instructions - Onboarding Guide

## Overview

This folder contains orchestration instructions that define how multiple AI agents collaborate to accomplish complex tasks. Instructions specify the coordination model, topology, and workflow patterns.

## Instruction Naming Convention

Instructions follow the pattern: `<model>_<topology>.instruction.md`

### Models
| Model | Description | Best For |
|-------|-------------|----------|
| `HIVE` | Hierarchical multi-agent system | Large projects, clear delegation |
| `TEAM` | Stable cooperative group | Continuous collaboration |
| `COALITION` | Temporary task force | One-off complex tasks |
| `HOLONIC` | Nested autonomous units | Modular, scalable systems |

### Topologies
| Topology | Structure | Use Case |
|----------|-----------|----------|
| `Hierarchical` | Tree structure, top-down | Enterprise, strict control |
| `Mesh` | Fully connected | Brainstorming, collaboration |
| `Ring` | Sequential pipeline | CI/CD, data processing |
| `Star` | Centralized hub | Simple coordination |

## Available Instructions

### Primary Orchestrators
- `HIVE_Hierarchical.instruction.md` - Main orchestrator for complex SDLC tasks
- `TEAM_Mesh.instruction.md` - Collaborative development workflows

## How Instructions Work

1. **Task Reception**: Instruction receives a task from user or parent orchestrator
2. **Task Analysis**: Breaks down task into subtasks
3. **Agent Selection**: Chooses appropriate agents for each subtask
4. **Delegation**: Assigns subtasks to agents with context
5. **Coordination**: Manages dependencies and communication
6. **Aggregation**: Combines results from agents
7. **Delivery**: Returns consolidated output

## Creating New Instructions

1. Define the coordination model and topology
2. Specify which agents are available
3. Define task routing rules
4. Establish communication protocols
5. Set quality gates between phases

## Integration Points

Instructions integrate with:
- **Agents**: Specialized workers for specific tasks
- **Prompts**: Domain knowledge and expertise
- **copilot-instructions.md**: Global configuration
