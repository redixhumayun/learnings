## Understanding Performance Analysis

### The Iron Law Of Performance

$$\text{Performance} \propto \frac{\text{IPC} \times \text{Frequency}}{\text{Instruction Count}}$$

Or equivalently:

$$\text{Execution Time} \propto \frac{\text{Instruction Count}}{\text{IPC} \times \text{Frequency}}$$


### Wait Time, Service Time & Response Time

**Core Definitions:**

- **Wait Time (Queue Time)**: Time a request spends waiting before processing begins
- **Service Time**: Time spent actually processing the request
- **Response Time**: Total time from request arrival to completion

**Key Relationships:**

$$\text{Response Time} = \text{Wait Time} + \text{Service Time}$$

$$\text{Max Throughput} = \frac{1}{\text{Average Service Time}}$$

### Amdahl's Law

The theoretical speedup in the latency of a task is constrained by the parts of that it can be parallelized

$$\text{Speedup} = \frac{1}{s + \frac{1-s}{p}}$$

where:
- $s$ = fraction of serial parts
- $p$ = number of parallel processors

Example: In a streaming engine, if 20% of processing time is spent on serialization (inherently serial) and 80% on data transformation (parallelizable), even with unlimited cores, you can't exceed 5x speedup.

### Wait Time vs Response Time

$$\text{Response Time} = \text{Wait Time} + \text{Service Time}$$

Request Lifecycle in Database Query Processor:

```text
Timeline: ──────────────────────────────────────────────────→
         T0    T1              T2                    T3
         │     │               │                     │
         │     │←─ Wait Time ──│                     │
         │     │               │← Service Time ──────│
         │←────── Response Time ─────────────────────│
```

T0: Request arrives

T1: Request starts being processed (queuing delay ended)

T2: Processing begins 

T3: Response sent

Components:

• Queue Time: $T_1 - T_0$ (waiting for available worker)

• Service Time: $T_3 - T_2$ (actual processing)

• Response Time: $T_3 - T_0$ (total user experience)

### Little's Law

Defines the relationship between throughput, latency & concurrency in a steady state

$$L = \lambda \times W$$

Average # in System = Arrival Rate × Average Time in System

#### Example

- Arrival Rate: $\lambda = 100$ req/sec
- Average Response Time: $W = 50ms = 0.05$ sec

Therefore: Average Connections Busy: $L = 100 \times 0.05 = 5$

### Queuing Theory

$$\text{Response Time} = \frac{\text{Service Time}}{1 - \text{Utilization}}$$

## Relevant Links

1. [Iron Law Of Performance](https://blog.codingconfessions.com/p/one-law-to-rule-all-code-optimizations)