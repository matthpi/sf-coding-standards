# Apex Testing

For general testing principles and best practices that apply to both Apex and LWC, see [Testing](testing.md).

## Modern Assert Methods

**Always use `Assert` class**, not deprecated `System.assert*`:

```apex
// CORRECT - modern syntax
Assert.areEqual(expected, actual, 'Values should match');
Assert.isNotNull(record, 'Record should exist');
Assert.isTrue(condition, 'Condition should be true');
Assert.isFalse(condition, 'Condition should be false');
Assert.isInstanceOfType(obj, MyClass.class, 'Should be MyClass instance');

// WRONG - deprecated methods
System.assertEquals(expected, actual);
System.assertNotEquals(null, record);
System.assert(condition);
```

## Test Structure and Organization

### Folder Organization

Test classes are organized in `tests/` subdirectories within each layer:

```
force-app/main/default/classes/
├── services/
│   ├── AccountService.cls
│   ├── AccountServiceImpl.cls
│   ├── IAccountService.cls
│   └── tests/
│       └── AccountServiceTest.cls
├── selectors/
│   ├── AccountSelector.cls
│   └── tests/
│       └── AccountSelectorTest.cls
├── domains/
│   ├── AccountDomain.cls
│   └── tests/
│       └── AccountDomainTest.cls
├── batch/
│   ├── AccountBatch.cls
│   └── tests/
│       └── AccountBatchTest.cls
```

**Benefits:**
- **Colocation** - Tests stay close to the code they test
- **Easy navigation** - Developers immediately see related tests
- **Consistent pattern** - Matches LWC's `__tests__/` subdirectory approach

### Given/When/Then Pattern

Required for all tests:

```apex
@IsTest
static void testServiceMethod()
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

### Section Separators

Organize test methods by category:

```apex
@IsTest
private class MyServiceTest
{
    // =======================================================================
    // CONSTRUCTOR TESTS
    // =======================================================================

    @IsTest
    static void testConstructorNullParameter() { }

    @IsTest
    static void testConstructorValidParameter() { }

    // =======================================================================
    // BUSINESS LOGIC TESTS
    // =======================================================================

    @IsTest
    static void testProcessRecords() { }
}
```

### Parallel Execution Control

```apex
// Use when tests MUST run sequentially or use shared state
@IsTest(IsParallel=false)
private class StatefulBatchJobTest { }

// Default - allows parallel execution for faster test runs
@IsTest(IsParallel=true)
private class UtilityMethodTest { }
```

**Note:** If you have mostly parallelizable tests with only 1-2 non-parallelizable methods (e.g., 19 parallel + 1 stateful), consider splitting into separate test classes to avoid slowing down the entire test suite. The `IsParallel` attribute applies to the entire class, not individual methods.

## Mocking Dependencies

### Mock fflib Dependencies

Always mock Service and Selector dependencies using `fflib_ApexMocks`:

```apex
@IsTest
static void testServiceMethod()
{
    // Setup mocks
    fflib_ApexMocks mocks = new fflib_ApexMocks();
    IMySelector selectorMock = (IMySelector) mocks.mock(MySelector.class);

    // Stub behavior
    mocks.startStubbing();
    mocks.when(selectorMock.sObjectType()).thenReturn(MyObject__c.SObjectType);
    mocks.when(selectorMock.selectById(new Set<Id>{testId}))
        .thenReturn(new List<MyObject__c>{testRecord});
    mocks.stopStubbing();

    // Inject mock
    Application.Selector.setMock(selectorMock);

    // Test execution
    Test.startTest();
    MyService.processRecords(testId);
    Test.stopTest();

    // Verify interactions
    ((IMySelector) mocks.verify(selectorMock, 1)).selectById(new Set<Id>{testId});
}
```

### Mocking Nebula Logger

For tests that verify logging behavior, use `LoggerMockDataStore`:

```apex
@IsTest
static void testLoggingBehavior()
{
    // Given
    LoggerDataStore.setMock(LoggerMockDataStore.getEventBus());
    LoggerTestConfigurator.setupMockSObjectHandlerConfigurations();

    // When
    MyService.processRecords(testIds);

    // Then
    List<SObject> publishedLogEvents = LoggerMockDataStore.getEventBus().getPublishedPlatformEvents();
    Integer warnLogCount = 0;
    for(SObject event : publishedLogEvents)
    {
        LogEntryEvent__e logEntry = (LogEntryEvent__e) event;
        if(logEntry.LoggingLevel__c == 'WARN' && logEntry.Message__c.contains('Expected message'))
        {
            warnLogCount++;
        }
    }
    Assert.areEqual(1, warnLogCount, 'Expected 1 warning log');
}
```

## Test Data Provider Pattern

**Note:** Test data provider classes are separate utility classes that create test data, not the test classes themselves. Test classes remain `@IsTest private class`.

**Key Practices:**
- Use `@IsTest` **public** for test data provider/factory classes (enables sharing across test classes)
- Use `@IsTest` **private** for actual test classes (e.g., `MyServiceTest`)
- Use `fflib_IDGenerator.generate(SObjectType)` for generating fake Ids without DML
- Use `fflib_ApexMocksUtils.makeRelationship()` to create parent-child relationships
- Use `putSObject()` to manually set relationship objects (e.g., RecordType, Parent)
- Create private helper methods for object creation
- Use constants for commonly reused values
- Static factory methods return pre-configured test objects

```apex
@IsTest
public class TestDataFactory
{
    public static List<Account> createAccountsWithContacts(Integer numContacts)
    {
        Account parent = new Account(
            Id = fflib_IDGenerator.generate(Account.SObjectType),
            Name = 'Test Account'
        );

        List<Contact> contacts = new List<Contact>();
        for(Integer i = 0; i < numContacts; i++)
        {
            contacts.add(new Contact(
                Id = fflib_IDGenerator.generate(Contact.SObjectType),
                AccountId = parent.Id,
                LastName = 'Test ' + i
            ));
        }

        // Create parent-child relationship without DML
        return (List<Account>) fflib_ApexMocksUtils.makeRelationship(
            List<Account>.class,
            new List<Account> { parent },
            Contact.AccountId,
            new List<List<Contact>> { contacts }
        );
    }
}
```

## Test Class Patterns

- Use `@TestVisible` for private fields that need test verification
- Create test data in `@TestSetup` when possible for performance
- Test both success and exception scenarios
- Mock external dependencies (callouts, platform events, etc.)
- Use test data providers for complex test setup
- Add `Assert.isTrue(true, 'Suppress PMD Warning')` to satisfy PMD when no meaningful assertion exists

## Best Practices

- **One assertion per test** when possible (makes failures obvious)
- **Test edge cases**: null inputs, empty collections, boundary conditions
- **Test exception paths**: verify proper exception handling
- **Verify side effects**: check DML operations, callouts, platform events
- **Use descriptive test names**: `testProcessRecordsWithNullInput()` vs `test1()`
- **Keep tests independent**: each test should work in isolation
