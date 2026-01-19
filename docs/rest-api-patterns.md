# REST API Patterns

## Directory Structure

**Structure:**
- `restapis/` - REST resource classes (optional - use in larger projects with multiple APIs)
- `restapis/v1/`, `restapis/v2/` - Version-specific resources for API versioning
- `restapis/tests/` - Test classes for REST resources

## Naming Conventions

- Resource classes suffix with `Resource`: `AccountResource`, `DataSyncResource`
- URL mappings use lowercase with version: `/api/v1/accounts/*`, `/api/v2/datasync/*`
- Exception classes as inner classes: `AccountResourceException`

## Key Practices

- Always use `global with sharing` for REST resources (enforces record-level security)
- Delegate all business logic to Service layer (resources are thin orchestrators)
- Define custom exception as inner class for API-specific errors
- Use `global` for HTTP method handlers and request/response classes
- Use `private` for validation and helper methods
- Always validate requests before processing
- Return meaningful error messages via custom exceptions
- Use HTTP-appropriate status codes (leverage RestContext for advanced scenarios)

## REST Resource Pattern

```apex
@RestResource(urlMapping='/api/v1/myresource/*')
global with sharing class MyResource
{
    @HttpPost
    global static MyResponse processRecords(MyRequest request)
    {
        try
        {
            validateRequest(request);
            List<Id> processedIds = MyService.processRecords(request.recordIds);
            return new MyResponse(true, 'Success', processedIds);
        }
        catch (DmlException e)
        {
            if (e.getMessage().contains('ENTITY_IS_DELETED'))
            {
                // Graceful handling of deleted records
                return new MyResponse(true, 'Records already deleted', new List<Id>());
            }
            else
            {
                throw new MyResourceException(e.getMessage());
            }
        }
        catch(Exception e)
        {
            throw new MyResourceException(e.getMessage());
        }
    }

    private static void validateRequest(MyRequest request)
    {
        if (request == null || request.recordIds == null || request.recordIds.isEmpty())
        {
            throw new MyResourceException('Request must contain recordIds');
        }
    }

    public class MyResourceException extends Exception { }

    global class MyRequest
    {
        public List<Id> recordIds { get; set; }
    }

    global class MyResponse
    {
        public Boolean success { get; set; }
        public String message { get; set; }
        public List<Id> processedIds { get; set; }

        public MyResponse(Boolean success, String message, List<Id> processedIds)
        {
            this.success = success;
            this.message = message;
            this.processedIds = processedIds;
        }
    }
}
```

## HTTP Methods

### GET - Retrieve Data

```apex
@HttpGet
global static MyResponse getData()
{
    try
    {
        String recordId = RestContext.request.params.get('id');
        validateRecordId(recordId);

        MyObject__c record = MySelector.newInstance().selectById(new Set<Id>{recordId})[0];
        return new MyResponse(true, 'Success', record);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

### POST - Create Records

```apex
@HttpPost
global static MyResponse createRecord(CreateRequest request)
{
    try
    {
        validateCreateRequest(request);
        Id newRecordId = MyService.createRecord(request.name, request.type);
        return new MyResponse(true, 'Record created', newRecordId);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

### PUT - Update Records

```apex
@HttpPut
global static MyResponse updateRecord(UpdateRequest request)
{
    try
    {
        validateUpdateRequest(request);
        MyService.updateRecord(request.recordId, request.data);
        return new MyResponse(true, 'Record updated', request.recordId);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

### DELETE - Delete Records

```apex
@HttpDelete
global static MyResponse deleteRecord()
{
    try
    {
        String recordId = RestContext.request.params.get('id');
        validateRecordId(recordId);

        MyService.deleteRecord(recordId);
        return new MyResponse(true, 'Record deleted', recordId);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

### PATCH - Partial Update

```apex
@HttpPatch
global static MyResponse patchRecord(PatchRequest request)
{
    try
    {
        validatePatchRequest(request);
        MyService.patchRecord(request.recordId, request.fields);
        return new MyResponse(true, 'Record patched', request.recordId);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

## Versioning Strategy

- Include version in URL mapping: `/api/v1/resource/*`, `/api/v2/resource/*`
- Create new resource class for breaking changes: `AccountResourceV2`
- Maintain backward compatibility within same version
- Document deprecation timeline in ApexDoc
- Use separate folders for major versions: `restapis/v1/`, `restapis/v2/`

Example versioning:

```apex
// Version 1
@RestResource(urlMapping='/api/v1/accounts/*')
global with sharing class AccountResource
{
    // Original implementation
}

// Version 2 - breaking changes
@RestResource(urlMapping='/api/v2/accounts/*')
global with sharing class AccountResourceV2
{
    // New implementation with breaking changes
}
```

## Advanced Patterns

### Setting HTTP Status Codes

```apex
@HttpPost
global static MyResponse createRecord(CreateRequest request)
{
    try
    {
        validateCreateRequest(request);
        Id newRecordId = MyService.createRecord(request.name, request.type);

        // Set 201 Created status
        RestContext.response.statusCode = 201;
        RestContext.response.addHeader('Location', '/api/v1/myresource/' + newRecordId);

        return new MyResponse(true, 'Record created', newRecordId);
    }
    catch (Exception e)
    {
        RestContext.response.statusCode = 400; // Bad Request
        throw new MyResourceException(e.getMessage());
    }
}
```

### Accessing Request Headers

```apex
@HttpPost
global static MyResponse processWithAuth(MyRequest request)
{
    try
    {
        String authToken = RestContext.request.headers.get('Authorization');
        if (String.isBlank(authToken))
        {
            RestContext.response.statusCode = 401; // Unauthorized
            throw new MyResourceException('Authorization header required');
        }

        // Process request
        return new MyResponse(true, 'Success', null);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}
```

### URL Parameters

```apex
@HttpGet
global static MyResponse getRecords()
{
    try
    {
        // Access URL parameters from RestContext
        String status = RestContext.request.params.get('status');
        String recordType = RestContext.request.params.get('recordType');

        List<MyObject__c> records = MyService.getRecordsByFilters(status, recordType);
        return new MyResponse(true, 'Success', records);
    }
    catch (Exception e)
    {
        throw new MyResourceException(e.getMessage());
    }
}

// Called via: /api/v1/myresource?status=Active&recordType=TypeA
```

## Testing REST Resources

Mock Service layer dependencies using fflib_ApexMocks:

```apex
@IsTest
private class MyResourceTest
{
    // =======================================================================
    // SUCCESS TESTS
    // =======================================================================

    @IsTest
    static void testProcessRecordsSuccess()
    {
        // Given
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        req.requestURI = '/api/v1/myresource/';
        req.httpMethod = 'POST';
        RestContext.request = req;
        RestContext.response = res;

        MyResource.MyRequest request = new MyResource.MyRequest();
        request.recordIds = new List<Id>{fflib_IDGenerator.generate(MyObject__c.SObjectType)};

        // When
        Test.startTest();
        MyResource.MyResponse response = MyResource.processRecords(request);
        Test.stopTest();

        // Then
        Assert.isTrue(response.success, 'Response should be successful');
        Assert.areEqual(1, response.processedIds.size(), 'Should process 1 record');
    }

    // =======================================================================
    // VALIDATION TESTS
    // =======================================================================

    @IsTest
    static void testProcessRecordsNullRequest()
    {
        // Given
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        req.requestURI = '/api/v1/myresource/';
        req.httpMethod = 'POST';
        RestContext.request = req;
        RestContext.response = res;

        // When/Then
        try
        {
            Test.startTest();
            MyResource.processRecords(null);
            Test.stopTest();

            Assert.fail('Expected MyResourceException');
        }
        catch (MyResource.MyResourceException e)
        {
            Assert.isTrue(e.getMessage().contains('recordIds'), 'Error should mention recordIds');
        }
    }

    @IsTest
    static void testProcessRecordsEmptyRecordIds()
    {
        // Given
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        RestContext.request = req;
        RestContext.response = res;

        MyResource.MyRequest request = new MyResource.MyRequest();
        request.recordIds = new List<Id>();

        // When/Then
        try
        {
            Test.startTest();
            MyResource.processRecords(request);
            Test.stopTest();

            Assert.fail('Expected MyResourceException');
        }
        catch (MyResource.MyResourceException e)
        {
            Assert.isTrue(e.getMessage().contains('recordIds'), 'Error should mention recordIds');
        }
    }

    // =======================================================================
    // ERROR HANDLING TESTS
    // =======================================================================

    @IsTest
    static void testProcessRecordsDeletedRecords()
    {
        // Given
        RestRequest req = new RestRequest();
        RestResponse res = new RestResponse();
        RestContext.request = req;
        RestContext.response = res;

        MyResource.MyRequest request = new MyResource.MyRequest();
        request.recordIds = new List<Id>{fflib_IDGenerator.generate(MyObject__c.SObjectType)};

        // Mock service to throw ENTITY_IS_DELETED
        // (Use fflib_ApexMocks pattern from testing.md)

        // When
        Test.startTest();
        MyResource.MyResponse response = MyResource.processRecords(request);
        Test.stopTest();

        // Then
        Assert.isTrue(response.success, 'Should handle deleted records gracefully');
        Assert.isTrue(response.message.contains('deleted'), 'Message should mention deletion');
    }
}
```

## Best Practices

- **Always delegate to Service layer** - Keep resources thin
- **Use explicit sharing** - `global with sharing` for security
- **Validate all inputs** - Never trust external data
- **Return consistent responses** - Use standard response wrapper
- **Version your APIs** - Include version in URL
- **Document with ApexDoc** - Include request/response examples
- **Test all scenarios** - Success, validation, errors
- **Set appropriate HTTP status codes** - Use RestContext when needed
- **Handle exceptions gracefully** - Return meaningful errors
- **Never expose sensitive data** - Sanitize responses
