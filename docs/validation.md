# Code Quality Validation

## Pre-Commit Validation

**Always run the Salesforce Code Analyzer before committing code:**

```powershell
# Run on entire workspace
sf code-analyzer run --view table

# Run on specific folder (recommended - avoids libs/ submodules)
sf code-analyzer run --target "force-app"
```

## What It Validates

This validates:
- **PMD rules** - Code quality and best practices
- **ApexDoc violations** - Missing or incomplete documentation
- **Security vulnerabilities** - Potential security issues
- **Performance issues** - Anti-patterns and inefficiencies

## Pre-Commit Workflow

**Address all warnings and errors** before committing:

1. Make code changes
2. Run `sf code-analyzer run` to validate
3. Address all warnings/errors
4. Re-run analyzer to confirm clean results
5. Commit code

## Critical Rules

### Never Suppress Warnings as AI

**NEVER add `@SuppressWarnings` - report violations to human for review**

Only human developers can add `@SuppressWarnings` annotations after careful review. AI must:
- ❌ DO NOT add @SuppressWarnings even if you think it's "legitimate"
- ❌ DO NOT add @SuppressWarnings to "clean up" PMD violations
- ✅ DO report PMD violations and let the human decide
- ✅ DO fix actual code issues when possible

### Address Legitimate Issues

Fix real code problems identified by the analyzer:
- Add missing ApexDoc comments (see [Code Style](code-style.md#comments-and-documentation))
- Fix security vulnerabilities
- Correct performance anti-patterns
- Ensure proper error handling
- Verify sharing declarations (see [Sharing Model](sharing-model.md))

## Common Violations

### Missing ApexDoc

**Problem:** Public methods without documentation

**Solution:** Follow ApexDoc requirements in [Code Style](code-style.md#comments-and-documentation)

### Avoid Hardcoded IDs

**Problem:** Hardcoded record IDs in code

**Solution:** Use custom metadata, custom settings, or dynamic queries

### SOQL in Loops

**Problem:** SOQL query inside a loop

**Solution:** Bulkify - query once before the loop

### DML in Loops

**Problem:** DML operation inside a loop

**Solution:** Use Unit of Work pattern to collect and commit changes

## IDE Integration

Most IDEs can integrate Code Analyzer for real-time feedback:
- VS Code: Install Salesforce Extension Pack
- IntelliJ: Install Illuminated Cloud 2
- Sublime Text: Install MavensMate (deprecated) or ApexPMD plugin

## Continuous Integration

Add Code Analyzer to your CI/CD pipeline:

```yaml
# Example GitHub Actions workflow
- name: Run Code Analyzer
  run: sf code-analyzer run --target "force-app" --severity-threshold 3
```

This fails the build if violations exceed severity threshold.

## Additional Resources

- [Salesforce Code Analyzer Documentation](https://forcedotcom.github.io/sfdx-scanner/)
- [PMD Rule Reference](https://pmd.github.io/latest/pmd_rules_apex.html)
