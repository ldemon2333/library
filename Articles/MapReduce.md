Users specify a map function that processes a key/value pair to generate a set of intermediate k/v pairs, and a reduce function that merges all intermediate values associated with the same intermediate key.

# 2.1 Example 
Consider the problem of counting the number of occurrences of each word in a large collection of documents. The user would write code similar to the following pseudo-code:
![[Pasted image 20250318131423.png]]
![[Pasted image 20250318131603.png]]
# 3.1 Execution Overview
The _Map_ invocations are distributed across multiple machines by automatically partitioning the input data into a set of _M_ splits. The input splits can be processed in parallel by different machines. _Reduce_ invocations are distributed by partitioning the intermediate key space into _R_ pieces using a partitioning function (hash(key) mod R).

# 3.3 Fault Tolerance
## Worker Failure
Completed map tasks are re-executed on a failure because their output is stored on the local disk of failed machine and is therefore inaccessible. Completed reduce tasks do not need to be re-executed since their output is stored in a global file system.

When a map task is executed first by worker A and then later executed by worker B (because A failed), all workers executing reduce tasks are notified of the re-execution. Any reduce task that has not already read the data from worker A will read the data from worker B.

## Semantics in the Presence of Failures
We rely on atomic commits of map and reduce task outputs to achieve this property. Each in-progress task writes its output to private temporary files. A reduce task produces one such file, and a map task produces R such files (one per reduce task). When a map task completes, the worker sends a message to the master and includes the names of the R temporary files in the message. If the master receives a completion message for an already completed map task, it ignores the message. Otherwise, it records the names of R files in a master data structure.

When a reduce task completes, the reduce worker atomically renames its temporary output file to the final output file. If the same reduce task is executed on multiple machines, multiple rename calls will be executed for the same final output file. We rely on the atomic rename operation provided by the underlying file system to guatantee that the final file system state contains just the data produced by one execution of the reduce task.

# 3.6 Backup Tasks
One of the common causes that lengthens the total time taken for a MapReduce operation is a "straggler"; a machine that takes an unusually long time to complete one of the last few map or reduce tasks in the computation. Stragglers can arise for a whole host of reasons. For example, a machine with a bad disk may experience frequent correctable errors that slow its read performance from 30 MB/s to 1 MB/s.

We have a general mechanism to allevitae the problem of stragglers. When a MapReduce operation is close to completion, the master schedules backup executions of the remaining in-progress tasks. The task is marked as completed whenever either the primary or the backup execution completes.