# Code Style

## Naming Conventions

- **Classes**: `PascalCase` (e.g., `AccountService`, `ContactSelector`)
- **Methods/Variables**: `camelCase` (e.g., `processAccounts()`, `recordCount`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `BATCH_SIZE`, `MAX_RETRIES`)
- **Interfaces**: Prefix with `I` (e.g., `IAccountService`, `IContactSelector`)
- **Test Classes**: Suffix with `Test` (e.g., `AccountServiceTest`, `ContactSelectorTest`)
- **Batch Classes**: Suffix with `Batch` or `Job` (e.g., `AccountCleanupBatch`, `DataMigrationJob`)
- **Queueable Classes**: Suffix with `Queueable` (e.g., `EmailSendQueueable`, `RecordProcessQueueable`)
- **Schedulable Classes**: Suffix with `Schedulable` (e.g., `DailyReportSchedulable`)
- **Invocable Classes**: Suffix with `Action` or `Invocable` (e.g., `CreateRecordAction`, `SendEmailInvocable`)
- **Controller Classes**: Suffix with `Controller` (e.g., `AccountController`, `DashboardController`)
- **Exceptions**: Suffix with `Exception` (e.g., `MergeServiceException`, `ValidationException`)
- **Trigger Handlers**: Suffix with `TriggerHandler` (e.g., `AccountTriggerHandler`)
- **DTO/Wrapper Classes**: Suffix with `Wrapper` or `DTO` recommended for clarity (e.g., `AccountWrapper`, `RequestDTO`)

## Allman-Style Braces

**Always use Allman style** (opening brace on new line):

```apex
// CORRECT
public void processRecords()
{
    if (records.isEmpty())
    {
        return;
    }

    for (Account acc : accounts)
    {
        // process
    }
}

// WRONG - K&R style not allowed
public void processRecords() {
    if (records.isEmpty()) {
        return;
    }
}
```

**Never mix K&R and Allman braces** - consistent Allman style only.

## Code Organization Patterns

### Method Grouping with Regions

**Use `#region` for Method Grouping** (especially for overloaded methods):

```apex
public class SObjectTypeUtils
{
    //#region SObjectType methods
    //
    // getSObjectType(String)
    // getSObjectType(List<SObject>)
    // getSObjectType(Set<SObject>)
    //
    public static Schema.SObjectType getSObjectType(String sObjectName) { }
    public static Schema.SObjectType getSObjectType(List<SObject> records) { }
    public static Schema.SObjectType getSObjectType(Set<SObject> records) { }
    //#endregion

    //#region SObjectField methods
    // ...
    //#endregion
}
```

### Fluent API / Method Chaining

Return `this` or interface type to enable method chaining:

```apex
// Interface defines return type
public interface IDynamicSelector extends fflibExt_ISObjectSelector
{
    IDynamicSelector addField(String fieldName);
    IDynamicSelector addChild(String childRelationshipName);
}

// Implementation returns this
public IDynamicSelector addField(String fieldName)
{
    this.fieldList.add(fieldName);
    return this; // Enable chaining
}

// Usage
selector.addField('Name')
    .addField('Industry')
    .addChild('Contacts');
```

### Null-Safe Navigation

Use `?.` operator consistently for safe property access:

```apex
// CORRECT - safe navigation
String devName = records.get(0)?.getSObject('RecordType')?.get('DeveloperName');
SObjectType sObjType = records.get(0)?.getSObjectType();

// WRONG - can throw NullPointerException
String devName = records.get(0).getSObject('RecordType').get('DeveloperName');
```

## Comments and Documentation

### ApexDoc Requirements

**All public classes, methods, and properties MUST have ApexDoc comments** using proper JavaDoc tags:

```apex
/**
 * @description Brief description of class or method purpose
 * @param paramName Description of parameter
 * @return Description of return value
 * @throws ExceptionType When and why this exception is thrown
 * @author Author Name (optional, for significant contributions)
 */
```

**Required Tags:**
- `@description` - REQUIRED for all public classes and methods
- `@param` - REQUIRED for each method parameter
- `@return` - REQUIRED for methods with return values (except void)
- `@throws` - REQUIRED when method can throw exceptions

**Code Analyzer Validation:**
The Salesforce Code Analyzer enforces ApexDoc standards. Missing or incomplete documentation will generate warnings that must be resolved before committing.

### Class-Level Documentation

Include examples for complex APIs:

```apex
/**
 * @description Utility class for processing account hierarchies
 *
 * Example Usage:
 * <code>
 * List<Account> accounts = AccountProcessor.processHierarchy(accountIds);
 * </code>
 */
```

### Inline Comments

- Explain *why*, not *what* (code shows what)
- Use TODO comments for planned improvements: `// TODO: Replace with CallableLogger.error() when Nebula Logger is integrated`
- Mark silent failures with explanation: `// Silent failure - prevents system crashes`

## Additional Patterns

### Custom Exception Classes

Define domain-specific exceptions as **inner classes** for Services, Domains, REST resources, and other classes:

```apex
public class MyService
{
    public class MyServiceException extends Exception {}

    public void validateInput(String input)
    {
        if (String.isBlank(input))
        {
            throw new MyServiceException('Input cannot be blank');
        }
    }
}
```

### PMD Suppression Annotations

**CRITICAL: AI should NEVER apply `@SuppressWarnings` annotations. Only human developers can add suppressions after careful review.**

### Silent Failure with Logging TODOs

When graceful degradation is needed, document future logging plans:

```apex
private static SObjectType getSObjectTypeFromBinding(ApplicationBinding__mdt binding)
{
    try
    {
        return SObjectTypeUtils.getSObjectType(binding.Source__c);
    }
    catch (Exception ex)
    {
        // Silent failure - log with Nebula Logger when available
        // TODO: Replace with CallableLogger.error() when Nebula Logger is integrated
        // For now, silent failure to prevent system crashes
        return null;
    }
}
```
