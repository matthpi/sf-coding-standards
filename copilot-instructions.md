# Copilot Instructions for Salesforce Development

This file provides critical constraints and references to detailed coding standards for Salesforce development using Enterprise Patterns (fflib).

## Critical Constraints

These constraints must **NEVER** be violated:

### Code Quality & Suppressions

- **Never** apply `@SuppressWarnings` as AI - only humans can add suppressions after review
  - ❌ DO NOT add @SuppressWarnings even if you think it's "legitimate"
  - ❌ DO NOT add @SuppressWarnings to "clean up" PMD violations
  - ✅ DO report PMD violations and let the human decide
  - ✅ DO fix actual code issues when possible

### Security & Sensitive Data

- **Never** include security keys, credentials, or authentication tokens in code
- **Never** include org-specific IDs (org IDs, record IDs, user IDs)
- **Never** include proprietary business logic details in comments or logs
- **Never** include customer/client names or identifying information
- **Always** use placeholders for sensitive values (e.g., `YOUR-ORG`, `<org-id>`)
- **Always** use custom metadata or custom settings for environment-specific configuration
- **Always** flag if user provides sensitive data that should be externalized

### Architecture & Layer Separation

- **Never** put SOQL in Service classes - always delegate to Selector
- **Never** put business logic in Batch/Queueable/Controller/REST resources - delegate to Service
- **Never** use `new ServiceImpl()` or `new Selector()` - always use factory methods
- **Never** mix K&R and Allman braces - consistent Allman style only
- **Never** use batch-from-batch - use `!System.isBatch()` guard or chain to queueable

### Testing & Logging

- **Always** use `Assert` methods, never `System.assert*`
- **Always** call `Logger.saveLog()` in async `finally` blocks
- **Always** call `Logger.saveLog()` in Consumer layer (controllers, invocables, batch, queueable)
- **Always** use savepoints in batch/queueable execute methods
- **Always** implement interfaces for Services and Selectors (enables mocking)
- **Consider** splitting test classes if mostly parallelizable with few non-parallel tests (`IsParallel` applies to entire class)

### Database Operations

- **Always** use `Application.UnitOfWork` for DML operations in services

### Security & Sharing

- **Always** use explicit sharing declarations on all classes (never omit)
- **Always** use `inherited sharing` on selectors, services, and utils
- **Always** use `with sharing` on controllers and invocables
- **Always** use `without sharing` on async jobs (batch/queueable/schedulable) for system context

### Documentation & Validation

- **Always** run `sf code-analyzer run` before committing code
- **Always** provide complete ApexDoc for all public APIs

## Detailed Documentation

For comprehensive patterns and examples, see:

- **[Architecture](docs/architecture.md)** - Enterprise patterns, layer separation, Application factory
- **[Code Style](docs/code-style.md)** - Naming conventions, braces, comments, ApexDoc
- **[Sharing Model](docs/sharing-model.md)** - Security and sharing declarations by layer
- **[Testing](docs/testing.md)** - Test principles and best practices
  - **[Apex Testing](docs/testing-apex.md)** - Apex test patterns, mocking, assertions
  - **[LWC Testing](docs/testing-lwc.md)** - Jest testing for Lightning Web Components
- **[Logging](docs/logging.md)** - Nebula Logger patterns and transaction tracking
- **[Validation](docs/validation.md)** - Code Analyzer and pre-commit checks
- **[Async Patterns](docs/async-patterns.md)** - Batch, Queueable, Schedulable patterns
- **[Invocable Patterns](docs/invocable-patterns.md)** - Flow actions and invocable methods
- **[LWC Patterns](docs/lwc-patterns.md)** - Lightning Web Components and controllers
- **[REST API Patterns](docs/rest-api-patterns.md)** - REST resource development

## Quick Reference

### Architecture Philosophy

All Salesforce projects follow **Enterprise Apex Patterns (fflib)** with strict separation of concerns and dependency injection via custom metadata.

### Four-Layer Architecture

**Never mix layers.** Each layer has a specific responsibility:

1. **Selector Layer** (`selectors/`) - All SOQL queries, FLS enforcement
   - Factory: `MySelector.newInstance()` → `Application.Selector`
   - Return `Database.QueryLocator` for batch queries

2. **Domain Layer** (`domains/`) - SObject collection logic, validation
   - Interface: `IMyDomain`, Implementation: `MyDomain extends fflib_SObjects`
   - Factory: `Application.Domain.newInstance(recordList)`
   - NO direct SOQL or DML

3. **Service Layer** (`services/`) - Business logic only
   - Interface: `IMyService`, Implementation: `MyServiceImpl`, Facade: `MyService`
   - Factory: `Application.Service.newInstance(IMyService.class)`
   - Delegate SOQL to Selectors, DML to `Application.UnitOfWork`

4. **Consumer Layer** (`batch/`, `queueable/`, `controllers/`, `triggers/`, `invocables/`)
   - Thin orchestrators - delegate to Service layer
   - Handle async context (savepoints, logging, chaining)
   - See [invocable-patterns.md](docs/invocable-patterns.md) for Flow integration
   - Handle async context (savepoints, logging, chaining)

See [docs/architecture.md](docs/architecture.md) for complete patterns and workflows.

### Code Style Essentials

- **Allman braces** - opening brace on new line, always
- **Naming**: `PascalCase` classes, `camelCase` methods, `UPPER_SNAKE_CASE` constants
- **Interfaces**: Prefix with `I` (e.g., `IAccountService`)
- **ApexDoc required** for all public APIs with proper tags

See [docs/code-style.md](docs/code-style.md) for full style guide.

### Testing Essentials

- Use modern `Assert` methods (not deprecated `System.assert*`)
- Given/When/Then structure for all tests
- Mock dependencies with `fflib_ApexMocks`
- Use `LoggerMockDataStore` for logger testing

See [docs/testing.md](docs/testing.md) for complete testing patterns.

## Maintaining These Instructions

When working with this codebase:
- **Notice patterns** not documented here and suggest additions
- **Flag inconsistencies** between these instructions and actual code patterns
- **Recommend clarifications** when instructions are ambiguous or incomplete
- **Propose improvements** to examples or explanations that could be clearer

If you observe repeated patterns in the codebase (coding style, architecture decisions, naming conventions, etc.) that aren't captured in these instructions, call them out so they can be documented for consistency.
