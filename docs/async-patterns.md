# Async Patterns

## Batch Jobs

### Key Patterns

- Use `Database.Stateful` when tracking counters across executions
- Always implement savepoint error handling in `execute()`
- Chain to queueables (not other batches) in `finish()` for fast follow-up work
- Use `!System.isBatch()` guard to prevent batch-from-batch execution
- Define `BATCH_SIZE` constant for consistent batch sizing

### Parent Transaction Tracking

All async operations capture and link to the parent transaction:

```apex
// In start() or constructor - capture parent transaction ID
this.parentLogTransactionId = Logger.getTransactionId();

// In execute() - link logs to parent transaction
Logger.setParentLogTransactionId(this.parentLogTransactionId);
Logger.info('Processing started');
Logger.saveLog(); // Always persist in async context
```

### Savepoint Error Handling

All batch/queueable execute methods follow this standard:

```apex
public void execute(Database.BatchableContext bc, List<SObject> scope)
{
    Logger.setParentLogTransactionId(this.originalTransactionId);

    Savepoint sp = Database.setSavepoint();
    try
    {
        // Business logic here
        MyService.processRecords(scope);
        numProcessed += scope.size(); // Update stateful counters
    }
    catch (Exception ex)
    {
        Database.rollback(sp);
        Logger.error('Error during processing', ex);
        // Re-throw if needed, or handle gracefully
    }
    finally
    {
        Logger.saveLog(); // Always persist logs
    }
}
```

### Batch Chaining Pattern

```apex
global void finish(Database.BatchableContext bc)
{
    Logger.setParentLogTransactionId(this.originalTransactionId);
    Logger.info('Batch complete, starting cleanup');
    Logger.saveLog();

    // Chain to queueable, not another batch
    System.enqueueJob(new MyCleanupQueueable(this.runId));
}
```

### Complete Batch Job Example

```apex
public without sharing class MyBatchJob
    implements Database.Batchable<SObject>, Database.Stateful
{
    public final static Integer BATCH_SIZE = 200;
    private String originalTransactionId;
    private SObjectType sObjectType;

    @TestVisible
    private Integer numProcessed = 0;

    public MyBatchJob(SObjectType sObjectType)
    {
        this.sObjectType = sObjectType;
    }

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
            numProcessed += scope.size();
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
        Logger.info(new LogMessage('Completed batch job. Processed {0} records', numProcessed));
        Logger.saveLog();
    }

    // Service facade methods for easy execution
    public static Id submit()
    {
        return Database.executeBatch(new MyBatchJob(MyObject__c.SObjectType), BATCH_SIZE);
    }

    public static String schedule(String cronExpression)
    {
        return System.schedule('MyBatchJob', cronExpression, new MyBatchSchedulable());
    }
}
```

## Queueable Jobs

### When to Use Queueables

- Prefer over batches for operations < 50k records (faster, no scheduling delay)
- Use for cleanup operations after batch jobs
- Self-chain with `!Test.isRunningTest()` guard for processing > 10k records

### Queueable Pattern

```apex
public without sharing class MyQueueable implements Queueable
{
    private String parentLogTransactionId;
    private List<Id> recordIds;

    public MyQueueable(List<Id> recordIds)
    {
        this.recordIds = recordIds;
        this.parentLogTransactionId = Logger.getTransactionId();
    }

    public void execute(QueueableContext context)
    {
        Logger.setParentLogTransactionId(this.parentLogTransactionId);

        Savepoint sp = Database.setSavepoint();
        try
        {
            MyService.processRecords(this.recordIds);
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

    // Service facade method
    public static Id enqueue(List<Id> recordIds)
    {
        return System.enqueueJob(new MyQueueable(recordIds));
    }
}
```

### Self-Chaining Queueable

For processing large datasets in chunks:

```apex
public void execute(QueueableContext context)
{
    Logger.setParentLogTransactionId(this.parentLogTransactionId);

    Savepoint sp = Database.setSavepoint();
    try
    {
        // Process current batch
        List<Id> currentBatch = getNextBatch();
        MyService.processRecords(currentBatch);

        // Chain to next batch if more work remains
        if (hasMoreWork() && !Test.isRunningTest())
        {
            System.enqueueJob(new MyQueueable(remainingIds));
        }
    }
    catch (Exception ex)
    {
        Database.rollback(sp);
        Logger.error('Error in queueable', ex);
    }
    finally
    {
        Logger.saveLog();
    }
}
```

## Schedulable Jobs

### Prevent Duplicate Execution

Check `isRunning()` before executing to prevent duplicate jobs:

```apex
public without sharing class MySchedulable implements Schedulable
{
    public void execute(SchedulableContext sc)
    {
        if (isRunning())
        {
            Logger.warn('Job already running, skipping execution');
            Logger.saveLog();
            return;
        }

        MyBatchJob.submit();
    }

    private Boolean isRunning()
    {
        List<AsyncApexJob> runningJobs = AsyncApexJobsSelector.newInstance()
            .selectRunningByClassName('MyBatchJob');

        return !runningJobs.isEmpty();
    }
}
```

### Scheduling Pattern

```apex
public static String schedule(String cronExpression)
{
    return System.schedule('MyScheduledJob', cronExpression, new MySchedulable());
}

// Example: Schedule to run daily at 2 AM
MySchedulable.schedule('0 0 2 * * ?');
```

## Best Practices

### Batch Jobs

- Use for processing > 50k records
- Always use `Database.Stateful` if tracking counts
- Set reasonable `BATCH_SIZE` (default 200, max 2000)
- Chain to queueables in `finish()`, never to another batch
- Always use savepoints in `execute()`

### Queueable Jobs

- Use for < 50k records (faster than batch)
- Can make callouts (unlike batch)
- Self-chain for large datasets with `!Test.isRunningTest()` guard
- Maximum 50 queueables can be enqueued per transaction
- Can chain only 1 queueable from another queueable

### Schedulable Jobs

- Always check for running instances before starting
- Use descriptive schedule names
- Test with `Test.startTest()` / `Test.stopTest()`
- Document cron expressions clearly

### All Async Operations

- Always capture parent transaction ID
- Always use savepoints in execute methods
- Always call `Logger.saveLog()` in `finally` blocks
- Use `without sharing` or no sharing declaration (system context)
- Delegate business logic to Service layer
- Use Service facade methods for submission

## Testing Async Jobs

```apex
@IsTest
static void testBatchJob()
{
    // Given
    insert new Account(Name = 'Test');

    // When
    Test.startTest();
    Id jobId = MyBatchJob.submit();
    Test.stopTest(); // Forces batch to complete

    // Then
    AsyncApexJob job = [SELECT Status, JobItemsProcessed FROM AsyncApexJob WHERE Id = :jobId];
    Assert.areEqual('Completed', job.Status, 'Batch should complete');
}
```
