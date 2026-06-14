# Assignment 1: Serverless Computing - Critical Analysis

## Student Information
- **Name:** Akash Patel
- **Student ID:** 041269598
- **Course:** CST8917 - Spring 2026

## Assignment Tasks

### Part 1: Paper Summary

Though serverless computing can be "one step forward" by being a fully managed autoscaling and execution technology, with no need for server provisioning from the user end, it can also be "two steps back" as a technology that is not suitable for workloads that demand efficiency with data, hardware acceleration, or distributed systems. To back this up, the paper highlights two ways in which the cloud is strong in high compute potential, which are both to enable ease of multi-tenancy and ease of scaling legacy data services, rather than cutting edge programming models. Comparatively, paper is about the early FaaS options, AWS Lambda, and the objective facts of how "serverless" solutions like these were worked into in 2019.

The execution time constraints detected are its extremely short lifetime (timeout) and the more complicated requirement to complete the execution of long running tasks. According to the paper, Lambda functions have a max duration of 15 minutes. Moreover, they will not necessarily be executed in the same VM in subsequent invocations and thus must be written "assuming that state will not be recoverable across invocations". In addition, some tasks which can run for an extended period of time, such as  model training, need to be divided into multiple invocations, which eventually leads to complexity and latency with orchestration and state management. In the case study, for instance, the cost of training the ML model on Lambda is 7.3x as high as that on EC2, and the training takes 21x longer.

Some of communication/network drawbacks are I/O bottlenecks, no direct network addressability, which leads to increased latency. This translates to an average of 538Mbps network bandwidth per single Lambda function, which is still relatively low compared to a modern SSD, up to 2.5x slower. In addition, since Lambda functions are not network addressable, they cannot receive direct messages. Communication must be directed through a communication with a communication intermediary such as S3. Therefore, for the same number of messages the direct EC2-EC2 messaging takes ~290 µs while the Lambda-S3-EC2 messaging takes ~106-108 ms or 365x-372x slower.

The "data shipping" anti-pattern is a FaaS "shipping data to code" rather than "shipping code to data" approach. This is because FaaS executes code on separate VMs, unshared from data, and has minimal state persistence across invocations. This means that with the data-intensive workloads such as the above mentioned ML training, there is high latency and bandwidth cost involved. Data is unnecessarily extracted from S3 multiple times rather than doing computational work.

A second issue that was found is hardware constraints. Only CPU slices and RAM are exposed by FaaS, not GPU or any other hardware. Furthermore, the largest Lambda instances still only has 3 GB RAM, which is inadequate for larger in-memory models. This also implies that FaaS is not suitable for any type of deep ML training, or similar hardware-accelerated use cases.

As mentioned, FaaS solutions do not work well with a distributed computing system or stateful workload. However, with standard Lambda functions, the only way to retain in-memory state between invocations is to read and write to storage such as S3 on every call. This has a negative affect on distributed communication as well as event-driven designs must cope with the inconsistency and latency of repeated read/writes to "slow storage.

Authors in the paper propose that future cloud programming must address the following issues:

1. **Support stateful/long-lived computation**: Functions must be enabled to run beyond the timeout limited when needed and be able to be addressed to directly over network. In addition, there must be a way to maintain state across multiple invocations without the need of repeated R/W to slow storage.

2. **Negate the anti-pattern by "shipping code to data" with low-latency access to storage**: The goal proposed here is to allow data-intensive workloads like ML training be competitive with similar workloads on traditional server-hosted services. FaaS solutions must be able to provide higher bandwidth as functions scale out, balancing compute with data retrieval needs.

3. **Enable hardware specialization under serverless models**: In order to make serverless solutions viable for ML training and other heavy workloads, CSPs must enable access to hardware features like GPU and appealing pricing models for autoscaling.

---

### Part 2: Azure Durable Functions Deep Dive

**1. Orchestration model: How do orchestrator, activity, and client functions work together? How does this differ from basic FaaS?**

Orchestrator functions define the workflow logic for managing various short-lived functions for a long-lived process via code. (support for Python, JS, C#, etc.) [1]

Activity functions are the basic units of work, being standard Azure Functions that perform tasks like database writes or API calls.[2] They return results to the orchestrator that then directs to the next step in the workflow.

Client functions as the entry point with typical function triggers of external events (ie. HTTP trigger) that activates downstream orchestration by the Durable Function. [3]

Compared to basic FaaS, Durable Function addresses the critique that FaaS lacks a way to coordinate compute due to its stateless nature. Durable Functions provides a stateful way to allow functions to execute together in a chain or parallel, rather than in isolation. Functions can now progress state  rather than R/W to storage for every result.


**2. Execution timeouts: How do orchestrators bypass the timeout limits that apply to regular Azure Functions? What limits still apply to activity functions?**

Orchestrators bypass typical FaaS timeout limits (15 min cap for Lambda & 10 min cap for Functions) using the aforementioned replay mechanism. Though the workflow may look like it is running for several days, the actual execution compute is made up of short bursts like a typical function. When the orchestrator is awaiting an async task, it checkpoints its state to storage and unloads from memory until it resumes using a Durable Timer. When the timer resets, the function replays from the start to rebuild state, thus virtualizing a seemingly long-running process. [1]

Activity functions do not have the replay function and are subject to typical timeout limits, like 10 min max for Functions. [6] As a result, any long-running tasks should be broken down into smaller activities or be hosted on higher cost plans that can ensure their completion. For example, anything above Consumption plan has an unbounded timeout limit at maximum [6]

**3. State management: How does Durable Functions manage state (event sourcing, checkpointing, replay)? How does this address the paper's criticism that functions are stateless?**

The event sourcing pattern is when instead of saving the entire memory dump, the orchestration persists an append-only execution history of events to the storage. [4] When the orchestrator hits an `await` statement, it checkpoints progress by committing new events since the last checkpoint to storage and unloads from memory. [5] When resuming, the orchestrator replays from the start all the previously executed tasks and returns stored state to rebuild local state without re-executing everything. [1]

This solves the paper's critique about the stateless problems of serverless solutions, which relied on expensive and slow R/Ws from storage like S3. By "replaying" state restoration from checkpointed events following the event sourcing pattern, Durable Functions attribute statefulness to the orchestration and thus allows workflows to virtually persist data over a long-running process.

**4. Communication between functions: How do orchestrator and activity functions communicate? Does this address the paper's criticism about functions needing slow storage intermediaries?**

Orchestrator & activity functions communicate via a Task Hub that uses queues for message passing and tables for storing execution history. [7] Since this is managed within Azure Storage, this does validate the paper's criticism that serverless solutions must use a slow storage intermediary for network communications, resulting in unappealing latency issues.

To address this, the fully-managed Durable Task Scheduler (DTS) (the successor to Netherite) aims to provide much higher throughput compared to the default storage polling option. It features direct gRPC connections between the workers and scheduler. [8] Example benchmarks show that DTS is about 5x faster than default Azure Storage. [8] As a result, Durable Functions now have modern solutions in the works to mitigate that slow storage critique.

**5. Parallel execution: (fan-out/fan-in) How does the fan-out/fan-in pattern work? How does it address the paper's concern about distributed computing?**

The fan-out/fan-in pattern allows orchestrators to execute multiple activity functions in parallel and only resume past the parallel operations once all have completed. This is done by scheduling the parallel batch of activity tasks (fan-out) without awaiting yet. Then, some sort of sync function/method is used to wait for results (fan-in) like `yield context.task_all(parallel_tasks)` in Python. [5]

This pattern directly addresses the concern the paper expresses that serverless solutions are incompatible with distributed computing. Rather than forcing functions to coordinate via shared storage and coordinating "fan-in" through complex series of queue triggers and external state management [9], Durable Functions lets developers treat distributed parallel operations as simple code. [5] Therefore, the gap between distributed requirements and stateless serverless computing is bridged.

---

### Part 3: Critical Evaluation

After reading the paper on first-gen FaaS and being impressed by how Azure Durable Functions (ADF) have managed to overcome some of the key concerns raised in the paper, I believe that ADF is a positive way forward from the stateless and orchestration issues of the early FaaS. But, there are two basic problems that ADF will have to address: the architecture of serverless platforms. Not all of what's described in the “data-shipping” anti-pattern or non-specialization of hardware is completely solved by ADF, and some of the solutions described by ADF appear to lie on top of the “flawed” architecture.

Recalling, the paper is critical of FaaS for being inefficient with data, because "shipping data to code" is "shipping code to data"—so moving compute away from storage introduces unacceptably high I/O bottlenecks/latency for data-intensive workloads such as ML training. Activity functions are executed as per workflow states, but on a compute separate from Azure Storage where the data is stored, although the workflow state can be kept virtually for as long as ADF needs. For this reason, in ML training, the activity function would still need to read datasets, process them, write them back again and, therefore, be subject to the same bandwidth/latency concerns that are discussed in the paper. DTS can tweak how the execution history is stored (gRPC & logs), but it doesn't solve the problem of the processing of app data.

One of the major criticism points is that FaaS only enables slicing of CPU and RAM, without access to a dedicated hardware such as GPU. Typically, standard activity functions with very limited hardware capabilities are used and orchestrated with ADF. At this time, there isn't a built-in mechanism in ADF to enable hardware acceleration. Tasks that can be hardware accelerated, such as ML training, then, have to be done partly in non-serverless, or even outside the main Azure Function suite. So, up to this day, the cutting-edge accelerated software innovation envisioned in the paper is not supported by a FaaS option.

However, ADF still showcases a “matured” version of FaaS with enhancements to some of the criticism. For instance, it is mainly a solution to the distributed systems problem. The paper argues that FaaS is not compatible with the distributed computing model because in order to communicate between functions, storage has to be used as an intermediary. Simple patterns that can be implemented as distributed patterns, such as fan-out/fan-in, can be written in ADF's Durable Task Framework. In addition, DTS works without storage polling with gRPC or event streams, which resolves many of the criticisms about slow storage. 

However, it is worth noting that ADF is not an attempt at redefining the inherently problematic compute-storage dichotomy, but rather something built on top of FaaS as an orchestration layer. It has certainly made serverless much more usable for long duration processes, but I don't feel like it has truly unleashed the future of data intensive computing.

---

### References

[1]cgillum, “Durable Orchestrations - Azure Functions,” Microsoft.com, Dec. 11, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp-inproc

[2]lily-ma, “Azure Durable Functions: FaaS for Stateful Logic and Complex Workflows,” TECHCOMMUNITY.MICROSOFT.COM, Sep. 06, 2024. https://techcommunity.microsoft.com/blog/appsonazureblog/azure-durable-functions-faas-for-stateful-logic-and-complex-workflows/4238858

[3]R. Dennyson, “The Ultimate Guide to Azure Durable Functions: A Deep Dive into Long-Running Processes, Best Practices, and Comparisons with Azure Batch,” Medium, Sep. 21, 2024. https://medium.com/@robertdennyson/the-ultimate-guide-to-azure-durable-functions-a-deep-dive-into-long-running-processes-best-bacc53fcc6ba

[4]Microsoft, “Event Sourcing pattern - Azure Architecture Center,” learn.microsoft.com. https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

[5]cgillum, “Durable Functions Overview - Azure,” Microsoft.com, Apr. 06, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=in-process%2Cnodejs-v3%2Cv1-model&pivots=python.

[6]ggailey777, “Azure Functions scale and hosting,” learn.microsoft.com. https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale

[7]cgillum, “Durable Functions storage providers - Azure,” Microsoft.com, May 08, 2025. https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-storage-providers.

[8]greenie-msft, “Announcing the public preview launch of Azure Functions durable task scheduler,” TECHCOMMUNITY.MICROSOFT.COM, Mar. 20, 2025. https://techcommunity.microsoft.com/blog/appsonazureblog/announcing-the-public-preview-launch-of-azure-functions-durable-task-scheduler/4389670

[9]J. Hellerstein et al., “Serverless Computing: One Step Forward, Two Steps Back,” Jan. 2019.

## AI Disclosure Statement

AI-assisted research tools were used to generate an initial list of 15–20 sources related to Durable Functions. After reviewing and assessing the relevance of each source, the eight most pertinent references were selected and manually cited in IEEE format.


