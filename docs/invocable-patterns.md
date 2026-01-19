# Invocable Patterns

## Overview

Invocables expose Service layer functionality to declarative automation (Flows, Process Builder). They act as thin adapters between Flow and Service methods, handling input/output formatting and error handling.

## Key Practices

- **Always use `inherited sharing`** - Invocables inherit sharing from the calling Flow
- Define `Request` and `Result` as inner classes
- Define custom exception as inner class for invocable-specific errors
- Provide clear `label`, `description`, and `category` for `@InvocableMethod`
- Provide descriptive labels and descriptions for all `@InvocableVariable` fields
- Handle exceptions gracefully - return errors in `Result` unless you want Flow to fail
- Delegate all business logic to Service layer
- Support batch operations when appropriate (Flow can pass multiple requests)

## Standard Invocable Pattern

```apex
/**
 * @description Invocable action for [functionality description]
 * Exposes [ServiceName] methods to declarative automation
 */
public inherited sharing class MyInvocable
{
    /**
     * @description [Method description]
     * @param requests List of requests from Flow
     * @return List of results indicating success/failure
     */
    @InvocableMethod(
        label='[Action Label]'
        description='[Detailed description of what this action does]'
        category='[Category Name]'
    )
    public static List<Result> processRecords(List<Request> requests)
    {
        List<Result> results = new List<Result>();

        try
        {
            // Validate all inputs
            validateRequests(requests);

            // Collect data from requests (Services handle bulk operations)
            List<Id> recordIds = new List<Id>();
            for (Request request : requests)
            {
                recordIds.add(request.recordId);
            }

            // Delegate to Service layer (handles sync or async internally)
            Map<Id, String> outcomes = MyService.processRecords(recordIds);

            // Build success result
            Result result = new Result();
            result.isSuccess = true;
            result.message = String.format('Processed {0} records', new List<Object>{outcomes.size()});
            result.processedCount = outcomes.size();
            results.add(result);
        }
        catch (Exception ex)
        {
            // Return error in result (Flow can check isSuccess)
            Result result = new Result();
            result.isSuccess = false;
            result.message = 'Error: ' + ex.getMessage();
            results.add(result);
        }

        return results;
    }

    /**
     * @description Validates request list and individual request data
     * @param requests The requests to validate
     */
    private static void validateRequests(List<Request> requests)
    {
        if (requests == null || requests.isEmpty())
        {
            throw new MyInvocableException('Requests cannot be null or empty');
        }

        for (Request request : requests)
        {
            if (request == null)
            {
                throw new MyInvocableException('Request cannot be null');
            }

            if (request.recordId == null)
            {
                throw new MyInvocableException('Record ID is required');
            }
        }
    }

    /**
     * @description Input parameters for the invocable method
     */
    public class Request
    {
        @InvocableVariable(
            label='Record ID'
            description='ID of the record to process'
            required=true
        )
        public Id recordId;

        @InvocableVariable(
            label='Data'
            description='Additional data for processing'
            required=false
        )
        public String data;
    }

    /**
     * @description Output result from the invocable method
     */
    public class Result
    {
        @InvocableVariable(
            label='Success'
            description='True if the operation completed successfully'
        )
        public Boolean isSuccess;

        @InvocableVariable(
            label='Message'
            description='Result message with details or error information'
        )
        public String message;

        @InvocableVariable(
            label='Processed Count'
            description='Number of records processed'
        )
        public Integer processedCount;
    }

    /** @description Custom exception for invocable errors */
    public class MyInvocableException extends Exception {}
}
```

**Note:** Service methods should accept `List<>` parameters for bulk operations. Whether the service runs synchronously, submits a batch job, or enqueues work is an implementation detail - the invocable simply delegates to the service.

## Error Handling Strategies

### Return Errors in Result (Recommended)

Flow can check `isSuccess` and branch accordingly:

```apex
catch (Exception ex)
{
    Result result = new Result();
    result.isSuccess = false;
    result.message = 'Error: ' + ex.getMessage();
    results.add(result);
}
```

### Throw Exception to Fail Flow

When you want Flow to fail immediately:

```apex
catch (Exception ex)
{
    throw new MyInvocableException('Critical error: ' + ex.getMessage());
}
```

## Testing Invocables

```apex
@IsTest
private class MyInvocableTest
{
    @IsTest
    static void testProcessRecordsSuccess()
    {
        // Given
        List<Account> testAccounts = new List<Account>();
        for (Integer i = 0; i < 3; i++)
        {
            testAccounts.add(new Account(Name = 'Test ' + i));
        }
        insert testAccounts;

        List<MyInvocable.Request> requests = new List<MyInvocable.Request>();
        for (Account acc : testAccounts)
        {
            MyInvocable.Request request = new MyInvocable.Request();
            request.recordId = acc.Id;
            request.data = 'Test Data';
            requests.add(request);
        }

        // When
        Test.startTest();
        List<MyInvocable.Result> results = MyInvocable.processRecords(requests);
        Test.stopTest();

        // Then
        Assert.areEqual(1, results.size(), 'Should return 1 result');
        Assert.isTrue(results[0].isSuccess, 'Result should be successful');
        Assert.areEqual(3, results[0].processedCount, 'Should process all 3 records');
    }

    @IsTest
    static void testProcessRecordsNullRequest()
    {
        // When
        Test.startTest();
        List<MyInvocable.Result> results = MyInvocable.processRecords(null);
        Test.stopTest();

        // Then
        Assert.areEqual(1, results.size(), 'Should return 1 result');
        Assert.isFalse(results[0].isSuccess, 'Result should indicate failure');
        Assert.isTrue(results[0].message.contains('Error'), 'Should contain error message');
    }

    @IsTest
    static void testProcessRecordsInvalidRecordId()
    {
        // Given
        MyInvocable.Request request = new MyInvocable.Request();
        request.recordId = null;

        // When
        Test.startTest();
        List<MyInvocable.Result> results = MyInvocable.processRecords(
            new List<MyInvocable.Request>{ request }
        );
        Test.stopTest();

        // Then
        Assert.areEqual(1, results.size(), 'Should return 1 result');
        Assert.isFalse(results[0].isSuccess, 'Result should indicate failure');
        Assert.isTrue(results[0].message.contains('Record ID'), 'Error should mention Record ID');
    }
}
```

## Best Practices

- **Always delegate to Service layer** - Keep invocables thin, Service decides sync vs async execution
- **Service methods accept List<>** - Invocables collect data and pass to Service bulk methods
- **Use `inherited sharing`** - Respects Flow's execution context
- **Provide clear labels** - Makes Flow Builder user-friendly
- **Handle errors gracefully** - Return `isSuccess = false` rather than throwing (unless you want Flow to fail)
- **Validate inputs** - Never trust Flow data without validation
- **Document thoroughly** - Use ApexDoc for all public APIs
- **Test all scenarios** - Success, validation errors, exceptions
- **Return meaningful messages** - Help Flow developers debug issues
