# STUDY 5: Concurrent Workflow Execution in n8n Open Source

## Executive Summary

The open source community version of n8n has **NO DEFAULT LIMIT** on concurrent workflow executions, making it more generous than the paid version which limits users to 5 concurrent executions. The concurrency system is fully configurable and designed for resource management rather than artificial licensing restrictions.

## 1. Concurrent Workflow Execution Limits

### Open Source vs Paid Comparison

| Version | Default Concurrent Executions | Configurable |
|---------|------------------------------|--------------|
| **Open Source Community** | **Unlimited (-1)** | ✅ Yes |
| **Paid ($20/month)** | **5** | Unknown |

### Key Configuration Parameters

```typescript
// Default values in packages/cli/src/config/schema.ts
executions: {
  concurrency: {
    productionLimit: {
      default: -1,  // -1 = UNLIMITED
      env: 'N8N_CONCURRENCY_PRODUCTION_LIMIT',
    },
    evaluationLimit: {
      default: -1,  // -1 = UNLIMITED  
      env: 'N8N_CONCURRENCY_EVALUATION_LIMIT',
    },
  },
}
```

### Cloud Deployment Notes

- There's a `CLOUD_TEMP_PRODUCTION_LIMIT = 999` constant used for telemetry tracking
- This appears to be for cloud deployment monitoring, not a hard limit for self-hosted instances

## 2. Understanding "Concurrent Workflow Execution"

### Scope of Concurrency

**Concurrent execution means N executions across ALL workflows, not per individual workflow template.**

The system operates with two separate concurrency queues:

#### Production Queue
- Handles real workflow executions
- Applies to execution modes: `webhook`, `trigger`
- Controlled by `N8N_CONCURRENCY_PRODUCTION_LIMIT`

#### Evaluation Queue  
- Handles testing/preview executions
- Applies to execution mode: `evaluation`
- Controlled by `N8N_CONCURRENCY_EVALUATION_LIMIT`

### Execution Modes and Concurrency

```typescript
// Modes that BYPASS concurrency limits (unlimited):
- 'cli'        // Command line executions
- 'error'      // Error workflow executions  
- 'integrated' // Integrated executions
- 'internal'   // Internal system executions
- 'manual'     // Manual test executions
- 'retry'      // Retry executions

// Modes that RESPECT concurrency limits:
- 'webhook'    // → Production queue
- 'trigger'    // → Production queue  
- 'evaluation' // → Evaluation queue
```

## 3. Technical Implementation

### Architecture Overview

The concurrency system is implemented in TypeScript/JavaScript (not Python) using:

- **Service**: `ConcurrencyControlService` - Main orchestration
- **Queue**: `ConcurrencyQueue` - FIFO queue with capacity management
- **Configuration**: Environment variables and config schema
- **Events**: Telemetry and throttling events

### Core Components

```typescript
// packages/cli/src/concurrency/concurrency-control.service.ts
export class ConcurrencyControlService {
  private readonly limits: Map<ConcurrencyQueueType, number>;
  private readonly queues: Map<ConcurrencyQueueType, ConcurrencyQueue>;
  
  // Key methods:
  async throttle({ mode, executionId })  // Queue execution if needed
  release({ mode })                      // Release capacity
  remove({ mode, executionId })          // Remove from queue
}
```

### Queue Behavior

```typescript
// packages/cli/src/concurrency/concurrency-queue.ts
export class ConcurrencyQueue {
  async enqueue(executionId: string) {
    this.capacity--;
    if (this.capacity < 0) {
      // Execution is queued, not rejected
      return new Promise<void>((resolve) => 
        this.queue.push({ executionId, resolve })
      );
    }
  }
}
```

## 4. Configuration Options

### Environment Variables

```bash
# Production executions (webhook, trigger modes)
N8N_CONCURRENCY_PRODUCTION_LIMIT=-1    # Default: unlimited

# Evaluation executions (testing mode)  
N8N_CONCURRENCY_EVALUATION_LIMIT=-1    # Default: unlimited
```

### Valid Configuration Values

| Value | Behavior |
|-------|----------|
| `-1` | **Unlimited** (default) |
| `> 0` | **Limited** to specified number |
| `0` | **Invalid** - throws error |
| `< -1` | **Treated as unlimited** |

### Worker Mode Configuration

```typescript
// packages/cli/src/commands/worker.ts
static flags = {
  concurrency: Flags.integer({
    default: 10,  // Default worker concurrency
    description: 'How many jobs can run in parallel.',
  }),
};

// Environment variable overrides flag:
this.concurrency = envConcurrency !== -1 ? envConcurrency : flags.concurrency;
```

## 5. Execution Flow Analysis

### When Concurrency Control Applies

```mermaid
graph TD
    A[Workflow Execution Request] --> B{Execution Mode?}
    B -->|cli, error, integrated, internal, manual, retry| C[Bypass Concurrency - Execute Immediately]
    B -->|webhook, trigger| D[Check Production Queue]
    B -->|evaluation| E[Check Evaluation Queue]
    
    D --> F{Production Capacity Available?}
    E --> G{Evaluation Capacity Available?}
    
    F -->|Yes| H[Execute Immediately]
    F -->|No| I[Queue Execution - Wait for Capacity]
    
    G -->|Yes| H
    G -->|No| I
    
    I --> J[Previous Execution Completes]
    J --> K[Release Capacity]
    K --> L[Dequeue Next Execution]
```

### What Happens When Limits Are Reached

**IMPORTANT: Executions are QUEUED, not DROPPED when limits are reached.**

#### Example Scenario: Limit Set to 10

```bash
N8N_CONCURRENCY_PRODUCTION_LIMIT=10
```

**Timeline of Events:**

1. **Requests 1-10**: Execute immediately (capacity available)
2. **Request 11**: Gets **QUEUED** (not dropped), waits for capacity
3. **Request 12**: Gets **QUEUED** behind request 11
4. **Request 13+**: Continue queuing in FIFO order

**When Execution #3 completes (after 100 seconds):**
- Capacity increases from 0 to 1
- Request 11 is **automatically dequeued** and starts executing
- Queue now has requests 12, 13, 14... waiting

#### Code Implementation

```typescript
// packages/cli/src/concurrency/concurrency-queue.ts
async enqueue(executionId: string) {
  this.capacity--;  // Reduce available capacity
  
  if (this.capacity < 0) {
    // QUEUE the execution (don't drop it)
    this.emit('execution-throttled', { executionId });
    
    return new Promise<void>((resolve) => 
      this.queue.push({ executionId, resolve })
    );
  }
  // If capacity >= 0, execution proceeds immediately
}
```

### Queue Management

- **FIFO**: First In, First Out execution order
- **Non-blocking**: Executions are queued, not rejected or dropped
- **Capacity tracking**: Real-time monitoring of available slots
- **Event-driven**: Telemetry and monitoring events
- **Promise-based**: Queued executions wait on promises that resolve when capacity becomes available

### Understanding "Concurrent" in Workflow Context

**"Concurrent workflow execution" means multiple COMPLETE workflow runs happening simultaneously.**

#### What Constitutes One "Concurrent Execution"

Each workflow execution (from trigger to completion) counts as ONE concurrent execution:

```
Workflow A: Trigger → Node 1 → Node 2 → Node 3 → Complete (100 seconds total)
Workflow B: Trigger → Node 1 → Node 2 → Complete (50 seconds total)
Workflow C: Trigger → Node 1 → Complete (10 seconds total)
```

If limit = 2:
- **Time 0s**: Workflow A starts, Workflow B starts (2/2 slots used)
- **Time 5s**: Workflow C gets **QUEUED** (waits for A or B to finish)
- **Time 50s**: Workflow B completes → Workflow C **automatically starts**
- **Time 100s**: Workflow A completes → Next queued workflow can start

#### Long-Running Workflows Impact

**Yes, the queue waits for long-running workflows to complete.**

**Example with 100-second workflow:**

```bash
# Limit set to 3 concurrent executions
N8N_CONCURRENCY_PRODUCTION_LIMIT=3

Timeline:
00:00 - Workflow 1 starts (webhook trigger) - will run for 100 seconds
00:05 - Workflow 2 starts (webhook trigger) - will run for 30 seconds  
00:10 - Workflow 3 starts (webhook trigger) - will run for 20 seconds
00:15 - Workflow 4 QUEUED (waits for any of 1,2,3 to finish)
00:20 - Workflow 5 QUEUED (waits behind Workflow 4)

00:30 - Workflow 3 completes → Workflow 4 starts immediately
00:35 - Workflow 2 completes → Workflow 5 starts immediately
01:40 - Workflow 1 completes (after 100 seconds)
```

**Key Points:**
- Each workflow holds its slot for the **entire duration** of execution
- Long-running workflows (100+ seconds) will block new executions if limit is reached
- The system does **NOT** interrupt or time-out long-running workflows
- Queued workflows wait patiently until capacity becomes available

#### Within-Workflow Concurrency

**Important distinction**: The concurrency limit applies to **complete workflow executions**, not individual nodes within a workflow.

```
Single Workflow Execution = 1 Concurrent Slot
├── Node 1 (HTTP Request) - 5 seconds
├── Node 2 (Data Processing) - 2 seconds  
├── Node 3 (Database Write) - 3 seconds
└── Node 4 (Email Send) - 1 second
Total: 11 seconds = 1 concurrent execution slot
```

Even if Node 1 makes 10 parallel HTTP requests internally, this still counts as **1 concurrent workflow execution**.

## 6. Monitoring and Telemetry

### Cloud Deployment Tracking

```typescript
// Telemetry thresholds for cloud deployments
export const CLOUD_TEMP_REPORTABLE_THRESHOLDS = [5, 10, 20, 50, 100, 200];

// Tracks when users approach limits
this.telemetry.track('User hit concurrency limit', {
  threshold: CLOUD_TEMP_PRODUCTION_LIMIT - capacity,
  concurrencyQueue: type,
});
```

### Events Emitted

- `execution-throttled`: When execution is queued due to capacity
- `execution-released`: When execution completes and releases capacity
- `concurrency-check`: Periodic capacity monitoring

## 7. Practical Implications

### For Self-Hosted Users

1. **No artificial limits** - Run as many concurrent workflows as your hardware can handle
2. **Resource-based scaling** - Limited only by CPU, memory, and I/O capacity
3. **Configurable controls** - Set limits based on your infrastructure needs

### Recommended Settings

```bash
# For small deployments (1-2 CPU cores)
N8N_CONCURRENCY_PRODUCTION_LIMIT=5

# For medium deployments (4-8 CPU cores)  
N8N_CONCURRENCY_PRODUCTION_LIMIT=20

# For large deployments (8+ CPU cores)
N8N_CONCURRENCY_PRODUCTION_LIMIT=50

# For unlimited (default - use with caution)
N8N_CONCURRENCY_PRODUCTION_LIMIT=-1
```

### Worker Mode Considerations

```bash
# Worker concurrency warning threshold
if (this.concurrency < 5) {
  this.logger.warn(
    'Concurrency is set to less than 5. THIS CAN LEAD TO AN UNSTABLE ENVIRONMENT.'
  );
}
```

## 8. Comparison with Other Systems

| System | Open Source Limit | Paid Limit | Configurable |
|--------|------------------|------------|--------------|
| **n8n** | **Unlimited** | 5 | ✅ Yes |
| Zapier | N/A | 20-100+ | ❌ No |
| Make.com | N/A | 10-40+ | ❌ No |
| Integromat | N/A | Variable | ❌ No |

## 9. Key Takeaways

1. **Open source n8n is MORE generous** than the paid cloud version for concurrent executions
2. **Fully configurable** - no artificial restrictions for self-hosted deployments  
3. **Smart architecture** - different execution types have appropriate concurrency controls
4. **Resource-focused** - designed for performance optimization, not monetization limits
5. **Production-ready** - includes proper queuing, monitoring, and telemetry systems

## 10. Configuration Examples

### Unlimited Concurrency (Default)
```bash
# .env file
N8N_CONCURRENCY_PRODUCTION_LIMIT=-1
N8N_CONCURRENCY_EVALUATION_LIMIT=-1
```

### Conservative Resource Management
```bash
# .env file  
N8N_CONCURRENCY_PRODUCTION_LIMIT=10
N8N_CONCURRENCY_EVALUATION_LIMIT=5
```

### High-Performance Setup
```bash
# .env file
N8N_CONCURRENCY_PRODUCTION_LIMIT=100
N8N_CONCURRENCY_EVALUATION_LIMIT=20
```

---

**Conclusion**: The open source version of n8n provides unlimited concurrent workflow execution by default, making it significantly more capable than the paid version's 5-execution limit. This demonstrates n8n's commitment to providing powerful automation capabilities in their open source offering, with concurrency controls designed for resource management rather than artificial limitations.
