# Testing

## Philosophy

Testing is critical for maintaining code quality, preventing regressions, and enabling confident refactoring. All code should be thoroughly tested before deployment.

## Test Structure and Organization

### Given/When/Then Pattern

**Required for all tests** (Apex and LWC):

```javascript
// LWC Jest Example
it('should display account name when data is loaded', () => {
    // Given
    const mockAccount = { Name: 'Test Account', Industry: 'Technology' };

    // When
    element.account = mockAccount;

    // Then
    const nameElement = element.shadowRoot.querySelector('.account-name');
    expect(nameElement.textContent).toBe('Test Account');
});
```

```apex
// Apex Example
@IsTest
static void testProcessAccountSuccess()
{
    // Given
    Account testAccount = new Account(Name = 'Test');
    insert testAccount;

    // When
    Test.startTest();
    MyService.processAccount(testAccount.Id);
    Test.stopTest();

    // Then
    Account result = [SELECT Status__c FROM Account WHERE Id = :testAccount.Id];
    Assert.areEqual('Processed', result.Status__c, 'Account should be processed');
}
```

**Why Given/When/Then:**
- **Given** - Sets up test data and preconditions
- **When** - Executes the code being tested
- **Then** - Verifies expected outcomes

This pattern makes tests readable, maintainable, and clearly documents intent.

### Test Organization

Organize tests by functionality:

```javascript
// LWC Jest - describe blocks
describe('MyComponent', () => {
    describe('Data Loading', () => {
        it('should display loading spinner initially', () => { });
        it('should hide spinner when data loads', () => { });
    });

    describe('User Interactions', () => {
        it('should handle button click', () => { });
        it('should validate form input', () => { });
    });
});
```

```apex
// Apex - section separators
@IsTest
private class MyServiceTest
{
    // =======================================================================
    // CONSTRUCTOR TESTS
    // =======================================================================

    @IsTest
    static void testConstructorValidation() { }

    // =======================================================================
    // BUSINESS LOGIC TESTS
    // =======================================================================

    @IsTest
    static void testProcessRecords() { }
}
```

## Universal Best Practices

These principles apply to both Apex and LWC testing:

### Test Coverage Goals

- **Quality over quantity** - 100% coverage with poor assertions is worthless
- **Test behavior, not implementation** - Focus on what the code does, not how
- **One assertion per test** when possible - Makes failures obvious and specific
- **Meaningful assertions** - Always include descriptive failure messages

### Test Isolation

- **Each test should be independent** - No shared state between tests
- **Mock external dependencies** - Services, APIs, platform events
- **Clean up after yourself** - Reset mocks, clear state in afterEach/teardown
- **Don't rely on execution order** - Tests should pass in any order

### Edge Cases and Exceptions

- **Test happy path AND error cases** - Success is only half the story
- **Null inputs** - Always test with null/undefined values
- **Empty collections** - Test with empty arrays/lists
- **Boundary conditions** - Test min/max values, limits
- **Exception handling** - Verify proper error handling and messages

### Descriptive Test Names

Use clear, descriptive test names that explain what is being tested:

```javascript
// GOOD
it('should throw error when account ID is null')
it('should display 3 contacts when account has 3 contacts')

// BAD
it('test1')
it('works')
```

```apex
// GOOD
@IsTest
static void testProcessRecordsThrowsExceptionWhenRecordIdIsNull() { }

// BAD
@IsTest
static void test1() { }
```

### Performance Considerations

- **Keep tests fast** - Slow tests discourage running them frequently
- **Minimize DML in Apex tests** - Use mocks when possible (see [testing-apex.md](testing-apex.md))
- **Avoid unnecessary DOM queries in LWC** - Cache element references
- **Use @TestSetup in Apex** - Shared test data for better performance
- **Parallel execution** - Structure tests to run in parallel when possible

## Platform-Specific Testing

For detailed testing patterns and frameworks:

- **[Apex Testing](testing-apex.md)** - Apex test classes, mocking with fflib_ApexMocks, test data factories
- **[LWC Testing](testing-lwc.md)** - Jest configuration, mocking wire services, DOM testing

## Continuous Integration

Tests should run automatically in CI/CD pipelines:

- Run all tests before merging to main branch
- Fail builds on test failures
- Track code coverage trends over time
- Run tests in scratch orgs to catch org-specific issues

## Best Practices Summary

✅ **DO:**
- Write tests for all new code
- Use Given/When/Then structure
- Test edge cases and error conditions
- Mock external dependencies
- Use descriptive test names
- Keep tests independent and isolated
- Run tests frequently during development

❌ **DON'T:**
- Write tests just for coverage percentage
- Share state between tests
- Test implementation details
- Skip error case testing
- Use vague test names like "test1"
- Rely on test execution order
- Commit code without running tests
