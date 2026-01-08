# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This is a new project in the **planning phase**. No source code has been written yet. The project is set up with the BMAD Method (BMM) framework for structured AI-assisted development.

## Project Overview

**Project Name:** iosGitlabNotifier
**Status:** Pre-development (BMAD workflow scaffolding complete)

## BMAD Method Framework

This project uses the BMad Method (BMM) - a structured framework for AI-assisted software development. The framework guides projects through four phases:

### Development Phases

1. **Analysis** - Product brief, market/technical research
2. **Planning** - UX design, PRD creation
3. **Solutioning** - Architecture, epics & stories creation, implementation readiness check
4. **Implementation** - Sprint planning, story development, code reviews, retrospectives

### Key Workflows

Use these slash commands to execute workflows:

| Command | Purpose |
|---------|---------|
| `/bmad:bmm:workflows:workflow-init` | Initialize project workflow tracking |
| `/bmad:bmm:workflows:workflow-status` | Check "what should I do now?" |
| `/bmad:bmm:workflows:create-product-brief` | Create product brief (Phase 1) |
| `/bmad:bmm:workflows:research` | Conduct market/technical research |
| `/bmad:bmm:workflows:create-prd` | Create PRD (Phase 2) |
| `/bmad:bmm:workflows:create-ux-design` | Plan UX patterns and design |
| `/bmad:bmm:workflows:create-architecture` | Design system architecture (Phase 3) |
| `/bmad:bmm:workflows:create-epics-and-stories` | Break down requirements into stories |
| `/bmad:bmm:workflows:sprint-planning` | Plan sprint (Phase 4) |
| `/bmad:bmm:workflows:dev-story` | Implement a story |
| `/bmad:bmm:workflows:code-review` | Adversarial code review |
| `/bmad:core:agents:bmad-master` | Launch BMAD Master orchestrator |

### Directory Structure

```
_bmad/               # BMAD framework files (do not modify)
  _config/           # Manifests and configuration
  bmm/               # BMad Method module
    config.yaml      # Project configuration
    workflows/       # Workflow definitions
    testarch/        # Test architecture knowledge base
  core/              # Core BMAD engine

_bmad-output/        # Generated artifacts
  planning-artifacts/   # PRD, architecture, UX docs
  implementation-artifacts/  # Implementation docs

docs/                # Project knowledge (when created)
```

### Configuration

Project config is in `_bmad/bmm/config.yaml`:
- `user_skill_level`: intermediate
- `communication_language`: English
- Output folders configured for planning and implementation artifacts

## Getting Started

1. Run `/bmad:bmm:workflows:workflow-init` to initialize project tracking
2. Run `/bmad:bmm:workflows:workflow-status` to see recommended next steps
3. Follow the BMAD workflow phases in order

## Workflow Execution Notes

- Workflows are interactive and collaborative - answer prompts to flesh out ideas
- Use "yolo" mode to auto-complete remaining sections without prompts
- Save outputs after each template-output section
- Use `/bmad:core:workflows:party-mode` for multi-agent discussions
