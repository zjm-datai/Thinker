

```
sequenceDiagram

      participant Client as API Client

      participant OpenAIServing as OpenAIServing

      participant EngineClient as Engine Client

      participant Scheduler as Scheduler

      participant SequenceGroup as Sequence Group

      participant ModelRunner as Model Runner

      participant KVCache as KV Cache

  

      Client->>OpenAIServing: Submit request (prompt/completion)

      OpenAIServing->>EngineClient: Add request to waiting queue

      EngineClient->>Scheduler: Check scheduling budget

      Scheduler->>Scheduler: Evaluate waiting/running/swapped queues

      Scheduler->>KVCache: Check available memory blocks

      KVCache-->>Scheduler: Memory availability status

  

      alt Budget and memory available

          Scheduler->>SequenceGroup: Create/assign SequenceGroup

          Scheduler->>Scheduler: Add to running queue

          Scheduler-->>EngineClient: Return scheduled sequence groups

          EngineClient->>ModelRunner: Execute forward pass

          ModelRunner->>ModelRunner: Process tokens using KV cache

          ModelRunner-->>EngineClient: Return output tokens

          EngineClient-->>OpenAIServing: Send response tokens

          OpenAIServing-->>Client: Return completion/response

      else Memory constraints

          Scheduler->>SequenceGroup: Preempt lowest-priority sequence
          Scheduler->>KVCache: Swap out blocks to CPU
          Scheduler->>Scheduler: Move sequence to swapped queue
          Scheduler->>SequenceGroup: Admit new sequence
          Scheduler->>KVCache: Allocate KV cache blocks
      end
```