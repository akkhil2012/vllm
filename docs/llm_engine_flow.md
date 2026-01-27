# LLMEngine flow diagram

This diagram summarizes the main control flow in `vllm/v1/engine/llm_engine.py`,
covering initialization, request intake, and the `step()` loop that drives
EngineCore output processing.

```mermaid
flowchart TD
    %% Initialization
    Start([LLMEngine.__init__]) --> LoadConfig[vllm_config + parallel config]
    LoadConfig --> Tokenizer{skip_tokenizer_init?}
    Tokenizer -->|yes| NoTok[Tokenizer = None]
    Tokenizer -->|no| InitTok[init_tokenizer_from_configs]
    NoTok --> Proc[Processor(vllm_config, tokenizer)]
    InitTok --> Proc
    Proc --> IOProc[get_io_processor]
    IOProc --> OutProc[OutputProcessor(stream_interval)]
    OutProc --> Core[EngineCoreClient.make_client]
    Core --> Loggers{log_stats?}
    Loggers -->|yes| LoggerMgr[StatLoggerManager]
    Loggers -->|no| Ready[Engine ready]
    LoggerMgr --> Ready

    %% add_request flow
    Ready --> AddReq[add_request]
    AddReq --> ValidateId[Validate request_id type]
    ValidateId --> InputType{prompt type?}
    InputType -->|EngineCoreRequest| UseReq[Use provided request]
    InputType -->|PromptType| ProcessInputs[Processor.process_inputs]
    UseReq --> Params[params = request.params]
    ProcessInputs --> Params
    Params --> FanOut{n == 1?}
    FanOut -->|yes| QueueReq[OutputProcessor.add_request]
    QueueReq --> CoreAdd[EngineCore.add_request]
    FanOut -->|no| ParentReq[ParentRequest + child fan-out]
    ParentReq --> ChildQueue[OutputProcessor.add_request (each child)]
    ChildQueue --> ChildCore[EngineCore.add_request (each child)]

    %% step loop
    Ready --> Step[step]
    Step --> Dummy{should_execute_dummy_batch?}
    Dummy -->|yes| DummyBatch[EngineCore.execute_dummy_batch]
    DummyBatch --> StepDone([return []])
    Dummy -->|no| GetOut[EngineCore.get_output]
    GetOut --> Process[OutputProcessor.process_outputs]
    Process --> UpdateStats[update_scheduler_stats]
    UpdateStats --> Abort[EngineCore.abort_requests]
    Abort --> Record{log_stats?}
    Record -->|yes| RecordStats[StatLoggerManager.record]
    RecordStats --> MaybeLog[do_log_stats_with_interval]
    Record -->|no| ReturnOut
    MaybeLog --> ReturnOut[return request_outputs]
```

## Notes
- `vllm/engine/llm_engine.py` is a thin alias to this v1 implementation, so the
  flow above reflects the behavior used by the public `LLMEngine` symbol.
- `step()` is the primary loop: fetch outputs from EngineCore, process them into
  request outputs, abort finished requests, and record stats when enabled.
