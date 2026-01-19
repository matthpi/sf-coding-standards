# Salesforce Coding Standards

This repository provides **coding standards** and **Copilot instructions** for Salesforce development using Enterprise Patterns (fflib). These standards are designed to be both **machine-readable** (for GitHub Copilot) and **human-readable** (for developers).

## Purpose

- Provide consistent coding standards across Salesforce projects
- Enable GitHub Copilot to generate code following enterprise patterns
- Serve as reference documentation for development teams
- Ensure maintainability, testability, and scalability

## Documentation

**Essential reading:**

1. **[Critical Constraints](copilot-instructions.md#critical-constraints)** ⚠️ Read first!
2. **[Project Setup](docs/project-setup.md)** - Initialize new projects with fflib libraries and tooling
3. **[Architecture](docs/architecture.md)** - Foundational patterns and layer separation
4. **[Code Style](docs/code-style.md)** - Naming, formatting, and documentation standards
5. **[Sharing Model](docs/sharing-model.md)** - Security and sharing declarations
6. **[Testing](docs/testing.md)** - Test principles and best practices
   - **[Apex Testing](docs/testing-apex.md)** - Apex-specific patterns
   - **[LWC Testing](docs/testing-lwc.md)** - Jest testing for LWC
7. **[Validation](docs/validation.md)** - Code quality and pre-commit checks

**Topic-specific guides:**
- **[Logging](docs/logging.md)** - Nebula Logger patterns and transaction tracking
- **[Async Patterns](docs/async-patterns.md)** - Batch, Queueable, and Schedulable jobs
- **[Invocable Patterns](docs/invocable-patterns.md)** - Flow actions and invocable methods
- **[LWC Patterns](docs/lwc-patterns.md)** - Lightning Web Components and controllers
- **[REST API Patterns](docs/rest-api-patterns.md)** - REST resource development

## Using These Standards in Your Project

Reference these standards in your project's `copilot-instructions.md`:

```markdown
# Copilot Instructions

## Enterprise Coding Standards

This project follows the [Salesforce Coding Standards](https://github.com/matthpi/sf-coding-standards).

## Project-Specific Context

### Requirements
- [Key functional requirements and acceptance criteria]

### Objects & Data Model
- [Important custom objects, relationships, and field usage]

### Business Processes
- [Critical business rules, workflows, and process logic]

### Integrations
- [External systems, APIs, and integration patterns]
```

GitHub Copilot will automatically discover and apply both the enterprise standards and your project-specific context when working in your codebase.

## Contributing

When working with codebases using these standards:

- **Notice patterns** not documented and suggest additions
- **Flag inconsistencies** between instructions and actual code
- **Recommend clarifications** when instructions are ambiguous
- **Propose improvements** to examples or explanations

Suggest changes via pull request or issue in this repository.

## Maintaining These Standards

These instructions should evolve with the codebase. If you observe repeated patterns not captured here, document them to ensure consistency across projects.

---

**Last Updated:** January 2026
