# Logging with Nebula Logger

## Overview

All operations use **Nebula Logger** for consistent, production-grade logging with transaction tracking.

## Basic Logging Pattern

```apex
public void processRecords(List<SObject> records)
{
    Logger.info('Starting to process records');

    try
    {
        // Business logic
        Logger.info(new LogMessage('Processed {0} records', records.size()));
    }
    catch (Exception ex)
    {
        Logger.error('Error processing records', ex);
        throw ex;
    }
    finally
    {
        Logger.saveLog(); // Always persist logs
    }
}
```

## Key Practices

### String Interpolation

Use `LogMessage` for string interpolation:

```apex
Logger.info(new LogMessage('Processed {0} records', recordCount));
Logger.warn(new LogMessage('Found {0} errors in {1} records', errorCount, totalCount));
```

### Exception Logging

Use `Logger.error('Message', exception)` to capture full stack traces:

```apex
catch (Exception ex)
{
    Logger.error('Error processing records', ex);
    throw ex;
}
```

### Always Save Logs

Call `Logger.saveLog()` in `finally` blocks to ensure logs persist even on error:

```apex
try
{
    // Business logic
}
catch (Exception ex)
{
    Logger.error('Error occurred', ex);
    throw ex;
}
finally
{
    Logger.saveLog(); // Critical - persists logs
}
```

### Service Method Pattern

Always call `Logger.saveLog()` at the end of service methods:

```apex
public void processRecords(Set<Id> recordIds)
{
    Logger.info('Starting record processing');

    try
    {
        // Business logic
    }
    catch (Exception ex)
    {
        Logger.error('Error in processRecords', ex);
        throw ex;
    }
    finally
    {
        Logger.saveLog();
    }
}
```

## Async Operation Logging

### Parent Transaction Tracking

All async operations (Batch, Queueable, Future) use **parent transaction tracking** to link logs across async boundaries:

```apex
// In start() or constructor - capture parent transaction ID
this.parentLogTransactionId = Logger.getTransactionId();

// In execute() - link logs to parent transaction
Logger.setParentLogTransactionId(this.parentLogTransactionId);
Logger.info('Processing started');
Logger.saveLog(); // Always persist in async context
```

### Batch Job Logging Pattern

```apex
public class MyBatchJob implements Database.Batchable<SObject>, Database.Stateful
{
    private String originalTransactionId;

    public Database.QueryLocator start(Database.BatchableContext bc)
    {
        this.originalTransactionId = Logger.getTransactionId();
        Logger.info('Started batch job');
        Logger.saveLog();

        return MySelector.newInstance().queryLocatorForProcessing();
    }

    public void execute(Database.BatchableContext bc, List<SObject> scope)
    {
        Logger.setParentLogTransactionId(this.originalTransactionId);

        Savepoint sp = Database.setSavepoint();
        try
        {
            MyService.processRecords(scope);
        }
        catch (Exception ex)
        {
            Database.rollback(sp);
            Logger.error('Error processing records', ex);
        }
        finally
        {
            Logger.saveLog();
        }
    }

    public void finish(Database.BatchableContext bc)
    {
        Logger.setParentLogTransactionId(this.originalTransactionId);
        Logger.info('Completed batch job');
        Logger.saveLog();
    }
}
```

### Queueable Job Logging Pattern

```apex
public class MyQueueable implements Queueable
{
    private String parentLogTransactionId;

    public MyQueueable()
    {
        this.parentLogTransactionId = Logger.getTransactionId();
    }

    public void execute(QueueableContext context)
    {
        Logger.setParentLogTransactionId(this.parentLogTransactionId);

        Savepoint sp = Database.setSavepoint();
        try
        {
            // Business logic
            Logger.info('Queueable processing complete');
        }
        catch (Exception ex)
        {
            Database.rollback(sp);
            Logger.error('Error in queueable', ex);
            throw ex;
        }
        finally
        {
            Logger.saveLog();
        }
    }
}
```

## Log Levels

Use appropriate log levels:

- `Logger.error()` - Errors and exceptions
- `Logger.warn()` - Warnings and unexpected conditions
- `Logger.info()` - General information and milestones
- `Logger.debug()` - Detailed debugging information
- `Logger.fine()` / `Logger.finer()` / `Logger.finest()` - Very detailed tracing

## Testing with Logger

### Mocking Logger in Tests

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

## Best Practices

- **Always call `Logger.saveLog()`** - Logs aren't persisted automatically
- **Use `finally` blocks** - Ensures logs are saved even on exception
- **Track parent transactions** - Links async operations to their origin
- **Use `LogMessage`** - Cleaner than string concatenation
- **Log exceptions with context** - Include the exception object for stack traces
- **Don't log sensitive data** - PII, credentials, etc.
