# AEM Workflows

## Engine Architecture & Lifecycle

The AEM Workflow engine is built on top of the **Granite Workflow** framework and operates as a state machine. Each transition/step is managed by the **Workflow Engine Service**.

### Granite Workflow vs. Sling Jobs

While workflows appear as a continuous process in the UI, they are technically executed as **Sling Jobs**.

- **Job Offloading:** In clustered environments, Sling Job distribution ensures that workflow steps can be offloaded to different instances/nodes.
- **Consistency:** Every step completion triggers a JCR write to persist the state, ensuring that if an instance crashes, the workflow can resume from the last persisted "checkpoint."

Under the hood, workflow steps are executed as **Sling Jobs**:
- In clustered environments, Sling Job distribution can offload steps to different nodes.
- Every step completion triggers a JCR write that persists state. If an instance crashes, the workflow resumes from the last persisted checkpoint.

### The Workflow Node Structure

| Data | Path |
|---|---|
| Workflow model definition | `/conf` or `/etc/workflow/models` — defines the graph of nodes (steps) and transitions |
| Runtime instance data | `/var/workflow/instances` — contains runtime state, history of each step, start/end times, and the identity of the acting user |

## State Management & Metadata Persistence

Understanding the different metadata maps available during a process step execution is fundamental to managing state and creating reusable code.

### Three Layers of Metadata Storage (Persistence Lifecycle View)

| Layer | Scope | Persistence Lifecycle |
|---|---|---|
| **WorkItem Metadata** (`item.getMetaDataMap()`) | Step-Specific | Exists only for the duration of a single step. Once the step completes, this map is typically discarded. |
| **WorkflowData Metadata** (`item.getWorkflowData().getMetaDataMap()`) | Instance-Wide | Persists across the entire workflow lifecycle. This is the primary location for sharing variables between disparate process steps. |
| **Workflow Variables** | Instance-Wide | A formalization of metadata introduced in later AEM 6.5 service packs, allowing for typed data (JSON, XML, String) to be passed through the graph. |

### The Three Maps — Practical Usage View

| Map | Source | Usage |
|---|---|---|
| **Step Metadata (`args`)** | The "Arguments" in the Workflow Model Step dialog | Reading static configurations for the code |
| **WorkItem Metadata (`item.getMetaDataMap()`)** | Specific to the current execution of this step | Very short-lived data used within the step's logic |
| **WorkflowData Metadata (`item.getWorkflowData().getMetaDataMap()`)** | The shared memory for the entire workflow instance | **Crucial** — use this to pass data from Step A to Step B (e.g. a "route" flag or an external system ID) |

## WorkItem vs. MetaDataMap (args)

### Architectural Roles

**WorkItem — The Runtime Container**

The `WorkItem` is the object that represents the current instance of a workflow as it passes through a specific step. It acts as the "handle" for the engine's execution state.
- **Identity:** Contains the ID of the current step and the overall workflow instance.
- **Data Access:** Provides the primary gateway to the **WorkflowData**, which contains the payload (the asset or page path) — via `item.getWorkflowData().getPayload()`.
- **Persistence:** Used to access the long-term memory of the workflow that persists across different steps.

**MetaDataMap (args) — The Design-Time Configuration**

The `MetaDataMap` passed as the third parameter in the `execute` method represents the **Process Arguments** — values configured by the developer or author within the Workflow Model editor.
- **Function:** Allows a single Java class to be reused across different workflow models by passing unique parameters.
- **Scope:** Local to the current step configuration.
- **Source:** Values are sourced from the "Process Arguments" text field or the metadata dialog in the Workflow Step UI.

### Technical Comparison

| Feature | `WorkItem` | `MetaDataMap` (args) |
|---|---|---|
| **Object Type** | `com.adobe.granite.workflow.exec.WorkItem` | `com.adobe.granite.workflow.metadata.MetaDataMap` |
| **Purpose** | "What is happening?" — runtime execution context | "How should it behave?" — design-time configuration |
| **Payload Access** | **Yes** via `item.getWorkflowData().getPayload()` | **No** |
| **Data Lifetime** | Exists for the duration of the workflow instance | Immutable configuration defined in the model |
| **Primary Use Case** | Retrieving the path of the asset/page being processed | Retrieving an API key, folder path, or flag set in the model UI |

### Practical Java Application

```java
public void execute(WorkItem item, WorkflowSession session, MetaDataMap args)
        throws WorkflowException {

    // 1. Using WorkItem to get the content path (The "What")
    String payloadPath = item.getWorkflowData().getPayload().toString();

    // 2. Using MetaDataMap 'args' to get configurations (The "How")
    // Values retrieved from the 'Process Arguments' field in the UI
    String folderName = args.get("PROCESS_ARGS", "default-folder");

    // 3. Using WorkflowData MetaData to pass info to the NEXT step (The "Shared Memory")
    item.getWorkflowData().getMetaDataMap().put("processingComplete", true);
}
```

### Summary Analogy

- The **WorkItem** is the **Passenger**: It knows where it is going (the payload) and carries a suitcase (WorkflowData Metadata) that it takes from house to house (step to step).
- The **MetaDataMap (args)** is the **House Manual**: Each house (step) has its own manual telling the passenger how to behave while they are inside that specific house.

## Deployment Paradigms: 6.5 vs. Cloud Service

The transition to AEM as a Cloud Service has fundamentally changed how workflows interact with assets.

### The Asset Microservices Shift

In AEM 6.5, the **DAM Update Asset** workflow was the "heavy lifter," performing binary processing locally. In AEMaaCS, this is replaced by **Asset Microservices**.

- **Post-Processing Workflows:** Custom workflows are now configured to run only after the external microservices have completed rendition generation and metadata extraction.
- **Binary Awareness:** Modern workflows should avoid direct binary manipulation within the JVM, instead relying on metadata triggers or external API orchestrations.

| | AEM 6.5 | AEM as a Cloud Service |
|---|---|---|
| Asset processing | **DAM Update Asset** workflow — all binary processing in-JVM | **Asset Microservices** — external cloud services handle rendition generation |
| Custom workflows | Can manipulate binaries directly | Should run as post-processing workflows after microservices complete |
| Binary handling | Direct JVM manipulation | Avoid — use metadata triggers and external API calls instead |

## Execution Patterns

### Transient Workflows

Designed for high-performance automation where an audit trail is not required.

- **Mechanism:** They do not create nodes under `/var/workflow/instances`. The entire process is managed in memory and persisted only upon the final JCR commit of the payload.
- **Benefit:** Dramatically reduces JCR contention and prevents Oak index bloat during high-volume ingestion (e.g. bulk asset imports).

### Participant Steps and the Inbox

For human-in-the-loop processes, the engine utilizes **Participant Steps**. These place a task in the **AEM Inbox**.

- **Dynamic Participant Choosers:** These utilize custom logic to evaluate the payload and assign the task to a specific user or group at runtime.

```java
@Component(
    service  = ParticipantStepChooser.class,
    property = { "chooser.label=Locale-Based Translator Chooser" }
)
public class LocaleParticipantChooser implements ParticipantStepChooser {
    public String getParticipant(WorkItem item, WorkflowSession session,
                                 MetaDataMap args) throws WorkflowException {
        String path = item.getWorkflowData().getPayload().toString();
        if (path.contains("/fr/")) return "fr-translators";
        if (path.contains("/de/")) return "de-translators";
        return "global-reviewers";
    }
}
```

## Automation & Launchers

Launchers are the event listeners that bridge the JCR and the Workflow Engine — they start a workflow automatically when content changes.

- **Event Types:** Triggered by `NODE_CREATED`, `NODE_MODIFIED`, or `NODE_REMOVED`.
- **Globbing & Filtering:** Advanced launchers use complex path filtering and property-level conditions (e.g. `jcr:content/metadata/dc:format == image/jpeg`) to ensure precise execution.
- **Exclusion List:** Launchers should be configured with exclusion lists to prevent infinite loops (e.g. a workflow that modifies a property should not re-trigger itself).

## Advanced Routing Logic

Modern workflow models utilize non-linear paths for complex business requirements.

| Feature | Description |
|---|---|
| **OR Splits** | Directs the flow into one of multiple paths based on a routing expression or script. Set a metadata variable in a process step; the OR Split evaluates it. |
| **AND Splits** | Parallelizes execution. The workflow will only proceed once all branches have reached a "join" point. |
| **Goto Steps** | Allows the engine to jump backward or forward in the model graph based on metadata values, effectively enabling "Retry" loops or skipping redundant approvals — without duplicating model steps. |

**Retry loop pattern using Goto Step:**

```java
// In the process step:
if (retryCount < maxRetries) {
    item.getWorkflowData().getMetaDataMap().put("retryCount", retryCount + 1);
    item.getWorkflowData().getMetaDataMap().put("syncStatus", "RETRY");
    // The Goto Step in the model checks syncStatus == "RETRY" and loops back
} else {
    item.getWorkflowData().getMetaDataMap().put("syncStatus", "FAILED");
    throw new WorkflowException("Max retries exceeded");
}
```

## Performance Optimization & Throttling

In high-scale environments, workflows can saturate system resources if not properly managed.

### Throttled Task Runner

This OSGi configuration manages the thread pool for workflow execution.

- **Max Parallel Jobs:** Defines how many workflow steps can run concurrently.
- **Resource Limits:** Can be configured to stop spawning new workflow threads if the system CPU or Max Heap exceeds a defined percentage.

### Workflow Purging

For non-transient workflows, the `/var/workflow/instances` node grows indefinitely.

- **Maintenance Tasks:** Regular purging is mandatory to maintain JCR performance.
- **Filtering:** Purge configurations can be targeted by status (COMPLETED, TERMINATED, ABORTED) and age.

## Best Practices for Scalability

- **Service User Integration:** All automated steps should execute under a restricted Service User rather than a broad administrative session.
- **Granular Process Steps:** Break complex logic into multiple, single-purpose process steps to allow for better error isolation and logic reuse.
- **Event-Driven Integration:** For external system synchronization, utilize Adobe I/O events to offload waiting periods, rather than keeping a workflow instance "Running" or "Stale" — do not leave a workflow in `RUNNING` state waiting for a slow API, as this holds threads and resources.
- **Transient workflows for bulk operations:** Always use transient workflows for high-volume asset processing to avoid JCR bloat.
