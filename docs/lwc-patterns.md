# Lightning Web Components Patterns

## LWC Component Patterns

### Deferred Rendering Pattern

When component state depends on DOM rendering completion:

- Store pending changes in a temporary property
- Apply changes in `renderedCallback()` after render completes
- Clear the pending state after applying
- Prevents race conditions with wire service updates

```javascript
@track pendingExpansion = null;

renderedCallback() {
    if (this.pendingExpansion) {
        this.expandedRows = this.pendingExpansion;
        this.pendingExpansion = null;
    }
}
```

### Wire Service Error Handling

Always handle errors from `@wire` decorated methods:

```javascript
@wire(getHierarchyData, { recordId: '$recordId', config: '$parsedConfig' })
wiredHierarchy(result) {
    this.wiredHierarchyResult = result;

    if (result.data) {
        this.processData(result.data);
        this.error = undefined;
    } else if (result.error) {
        this.error = this.reduceErrors(result.error);
        this.data = undefined;
    }
}
```

## @AuraEnabled Controller Patterns

### Controller Method Structure

**Key Patterns:**
- Use `@AuraEnabled(cacheable=true)` for read-only methods (enables LWC wire service)
- Always `static` methods for LWC controllers
- Convert all exceptions to `AuraHandledException` for proper LWC error handling
- Use `auraException.setMessage()` to preserve original error message
- Extract validation logic to private methods
- Throw `IllegalArgumentException` for invalid inputs
- Use `with sharing` to enforce record-level security

```apex
@AuraEnabled(cacheable=true)
public static List<MyWrapper> getData(Id recordId, String config)
{
    try
    {
        validateInputs(recordId, config);
        return processData(recordId, config);
    }
    catch (AuraHandledException e)
    {
        throw e;
    }
    catch (Exception e)
    {
        AuraHandledException auraException = new AuraHandledException(e.getMessage());
        auraException.setMessage(e.getMessage());
        throw auraException;
    }
}

private static void validateInputs(Id recordId, String config)
{
    if (recordId == null)
    {
        throw new IllegalArgumentException('recordId cannot be null');
    }

    if (String.isBlank(config))
    {
        throw new IllegalArgumentException('config cannot be blank');
    }
}
```

### Wrapper/DTO Classes for LWC

**Rules:**
- All properties exposed to LWC must have `@AuraEnabled`
- Use explicit `{ get; set; }` declarations for properties
- Inner classes for nested data structures
- Constructor parameters don't need `@AuraEnabled`

```apex
public class MyWrapper
{
    @AuraEnabled
    public Id recordId { get; set; }

    @AuraEnabled
    public String name { get; set; }

    @AuraEnabled
    public List<ChildWrapper> children { get; set; }

    public MyWrapper(Id recordId, String name)
    {
        this.recordId = recordId;
        this.name = name;
        this.children = new List<ChildWrapper>();
    }

    public class ChildWrapper
    {
        @AuraEnabled
        public String label { get; set; }

        @AuraEnabled
        public Object value { get; set; }
    }
}
```

### Cacheable vs Non-Cacheable

**Use `cacheable=true` when:**
- Method is read-only (no DML)
- Used with `@wire` service
- Data doesn't change frequently
- Same inputs always return same results

**Do NOT use `cacheable=true` when:**
- Method performs DML operations
- Results vary based on user context
- Need real-time data
- Called from imperative JavaScript

```apex
// Cacheable - read-only, used with @wire
@AuraEnabled(cacheable=true)
public static List<Account> getAccounts()
{
    return AccountSelector.newInstance().selectAll();
}

// Not cacheable - performs DML
@AuraEnabled
public static void updateAccount(Id accountId, String newName)
{
    AccountService.updateAccountName(accountId, newName);
}
```

## Best Practices

### Controller Organization

```apex
public with sharing class MyController
{
    // =======================================================================
    // PUBLIC METHODS
    // =======================================================================

    @AuraEnabled(cacheable=true)
    public static List<MyWrapper> getData(Id recordId)
    {
        try
        {
            validateRecordId(recordId);
            return processData(recordId);
        }
        catch (AuraHandledException e)
        {
            throw e;
        }
        catch (Exception e)
        {
            throw createAuraException(e);
        }
    }

    @AuraEnabled
    public static void saveData(Id recordId, String data)
    {
        try
        {
            validateInputs(recordId, data);
            MyService.saveData(recordId, data);
        }
        catch (AuraHandledException e)
        {
            throw e;
        }
        catch (Exception e)
        {
            throw createAuraException(e);
        }
    }

    // =======================================================================
    // PRIVATE HELPER METHODS
    // =======================================================================

    private static void validateRecordId(Id recordId)
    {
        if (recordId == null)
        {
            throw new IllegalArgumentException('recordId cannot be null');
        }
    }

    private static void validateInputs(Id recordId, String data)
    {
        validateRecordId(recordId);

        if (String.isBlank(data))
        {
            throw new IllegalArgumentException('data cannot be blank');
        }
    }

    private static AuraHandledException createAuraException(Exception e)
    {
        AuraHandledException auraException = new AuraHandledException(e.getMessage());
        auraException.setMessage(e.getMessage());
        return auraException;
    }

    private static List<MyWrapper> processData(Id recordId)
    {
        // Implementation
        return new List<MyWrapper>();
    }
}
```

### Error Handling in LWC

```javascript
// In LWC component
import { LightningElement, api, wire } from 'lwc';
import getData from '@salesforce/apex/MyController.getData';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class MyComponent extends LightningElement {
    @api recordId;
    error;
    data;

    @wire(getData, { recordId: '$recordId' })
    wiredData({ error, data }) {
        if (data) {
            this.data = data;
            this.error = undefined;
        } else if (error) {
            this.error = error;
            this.data = undefined;
            this.showErrorToast(error);
        }
    }

    showErrorToast(error) {
        const evt = new ShowToastEvent({
            title: 'Error',
            message: this.reduceErrors(error),
            variant: 'error'
        });
        this.dispatchEvent(evt);
    }

    reduceErrors(error) {
        if (!error) {
            return 'Unknown error';
        }
        if (Array.isArray(error.body)) {
            return error.body.map(e => e.message).join(', ');
        }
        if (error.body && typeof error.body.message === 'string') {
            return error.body.message;
        }
        if (typeof error.message === 'string') {
            return error.message;
        }
        return JSON.stringify(error);
    }
}
```

## Testing Controllers

```apex
@IsTest
private class MyControllerTest
{
    @IsTest
    static void testGetDataSuccess()
    {
        // Given
        Account testAccount = new Account(Name = 'Test');
        insert testAccount;

        // When
        Test.startTest();
        List<MyController.MyWrapper> results = MyController.getData(testAccount.Id);
        Test.stopTest();

        // Then
        Assert.isNotNull(results, 'Results should not be null');
    }

    @IsTest
    static void testGetDataNullRecordId()
    {
        // When/Then
        try
        {
            Test.startTest();
            MyController.getData(null);
            Test.stopTest();

            Assert.fail('Expected IllegalArgumentException');
        }
        catch (IllegalArgumentException e)
        {
            Assert.isTrue(e.getMessage().contains('recordId'), 'Error should mention recordId');
        }
    }

    @IsTest
    static void testSaveDataSuccess()
    {
        // Given
        Account testAccount = new Account(Name = 'Test');
        insert testAccount;

        // When
        Test.startTest();
        MyController.saveData(testAccount.Id, 'Test Data');
        Test.stopTest();

        // Then - verify changes were saved
        // Add assertions here
    }
}
```
