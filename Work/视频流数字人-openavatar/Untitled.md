

```
sequenceDiagram
    autonumber
    participant Browser as Browser (WebRTC)
    participant RtcStream as RtcStream<br/>(WebRTC Handler)
    participant Delegate as RtcClientSessionDelegate
    participant Session as ChatSession<br/>(inputs_pumper / handler_pumper / SignalBus)
    participant Engine as 引擎/Handlers
    participant ClientRTC as ClientHandlerRtc

    Browser->>RtcStream: 音频帧 receive()
    RtcStream->>Delegate: put_data(AUDIO, ndarray, ts)
    Delegate->>Session: data_submitter.submit(ChatData[MIC_AUDIO])
    Session->>Engine: 分发到各 Handler 输入队列

    Browser->>RtcStream: 视频帧 video_receive()
    RtcStream->>Delegate: put_data(VIDEO, frame, ts)
    Delegate->>Session: submit(ChatData[CAMERA_VIDEO])
    Session->>Engine: 分发到各 Handler

    Engine-->>ClientRTC: 产出 ChatData[MIC_AUDIO/CAMERA_VIDEO/HUMAN_TEXT]
    ClientRTC-->>Delegate: 放入 output_queues[对应 EngineChannelType]

    RtcStream->>Delegate: get_data(AUDIO)
    Delegate-->>RtcStream: ChatData -> (sr, audio_array)
    RtcStream-->>Browser: emit() 音频回推

    RtcStream->>Delegate: get_data(VIDEO)
    Delegate-->>RtcStream: ChatData -> frame
    RtcStream-->>Browser: video_emit() 视频回推
```


```
sequenceDiagram
    autonumber
    participant InQ as SessionContext.input_queues
    participant InPump as inputs_pumper
    participant DS as data_sinks[type]->List[DataSink]
    participant H1 as Handler A<br/>(handler_pumper)
    participant H2 as Handler B<br/>(handler_pumper)
    participant OUT as outputs[(handler,type)]->DataSink<br/>(session output_queues)
    participant RTC as ClientHandlerRtc / RTC

    Note over InQ: 音/视/文来自 RTC/其他客户端
    InQ->>InPump: 取一帧原始输入
    InPump->>InPump: packet_input_data() → ChatData(type)
    InPump->>DS: 查 data_sinks[ChatData.type]
    InPump->>H1: 投递到 H1.input_queue
    InPump->>H2: 投递到 H2.input_queue

    par Handler A 循环
      H1->>H1: 先 drain SignalBus (INTERRUPT 等)
      H1->>H1: input_queue.get_nowait()
      H1->>H1: handler.handle(...) → Iterable[outputs]
      loop 遍历每个输出
        H1->>OUT: 根据 outputs[(H1, out.type)] 投递到会话输出队列
        H1->>DS: 同时广播到 data_sinks[out.type]（除自己以外）
      end
    and Handler B 循环
      H2->>H2: 同上（先 drain signals）
      H2->>H2: input_queue.get_nowait()
      H2->>H2: handle → 产出
      H2->>OUT: 投 outputs[(H2, out.type)]
      H2->>DS: 广播到 data_sinks[out.type]（除自己）
    end

    Note over OUT,RTC: 会话输出队列由 ClientHandlerRtc 消费，<br/>再写入 RtcClientSessionDelegate.output_queues
    RTC-->>RTC: RtcStream.emit()/video_emit() 从这些队列取帧回推浏览器
```

![[Untitled diagram _ Mermaid Chart-2025-09-26-090314.png]]

```
sequenceDiagram
    participant Browser as Browser
    participant RTC as RTC Service
    participant ClientH as Client Handler
    participant SessionC as SessionContext
    participant ChatS as ChatSession
    participant Handlers as All Handlers
    participant Output as Output Queues

    Note over Browser,Output: 1. 会话初始化
    Browser->>RTC: 连接 WebRTC
    RTC->>ClientH: start_session
    ClientH->>ChatS: create_client_session
    ChatS->>SessionC: 创建 SessionContext
    ChatS->>Handlers: prepare all handlers
    Handlers-->>ChatS: 返回 HandlerEnv 和 HandlerContext

    Note over Browser,Output: 2. 数据处理循环
    loop 持续交互
        Browser->>RTC: 发送音视频
        RTC->>ClientH: 接收数据
        ClientH->>SessionC: 提交到输入队列
        SessionC->>Handlers: 分发数据到各处理器
        Handlers->>Handlers: 依次处理 VAD → ASR → LLM → TTS → Avatar
        Handlers->>Output: 输出到输出队列
        Output->>RTC: 发送响应
        RTC->>Browser: WebRTC 响应
    end
```


![[Untitled diagram _ Mermaid Chart-2025-09-26-083035.png]]

```
graph TB
    %% ===== 核心 Context =====
    subgraph Core_Context
        SC["SessionContext<br/>• 会话全局状态<br/>• 所有处理器共享<br/>• 生命周期: 整个会话"]
    end

    %% ===== 处理器特定 Context =====
    subgraph Handler_Specific_Context
        HC1["HandlerContext<br/>• 通用基类<br/>• 每个处理器实例一个"]

        VADC["HumanAudioVADContext<br/>• VAD 状态<br/>• 音频历史"]
        ASRC["ASRContext<br/>• 临时数据<br/>• 音频切片"]
        LLMC["LLMContext<br/>• 对话历史<br/>• API 客户端"]
        TTSC["TTSContext<br/>• TTS 任务队列<br/>• 音频缓存"]
        RTCC["ClientRtcContext<br/>• RTC 配置<br/>• 客户端代理"]
    end

    %% ===== 共享状态 =====
    subgraph Shared_States
        SS["SharedStates<br/>• active: 会话激活<br/>• enable_vad: VAD 开关"]
    end

    %% ===== 数据流管道 =====
    subgraph Data_Pipeline
        IQ["Input Queues<br/>• 音频/视频/文本输入"]
        OQ["Output Queues<br/>• 音频/视频输出"]
        DQ["DataBundle<br/>• 封装实际数据"]
    end

    %% ==== 关系连接 ====
    SC --> SS
    SC --> IQ
    SC --> OQ
    SC --> HC1

    HC1 --> VADC
    HC1 --> ASRC
    HC1 --> LLMC
    HC1 --> TTSC
    HC1 --> RTCC

    VADC --> SS
    ASRC --> SS
    LLMC --> SS
    TTSC --> SS
    RTCC --> SS

    %% ==== 数据使用关系 ====
    VADC -.-> DQ
    ASRC -.-> DQ
    LLMC -.-> DQ
    TTSC -.-> DQ
    RTCC -.-> DQ

    %% ==== 输入输出流向 ====
    IQ -.-> VADC
    VADC -.-> ASRC
    ASRC -.-> LLMC
    LLMC -.-> TTSC
    TTSC -.-> OQ

    %% ==== 样式 ====
    style SC fill:#e1f5fe
    style SS fill:#fff9c4
    style IQ fill:#f1f8e9
    style OQ fill:#f1f8e9
    style VADC fill:#e8f5e8
    style ASRC fill:#e8f5e8
    style LLMC fill:#e8f5e8
    style TTSC fill:#e8f5e8
    style RTCC fill:#e8f5e8
```

![[Untitled diagram _ Mermaid Chart-2025-09-26-083123.png]]

```
sequenceDiagram
    participant CE as ChatEngine
    participant CS as ChatSession
    participant HH as HandlerHub
    participant CH as ClientHandler
    participant H as HandlerX
    participant SC as SessionContext
    participant HC as HandlerContext

    Note over CE: 1 会话创建阶段
    CE->>CS: create_client_session
    CS->>SC: SessionContext init with input queues and output queues
    Note over SC: SessionContext 创建并初始化

    Note over CE: 2 处理器准备阶段
    CE->>HH: prepare_handlers with chat_config
    loop 每个 Handler
        HH->>H: get_handler_info
        HH->>H: get_handler_detail with session_context and context
        HH->>H: create_context with session_context and handler_config
        H-->>HC: 返回 HandlerContext
        Note over HC: HandlerContext 创建
    end

    Note over CE: 3 会话开始阶段
    CE->>CS: start
    CS->>CH: on_setup_session_delegate with session_context handler_context session_delegate
    Note over CH: ClientHandler 设置会话代理
    CS->>H: start_context with session_context and context
    Note over H: HandlerContext 启动

    Note over CE: 4 数据处理阶段
    loop 实时数据处理
        CS->>H: handle with context inputs output_definitions
        Note over H: Handler 处理数据 使用 HandlerContext
    end

    Note over CE: 5 会话结束阶段
    CE->>CS: stop_session
    CS->>H: destroy_context with context
    Note over H: HandlerContext 销毁
    CS->>SC: cleanup
    Note over SC: SessionContext 清理
```



```
sequenceDiagram
    participant Browser as Browser
    participant RTC as RtcStream
    participant CH as ClientHandlerRtc
    participant SC as SessionContext
    participant CS as ChatSession
    participant VADHE as VAD HandlerEnv
    participant VADH as VAD Handler
    participant VADHC as VAD HandlerContext
    participant ASRHE as ASR HandlerEnv
    participant ASRH as ASR Handler
    participant ASRHC as ASR HandlerContext
    participant LLMHE as LLM HandlerEnv
    participant LLMH as LLM Handler
    participant LLMHC as LLM HandlerContext
    participant TTSH as TTS Handler
    participant AvatarH as Avatar Handler
    participant VP as VAD_pumper
    participant AP as ASR_pumper
    participant LP as LLM_pumper
    participant VADQ as VAD_input_queue
    participant ASRQ as ASR_input_queue
    participant LLMQ as LLM_input_queue

    Note over Browser,AvatarH: 1. 系统初始化
    Browser->>RTC: 连接 WebRTC
    RTC->>CH: start_session
    CH->>CS: create_client_session
    CS->>SC: SessionContext()
    CS->>VADHE: prepare_handler(VAD)
    VADHE->>VADH: create_context
    VADH->>VADHC: 构建 VAD Context
    VADHC-->>VADHE: 返回 VADHandlerContext
    VADHE->>VADQ: 创建 VAD_input_queue
    CS->>ASRHE: prepare_handler(ASR)
    ASRHE->>ASRH: create_context
    ASRH->>ASRHC: 构建 ASR Context
    ASRHC-->>ASRHE: 返回 ASRHandlerContext
    ASRHE->>ASRQ: 创建 ASR_input_queue
    CS->>LLMHE: prepare_handler(LLM)
    LLMHE->>LLMH: create_context
    LLMH->>LLMHC: 构建 LLM Context
    LLMHC-->>LLMHE: 返回 LLMHandlerContext
    LLMHE->>LLMQ: 创建 LLM_input_queue

    Note over Browser,AvatarH: 2. 开始数据处理线程
    CS->>VP: VAD_pumper(session_context, VAD_handler_env,…)
    CS->>AP: ASR_pumper(session_context, ASR_handler_env,…)
    CS->>LP: LLM_pumper(session_context, LLM_handler_env,…)

    Note over Browser,AvatarH: 3. 音频数据从客户端到 VAD
    Browser->>RTC: 发送音频数据
    RTC->>CH: put_data(AUDIO,…)
    CH->>SC: submit 到 VAD 处理器
    SC->>VADQ: put 音频到 VAD 队列
    VP->>VADQ: get 音频数据
    VP->>VADH: handle(VAD_context, audio_data,…)
    VADH->>VADHC: 使用 VAD 上下文处理音频
    VADH-->>VP: 返回 HUMAN_AUDIO 数据
    VP->>SC: 分发到 ASR 处理器

    Note over Browser,AvatarH: 4. VAD 输出到 ASR 处理
    SC->>ASRQ: put 到 ASR 队列
    AP->>ASRQ: get 音频数据
    AP->>ASRH: handle(ASR_context, human_audio,…)
    ASRH->>ASRHC: 使用 ASR 上下文处理
    ASRH-->>AP: 返回 HUMAN_TEXT
    AP->>SC: 分发到 LLM 处理器

    Note over Browser,AvatarH: 5. ASR 输出到 LLM 处理
    SC->>LLMQ: put 到 LLM 队列
    LP->>LLMQ: get 文本数据
    LP->>LLMH: handle(LLM_context, human_text,…)
    LLMH->>LLMHC: 使用 LLM 上下文处理
    LLMH-->>LP: 返回 AVATAR_TEXT
    LP->>SC: 分发到 TTS 处理器

    Note over Browser,AvatarH: 6. LLM 输出到 TTS 和 Avatar
    SC->>TTSH: TTS 处理 AVATAR_TEXT
    TTSH-->>SC: 返回 AVATAR_AUDIO
    SC->>AvatarH: Avatar 处理 AVATAR_AUDIO
    AvatarH-->>SC: 返回 AVATAR_VIDEO
    SC->>RTC: 通过 output_queues 发送给客户端
    RTC->>Browser: WebRTC 发送音视频

    Note over Browser,AvatarH: 7. 循环持续处理
    loop 语音交互持续
        Browser->>RTC: 说话
        RTC->>VADQ: 音频数据
        VP->>VADH: VAD 检测
        VADH->>ASRH: 语音转文字
        ASRH->>LLMH: 智能对话
        LLMH->>TTSH: 文字转语音
        TTSH->>AvatarH: 生成头像
        AvatarH->>Browser: 返回音视频
    end
```




