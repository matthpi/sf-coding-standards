# Contributing to Salesforce Coding Standards

Thank you for your interest in improving these coding standards! This document provides guidelines for contributing.

## How to Contribute

### Suggesting Improvements

If you notice patterns in your codebase that aren't documented, or find inconsistencies:

1. **Check existing documentation first** - Search all docs to ensure it's not already covered
2. **Open an issue** - Describe the pattern, why it's useful, and provide examples
3. **Propose changes** - Submit a pull request with your suggested additions

### Reporting Issues

If you find:
- **Conflicting information** between files
- **Outdated patterns** that no longer align with best practices
- **Unclear instructions** that need clarification
- **Missing examples** for documented patterns

Please open an issue with:
- File(s) affected
- Description of the problem
- Suggested fix (if you have one)

### Making Changes

1. **Fork the repository**
2. **Create a feature branch** - `git checkout -b feature/trigger-handler-examples`
3. **Make your changes**
   - Follow existing documentation style
   - Include code examples where appropriate
   - Update cross-references in other files if needed
4. **Test your changes**
   - Ensure all internal links work
   - Verify code examples are syntactically correct
   - Check for consistency with existing patterns
5. **Submit a pull request**
   - Provide clear description of changes
   - Explain why the change improves the standards
   - Reference any related issues

## Documentation Guidelines

### Style Conventions

- **Headers**: Use sentence case (not title case)
- **Code examples**: Always use Allman-style braces
- **Emphasis**: Use **bold** for critical constraints, *italics* for emphasis
- **Lists**: Use `-` for unordered lists, numbers for ordered/sequential steps
- **Links**: Use relative paths for internal links

### Code Examples

- **Complete examples** - Show full class structure (interface, implementation, facade)
- **Realistic examples** - Use meaningful names like `Account`, not `Foo`
- **Comments** - Include comments explaining non-obvious patterns
- **Tested** - Code examples should be syntactically valid

### Organization

- **One topic per file** - Don't mix unrelated patterns
- **Cross-reference** - Link to related topics instead of duplicating content
- **Examples at end** - Keep descriptions concise at top, detailed examples at bottom

## What We're Looking For

### High Priority

- **Pattern gaps** - Common scenarios not yet documented
- **Real-world examples** - Actual implementations you've found useful
- **Clarifications** - Making complex topics easier to understand
- **Consistency improvements** - Aligning terminology and structure

### Lower Priority

- **Minor wording changes** - Only if they significantly improve clarity
- **Reformatting** - Unless it improves readability substantially
- **Personal preferences** - Must have broad applicability

## Review Process

1. **Automated checks** - Links, formatting, syntax validation
2. **Peer review** - At least one maintainer reviews changes
3. **Consistency check** - Ensure changes align with existing standards
4. **Merge** - Changes merged to main branch
5. **Release** - Updated in projects that reference these standards

## Questions?

If you're unsure whether a contribution would be valuable, open an issue first to discuss it before investing time in a pull request.

## Code of Conduct

- **Be respectful** - These standards serve diverse teams
- **Be constructive** - Focus on improving the standards
- **Be collaborative** - Work with maintainers to refine contributions
- **Be patient** - Reviews take time to ensure quality

Thank you for helping make these coding standards better for everyone!
