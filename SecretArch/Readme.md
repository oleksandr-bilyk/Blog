KeyVault or other secret vault capabilities may be extended by secret managing service that will provide additional lifetime features.
This document contains concept of Managed Secret that is kind of service that extends basic abstract 
## Secret Vault features:

1. Potential count of active secrets 100.000+ daily or more.
1. Secrets may be reset to new value.
1. Secrets may expire after some period.
1. All operations should be transactional and provide eventual consistency of all aggregates. 
1. SLA near to CosmosDb which is P99.
1. No dependency on Azure only services (e.g. Service Buss) to let run in out of Azure.
1. Underlying Secret Vault is abstract storage with limited API just to setSecret, getSecret, removeSecret. Other alternatives to KeyVault may be used.

## Concepts

- Allocation is double phase commit process of insert secret into DB and SecretVault.
- Arch is managed secret entity that is guarantee consistency of set, get, renew and remove operations.

::: mermaid
sequenceDiagram
    participant Client
    participant Allocation
    participant Arch
    participant Vault

    Client ->> Allocation: SetSecret
    Allocation ->> Vault : SetSecret
    Allocation ->> Arch: SetSecret
    Client ->> Arch: GetSecret
    Arch ->> Vault : GetSecret
    Vault -->> Arch: Secret
    Arch -->> Client: Secret
    Note right of Arch: When secret expired
    Arch ->> Vault: RemoveSecret 
:::

## Cron
Chron provides deferred operations. Analog or Azure ServiceBuss deferred message but doesn't not depend on Azure. It can be implemented as No-SQL change feed.
Chron is internal mechanism that is not visible for client but is used by Allocation, Arch and deallocation services.

- Consumer - procedure that requests some task to be scheduled
- Schedule_DB - No-SQL collection with change feed.
- ScheduleObserver - Schedule_DB change feed listening procedure.
- Scanner - Infinite loop procedure, preferably single instance with [leader election](https://docs.microsoft.com/en-us/azure/architecture/patterns/leader-election) that query Schedule_DB to filter active records.

::: mermaid
sequenceDiagram

    participant Consumer
    participant Schedule_DB
    participant ScheduleObserver
    participant Scanner
  
    Consumer ->> Schedule_DB : Insert row with State=Delayed
    Schedule_DB ->> ScheduleObserver : delated task inserted.
    ScheduleObserver ->> ScheduleObserver : ignore delayed row.

    Note left of Scanner: Scanner interval
    Scanner ->> Schedule_DB : Get top N pending records (where State=Delayed AND DelayedUntil <= NowUtc)
    Scanner ->> Schedule_DB : Insert all pending tasks with State=Pending
    
    Schedule_DB -->> ScheduleObserver: Task with State=Pending inserted
    ScheduleObserver -->> Consumer: The task with State=Pending
:::

# Allocation transaction

Secret Allocation is analog of KeyVault set but just should have transactional SecretVault and No-SQL registry update.

## Samples of non-transactional operations between SecretVault and No-SQL Registry.

#### Case 1: SecretVault first and Database second.
Let's consider case when we insert to SecretVault first and to database Registry second. If first SecretVault was successful and second operation failed then secret may be abandoned in Secret Vault. Second operation may fail by many reasons as process failure or, which is more probable, No-SQL database connectivity issue.

### Case 2: Database first and SecretVault second.
Let's consider case when we insert to Database first and to SecretVault second. If first operation was successful and second operation failed then database record will be inconsistent. Second operation may fail by many reasons as process failure or, which is more probable, SecretVault connectivity issue.

### Case 3: Two phase commit.
We may set a secret in tree phases.
1. Insert Database secret set initialization. That operation should schedule transaction consistency audit operation. Consistency Audit operation may be executed in few minutes after initialization phase.
2. Set Secret Vault. 
3. Insert Database secret commit.
Consistency audit should remove SecretVault item if it was not committed in database.

## Legacy secrets pruning.
To remove all legacy secrets it is possible to register then for removing using tha same engine that is used in two phase commit. Any secret that had only first phase DB record and don't have second phase record will be automatically removed after few minutes. See Allocation transaction rollback in this document.

## Allocation two phase transaction Committed - AllocationRequested, keyVaultSet and AllocationCompleted completed.

::: mermaid
sequenceDiagram
    participant Client
    participant AllocationTransaction
    participant AllocationEvent_DB
    participant AllocationEventObserver
    participant Cron
    participant Vault
    participant Arch
  
    Client ->> AllocationTransaction: Set secret
    Note right of AllocationTransaction: Begin transaction
    AllocationTransaction ->> AllocationEvent_DB: Insert AllocationRequested
    AllocationEvent_DB -->> AllocationEventObserver: AllocationRequested 
    AllocationEventObserver ->> Cron : Audit in 5 min
    AllocationTransaction ->> Vault: SetSecret
    AllocationTransaction ->> AllocationEvent_DB: Insert AllocationCompleted
    Note right of AllocationTransaction: End transaction
    AllocationEvent_DB -->> AllocationEventObserver: AllocationCompleted 
    AllocationEventObserver ->> Arch: SetSecret

    Note left of Cron: 5 min later
    Cron -->> AllocationEvent_DB: Insert Audit
    AllocationEvent_DB ->> AllocationEventObserver : Audit
    AllocationEventObserver ->>  AllocationEvent_DB : Get by secret id
    AllocationEventObserver ->> AllocationEventObserver: Audit validated
:::

## Allocation transaction rollback - AllocationRequested but no progress anymore.

::: mermaid
sequenceDiagram
    participant Client
    participant AllocationTransaction
    participant AllocationEvent_DB
    participant AllocationEventObserver
    participant Cron
    participant Vault
  
    Client ->> AllocationTransaction: Set secret
    Note right of AllocationTransaction: Begin transaction
    AllocationTransaction ->> AllocationEvent_DB: Insert AllocationRequested
    AllocationEvent_DB -->> AllocationEventObserver: AllocationRequested 
    AllocationEventObserver ->> Cron : Audit in 5 min

    Note left of Cron: 5 min later
    Cron -->> AllocationEvent_DB: Insert Audit
    AllocationEvent_DB ->> AllocationEventObserver : Audit
    AllocationEventObserver ->>  AllocationEvent_DB : Get by secret id
    AllocationEventObserver ->> AllocationEventObserver : Abandoned allocation detected
    AllocationEventObserver ->>  AllocationEvent_DB : Insert AllocationCancelled
    AllocationEvent_DB -->> AllocationEventObserver: AllocationCancelled
    AllocationEventObserver ->> Vault: Remove
    Vault -->> AllocationEventObserver: None
:::

## Allocation transaction rollback - AllocationRequested, keyVaultSet but no completion.

::: mermaid
sequenceDiagram
    participant Client
    participant AllocationTransaction
    participant AllocationEvent_DB
    participant AllocationEventObserver
    participant Cron
    participant Vault
  
    Client ->> AllocationTransaction: Set secret
    Note right of AllocationTransaction: Begin transaction
    AllocationTransaction ->> AllocationEvent_DB: Insert AllocationRequested
    AllocationEvent_DB -->> AllocationEventObserver: AllocationRequested 
    AllocationEventObserver ->> Cron : Audit in 5 min
    AllocationTransaction ->> Vault: SetSecret

    Note left of Cron: 5 min later
    Cron ->> AllocationEvent_DB: Insert Audit
    AllocationEvent_DB -->> AllocationEventObserver : Audit
    AllocationEventObserver ->>  AllocationEvent_DB : Get by secret id
    AllocationEventObserver ->> AllocationEventObserver : Abandoned allocation detected
    AllocationEventObserver ->>  AllocationEvent_DB : Insert AllocationCancelled
    AllocationEvent_DB -->> AllocationEventObserver: AllocationCancelled
    AllocationEventObserver ->> Vault: Remove
    Vault -->> AllocationEventObserver: Removed
:::

## Arch
Arch is managed secret entity that is guarantee consistency of set, get, renew and remove operations. Note that Arch service doesn't update Vault itself. If command comes from Allocation to Arch that means that secret is already stored in Vault. When arch decides that secret can be deallocated it calls Deallocation service to defer secret removal. It will provide secret read consistency because if secret in arch is marked as actual then Vault secret will be available for read from Vault at least for few minutes.

::: mermaid
sequenceDiagram
    participant Allocation
    participant ArchCommand_DB
    participant ArchCommandObserver
    participant ArchEvent_DB
    participant ArchEventObserver
    participant Cron
    participant Vault
  
    Allocation ->> ArchCommand_DB: Insert SetSecret 1
    ArchCommand_DB -->> ArchCommandObserver: SetSecret 1
    ArchCommandObserver ->> ArchEvent_DB: Insert SecretSet 1
    ArchEvent_DB -->> ArchEventObserver: SecretSet 1
    ArchEventObserver ->> Cron: Schedule DisposeSecret 1 in 7 days

    Allocation ->> ArchCommand_DB: Insert SetSecret 2
    ArchCommand_DB -->> ArchCommandObserver: SetSecret 2
    ArchCommandObserver ->> ArchEvent_DB: Insert SecretSet 2
    ArchCommandObserver ->> ArchEvent_DB: Insert SecretDisposed 1
    ArchEvent_DB -->> ArchEventObserver: SecretSet 2
    ArchEventObserver ->> Cron: Schedule DisposeSecret 2 in 7 days
    ArchEvent_DB -->> ArchEventObserver: SecretDisposed 1
    ArchEventObserver ->> Cron: Schedule RemoveSecret 1 in 5 min
    Note left of Cron: Deferred RemoveSecret interval
    Cron ->> Vault: RemoveSecret 1

    Note left of Cron: After Secret 1 expiration
    Cron ->> ArchCommand_DB: Insert DisposeSecret 1
    ArchCommand_DB -->> ArchCommandObserver: DisposeSecret 1
    ArchCommandObserver ->> ArchCommandObserver: Secret 1 was already disposed

    Note left of Cron: After Secret 2 expiration
    Cron ->> ArchCommand_DB: Insert DisposeSecret 2
    ArchCommand_DB -->> ArchCommandObserver: DisposeSecret 2
    ArchCommandObserver ->> ArchEvent_DB: Insert SecretDisposed 2
    ArchEvent_DB -->> ArchEventObserver: SecretDisposed 2
    ArchEventObserver ->> Cron: Schedule RemoveSecret 2 in 5 min
    Note left of Cron: Deferred RemoveSecret interval
    Cron ->> Vault: RemoveSecret 1

:::
