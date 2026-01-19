# Architecture

## Philosophy

All Salesforce projects follow **Enterprise Apex Patterns (fflib)** with strict separation of concerns and dependency injection via custom metadata.

## Core Patterns

### 1. Enterprise Apex Patterns (fflib) - Four-Layer Separation

**Never mix layers.** Each layer has a specific responsibility:

#### Selector Layer (`selectors/`, `selectors/tests/`)

All SOQL queries, FLS enforcement via `fflib_QueryFactory`

- Factory pattern: `XyzSelector.newInstance()` delegates to `Application.Selector`
- Methods named descriptively: `selectByAccountId()`, `selectActiveRecords()`, `selectWithRelatedData()`
- Use `Database.countQuery()` for count operations (fflib standard)
- Return `Database.QueryLocator` for batch-compatible queries:  `queryLocatorById(Set<Id>)`

#### Domain Layer (`domains/`, `domains/tests/`)

SObject collection logic, validation, and calculations

- Interface: `IXyzDomain` (contract for dependency injection)
- Implementation: `XyzDomain` extends `fflib_SObjects` or `fflibExt_SObjects`
- Constructor: Implements `fflib_IDomainConstructor` for factory pattern
- Factory: `Application.Domain.newInstance(recordList)` or `Application.Domain.newInstance(recordIdSet)`
- Encapsulates collection-based operations on records (filtering, transformations, calculations)
- Does NOT perform SOQL queries or DML operations directly
- Static factory methods: `newInstance(List<SObject>)` and `newInstance(Set<Id>)`

#### Service Layer (`services/`, `services/tests/`)

Business logic only, NO direct SOQL or DML

- Interface: `IXyzService` (contract for dependency injection)
- Implementation: `XyzServiceImpl` (actual business logic)
- Facade: `XyzService` (static entry point for consumers)
- Factory: `Application.Service.newInstance(IXyzService.class)`
- All data access delegated to Selectors
- All database operations via `Application.UnitOfWork` (DML isolation)
- **Exception handling**: Prefer throwing exceptions to Consumer layer; only log when intentionally handling errors (e.g., warning for bad config item to avoid failing entire process)
- **Providers**: Use same pattern for external integrations (e.g., `IWeatherProvider`, `WeatherProviderImpl`, `WeatherProvider`) - isolates external dependencies from core business logic

#### Consumer Layer (`batch/`, `queueable/`, `controllers/`, `triggers/`, `invocables/`)

Thin orchestrators

- Delegates all business logic to Service layer
- Delegates all queries to Selector layer
- Handles only async context (savepoints, logging, chaining)
- Invocables expose services to Flow with `@InvocableMethod`

## External Dependencies

This architecture relies on:
- **fflib-apex-common** - Enterprise patterns framework (Service/Selector/Domain/UnitOfWork)
- **fflib-apex-mocks** - Mocking framework for unit tests
- **Nebula Logger** - Production-grade logging with transaction tracking
- **sf-apex-common** (custom) - Enhanced fflib extensions (`fflibExt_SObjects`, `fflibExt_SObjectSelector`), common selectors (AsyncApexJobs, RecordTypes), and utilities (SObjectType, RecordType)


## Supporting Components

While not part of the four core layers, these components play important roles:

### Utils (`utils/`)

Static utility classes providing reusable helper methods

- **Always use `inherited sharing`** to respect caller context
- **Stateless** - no instance variables, all methods static
- **Domain-focused** - organized by concern (e.g., `RecordTypeUtils`, `SObjectTypeUtils`, `DateUtils`)
- **No business logic** - pure utility functions only
- Called directly by any layer when needed

Example:
```apex
public inherited sharing class RecordTypeUtils
{
    public static String getRecordTypeDeveloperName(Id recordId)
    {
        // Utility implementation
    }
}
```

### DTOs/Wrappers

Data Transfer Objects that facilitate communication between layers

- **Not a layer** - helpers for passing data between layers
- Structure non-SObject data for use throughout the application
- Commonly used for API requests/responses, but also for any complex data structures
- Simplify data exchange and encapsulate related information
- Typically defined by Services, returned to Controllers/REST resources
- Suffix with `Wrapper` or `DTO` recommended for clarity, but not required

Example:
```apex
// Service defines the data structure
public class AccountData
{
    @AuraEnabled
    public Id accountId { get; set; }

    @AuraEnabled
    public String name { get; set; }

    @AuraEnabled
    public List<ContactData> contacts { get; set; }
}
```

### 2. Application Service Factory Pattern

All Services, Selectors, and Domains use **dependency injection via Custom Metadata**:

```apex
// Service usage - call static facade methods
MyService.processRecords(recordIds);

// Selector usage - factory method returns typed interface
IMySelector selector = MySelector.newInstance();
List<MyObject__c> records = selector.selectById(recordIds);

// Domain usage - factory method returns typed interface
IMyDomain domain = MyDomain.newInstance(recordList);
domain.performValidation();
```

**Internal implementation uses Application factories:**

```apex
// Inside Service facade class:
private static IMyService service()
{
    return (IMyService) Application.Service.newInstance(IMyService.class);
}

// Inside Selector factory method:
public static IMySelector newInstance()
{
    return (IMySelector) Application.Selector.newInstance(MyObject__c.SObjectType);
}

// Inside Domain factory methods:
public static IMyDomain newInstance(List<MyObject__c> recordList)
{
    return (IMyDomain) Application.Domain.newInstance(recordList);
}

public static IMyDomain newInstance(Set<Id> recordIdSet)
{
    return (IMyDomain) Application.Domain.newInstance(recordIdSet);
}
```

**Configuration via `ApplicationBinding__mdt` custom metadata:**

- `BindingType__c`: "Service", "Selector", "Domain", or "UnitOfWork"
- `Source__c`: Interface name (e.g., "IMyService"), SObjectType (e.g., "Account"), or object API name
- `Target__c`: Implementation class (e.g., "MyServiceImpl", "MySelector", "MyDomain.Constructor")
- `Priority__c`: Order for multiple implementations (optional)
- `IsActive__c`: Boolean flag to enable/disable binding

**Benefits:** Mockable in tests, swappable implementations, zero code changes for different environments

### 3. Unit of Work Pattern for DML

**Always use `Application.UnitOfWork` for database operations in services:**

```apex
public void processRecords(List<Account> accounts)
{
    fflib_ISObjectUnitOfWork uow = Application.UnitOfWork.newInstance();

    for (Account acc : accounts)
    {
        // Business logic modifications
        uow.registerDirty(acc);
    }

    // Commit all changes in single transaction
    uow.commitWork();

    Logger.saveLog();
}
```

**Key Benefits:**
- Bulkified DML operations
- Automatic ordering of DML operations
- Transaction consistency
- Easier mocking in tests

## Layer Implementation Examples

### Selector Layer Example

**Interface** (`IAccountSelector.cls`):
```apex
public interface IAccountSelector extends fflib_ISObjectSelector
{
    List<Account> selectById(Set<Id> accountIds);
    List<Account> selectByStatus(Set<String> statuses);
    Database.QueryLocator queryLocatorByStatus(Set<String> statuses);
}
```

**Implementation** (`AccountSelector.cls`):
```apex
public inherited sharing class AccountSelector extends fflib_SObjectSelector implements IAccountSelector
{
    public AccountSelector()
    {
        super();
    }

    public Schema.SObjectType getSObjectType()
    {
        return Account.SObjectType;
    }

    public List<Schema.SObjectField> getSObjectFieldList()
    {
        return new List<Schema.SObjectField>
        {
            Account.Id,
            Account.Name,
            Account.Status__c,
            Account.Industry
        };
    }

    public List<Account> selectById(Set<Id> accountIds)
    {
        return (List<Account>) selectSObjectsById(accountIds);
    }

    public List<Account> selectByStatus(Set<String> statuses)
    {
        return Database.query(
            newQueryFactory() // Inherited from fflib_SObjectSelector base class
                .setCondition('Status__c IN :statuses')
                .toSOQL()
        );
    }

    // For batch-compatible queries
    public Database.QueryLocator queryLocatorByStatus(Set<String> statuses)
    {
        return Database.getQueryLocator(
            newQueryFactory()
                .setCondition('Status__c IN :statuses')
                .toSOQL()
        );
    }

    public static IAccountSelector newInstance()
    {
        return (IAccountSelector) Application.Selector.newInstance(Account.SObjectType);
    }
}
```

### Domain Layer Example

**Trigger** (thin orchestrator, one per object):
```apex
trigger AccountTrigger on Account (before insert, before update, after insert, after update)
{
    fflib_SObjectDomain.triggerHandler(AccountDomain.class);
}
```

**Interface** (`IAccountDomain.cls`):
```apex
public interface IAccountDomain extends fflib_ISObjectDomain
{
    void applyDefaults();
    void validateCreditLimit();
}
```

**Implementation** (`AccountDomain.cls`):
```apex
public inherited sharing class AccountDomain extends fflib_SObjects implements IAccountDomain
{
    public AccountDomain(List<Account> records)
    {
        super(records);
    }

    // Called by trigger on before insert
    public override void onBeforeInsert()
    {
        applyDefaults();
        validateCreditLimit();
    }

    // Called by trigger on before update
    public override void onBeforeUpdate(Map<Id, SObject> existingRecords)
    {
        Map<Id, Account> oldMap = (Map<Id, Account>) existingRecords;

        validateCreditLimit();

        // Detect field changes
        List<Account> accountsWithNameChange = new List<Account>();
        for (Account acc : (List<Account>) Records)
        {
            if (acc.Name != oldMap.get(acc.Id).Name)
            {
                accountsWithNameChange.add(acc);
            }
        }

        if (!accountsWithNameChange.isEmpty())
        {
            // Delegate complex logic to Service layer
            AccountService.handleNameChanges(accountsWithNameChange, oldMap);
        }
    }

    public void applyDefaults()
    {
        for (Account acc : (List<Account>) Records)
        {
            if (String.isBlank(acc.Industry))
            {
                acc.Industry = 'Other';
            }
        }
    }

    public void validateCreditLimit()
    {
        for (Account acc : (List<Account>) Records)
        {
            if (acc.CreditLimit__c != null && acc.CreditLimit__c < 0)
            {
                acc.addError('Credit limit cannot be negative');
            }
        }
    }

    public class Constructor implements fflib_IDomainConstructor
    {
        public fflib_ISObjectDomain construct(List<SObject> sObjectList)
        {
            return new AccountDomain(sObjectList);
        }
    }

    public static IAccountDomain newInstance(List<Account> records)
    {
        return (IAccountDomain) Application.Domain.newInstance(records);
    }

    public static IAccountDomain newInstance(Set<Id> recordIds)
    {
        return (IAccountDomain) Application.Domain.newInstance(recordIds);
    }
}
```

### Service Layer Example

**Interface** (`IAccountService.cls`):
```apex
public interface IAccountService
{
    void processAccountsByStatus(Set<String> statuses);
}
```

**Implementation** (`AccountServiceImpl.cls`):
```apex
public inherited sharing class AccountServiceImpl implements IAccountService
{
    public void processAccountsByStatus(Set<String> statuses)
    {
        // Delegate query to Selector
        List<Account> accounts = AccountSelector.newInstance()
            .selectByStatus(statuses);

        // Use Unit of Work for DML
        fflib_ISObjectUnitOfWork uow = Application.UnitOfWork.newInstance();

        // Business logic
        for (Account acc : accounts)
        {
            acc.LastProcessedDate__c = Date.today();
            uow.registerDirty(acc);
        }

        uow.commitWork();
    }
}
```

**Facade** (`AccountService.cls`):
```apex
public inherited sharing class AccountService
{
    public static void processAccountsByStatus(Set<String> statuses)
    {
        service().processAccountsByStatus(statuses);
    }

    private static IAccountService service()
    {
        return (IAccountService) Application.Service.newInstance(IAccountService.class);
    }
}
```
