---
title: 'Moving to DAG for my workflow engine'
date: 2025-08-02
permalink: /posts/2025/08/moving-orchestration-to-dag/
excerpt: "Migration from linear execution to DAG<br/><img src='/images/linear-to-dag.png'>"
tags:
  - dev
  - updates
---

Current state of my workflow engine is a bit of a mess. Been using a simple chain of tasks that run one after another, but it's not cutting it anymore. As my projects grow, I need something more flexible and powerful.

To address these challenges, I started looking into what kind of Data structures I can use to handle orchestration. The common thread? Directed Acyclic Graphs (DAGs). They allow for complex branching and parallel execution, which is exactly what I need.

## Why Move to DAGs?

The linear chain approach had several critical limitations:
- **Blocking failures**: If one task failed, the entire workflow stopped
- **No parallelization**: Tasks that could run simultaneously were forced to wait
- **Complex branching**: Conditional workflows were impossible to implement

All of these issues were already solved by existing solutions like n8n or Zapier, not having this feature was a major blocker for me.

DAGs solve these problems by allowing nodes to execute in parallel when their dependencies are met, while ensuring proper execution order through dependency relationships.

<img src='/images/linear-to-dag.png' alt="Diagram showing the transformation from linear task execution to DAG-based parallel execution">


## What Changed

### Core Architecture Overhaul

The biggest change was completely reworking the `processEvent` function from sequential execution to parallel DAG processing. The old system processed nodes one by one:

```go
for {
    _, nodeID, err := s.manager.ProcessExecution(ctx, execution, connection)
    if nodeID == execution.LastNodeID && !execution.Running {
        return
    }
    time.Sleep(time.Second)
}
```

The new system identifies all runnable nodes and processes them concurrently:

```go
for {
    currentNodes := execution.CurrentNodeID
    if len(currentNodes) == 0 {
        currentNodes = execution.FirstNodeID
    }
    
    results := processNodesConcurrently(ctx, s.manager, execution, connection, currentNodes, maxWorkers)
    
    nextNodes := orchestrator.GetUnblockedNodes(workflow, execution.CompletedNodeIDs, allNodeIDs)
}
```

### Dependency Resolution System

Implemented proper dependency resolution using connection mappings. The system now analyzes workflow connections to determine which nodes can run in parallel:

```go
func getUpstreamNodes(workflow orchestrator.Workflow, nodeID string) []string {
    var upstream []string
    for _, conn := range workflow.Connections {
        if conn.Destination.ID == nodeID {
            upstream = append(upstream, conn.Source.ID)
        }
    }
    return upstream
}

func GetUnblockedNodes(workflow Workflow, completedNodes []string, allNodes []string) []string {
    var unblocked []string
    completed := make(map[string]struct{}, len(completedNodes))
    for _, n := range completedNodes {
        completed[n] = struct{}{}
    }
    for _, node := range allNodes {
        upstream := getUpstreamNodes(workflow, node)
        allDone := true
        for _, up := range upstream {
            if _, ok := completed[up]; !ok {
                allDone = false
                break
            }
        }
        if allDone {
            unblocked = append(unblocked, node)
        }
    }
    return unblocked
}
```

### Parallel Execution with Worker Pools

Added bounded worker pools to prevent resource exhaustion while maximizing concurrency:

```go
func processNodesConcurrently(ctx context.Context, manager *orchestrator.ExecutionManager, execution database.Execution, connection database.Connection, nodeIDs []string, maxWorkers int) []processResult {
    results := make([]processResult, 0, len(nodeIDs))
    jobs := make(chan string)
    resCh := make(chan processResult)
    var wg sync.WaitGroup

    worker := func() {
        defer wg.Done()
        for nodeID := range jobs {
            _, _, err := manager.ProcessExecution(ctx, execution, connection, nodeID)
            resCh <- processResult{nodeID: nodeID, err: err}
        }
    }

    for i := 0; i < maxWorkers; i++ {
        wg.Add(1)
        go worker()
    }
}
```

### Advanced Status Tracking

Replaced simple boolean flags with comprehensive node status tracking:

```go
type ExecutionManager struct {
    NodeStatusMap    map[string]string // nodeID -> "success"/"failed"
    NodeStatusIDs    map[string]string // nodeID -> RCE command ID
    CompletedNodeIDs []string
    FailedNodeIDs    []string
    CurrentNodeID    []string
}

func CalculateExecutionStatus(execution *database.Execution, workflow *Workflow) string {
    // 1. If all last nodes are completed, status is "completed"
    if allNodesInSlice(execution.LastNodeID, execution.CompletedNodeIDs) {
        return "completed"
    }
    
    // 2. Check if failed nodes block downstream execution
    for _, failedNode := range execution.FailedNodeIDs {

    }
    
    // 3. If failed node do not block downstream execution mark it as partial completion
    if len(execution.FailedNodeIDs) > 0 {
        return "partial"
    }
    
    return "running"
}
```

## Performance Impact

- **Execution Time**: Workflows with parallel branches now complete significantly faster
- **Failure Isolation**: Independent workflow branches continue executing even when other paths fail

## Migration Process

The migration involved several key database schema changes:

1. **Multi-Node Support**: Changed `CurrentNodeID` from single string to `[]string` slice
2. **Status Tracking**: Added `NodeStatusMap`, `NodeStatusIDs`, `CompletedNodeIDs`, and `FailedNodeIDs`
3. **Execution States**: Introduced "partial" status for workflows with mixed success/failure outcomes

## What's Next

The DAG implementation is working well, but there are still some areas for improvement:

- **Dynamic Scaling**: Currently using fixed worker pools, could implement auto-scaling based on workflow complexity
- **Moving to Temporal.io**: It would be a good idea to migrate to Temporal.io for more robust workflow orchestration as we scale but thats a problem for future me.

The migration to DAGs has been one of the best architectural decisions I've made this year. The project now feels much more closer to n8n or zapier. Nonetheless, there's still a lot for work for me to do and improve.