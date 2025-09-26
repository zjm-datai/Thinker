
CosyVoice模型加载流程图

![[Untitled diagram _ Mermaid Chart-2025-09-26-032712.png|741x1827]]

```
flowchart TD
    %% 初始化入口
    Start["CosyVoice/CosyVoice2初始化"] --> ModelDirCheck{"模型目录存在?"}
    ModelDirCheck -->|否| ModelDownload["snapshot_download()从ModelScope下载"]
    ModelDirCheck -->|是| ConfigLoad["加载配置文件"]
    ModelDownload --> ConfigLoad
    
    %% 配置文件加载
    ConfigLoad --> ConfigType{"配置文件类型"}
    ConfigType -->|CosyVoice v1.0| LoadV1Config["加载cosyvoice.yaml"]
    ConfigType -->|CosyVoice v2.0| LoadV2Config["加载cosyvoice2.yaml"]
    
    %% 配置文件处理
    LoadV1Config --> ParseV1["hyperpyyaml.load_hyperpyyaml()"]
    LoadV2Config --> ParseV2["hyperpyyaml.load_hyperpyyaml()<br/>覆盖qwen_pretrain_path"]
    
    %% Frontend初始化
    ParseV1 --> InitFrontend["初始化CosyVoiceFrontEnd"]
    ParseV2 --> InitFrontend
    
    InitFrontend --> LoadTokenizer["加载tokenizer"]
    InitFrontend --> LoadCampplus["加载campplus.onnx<br/>说话人特征提取"]
    InitFrontend --> LoadSpeechTokenizer["加载speech_tokenizer_v1/v2.onnx<br/>语音tokenizer"]
    InitFrontend --> LoadSpkInfo["加载spk2info.pt<br/>说话人信息字典"]
    
    %% 核心模型初始化
    LoadSpkInfo --> InitModel["初始化CosyVoiceModel/CosyVoice2Model"]
    InitModel --> LoadCoreWeights["加载核心权重文件"]
    
    %% 核心权重加载
    LoadCoreWeights --> LoadLLM["torch.load('llm.pt')<br/>语言模型权重"]
    LoadCoreWeights --> LoadFlow["torch.load('flow.pt')<br/>Flow声学模型权重"]
    LoadCoreWeights --> LoadHiFT["torch.load('hift.pt')<br/>HiFT声码器权重"]
    
    %% 权重处理
    LoadLLM --> ProcessLLM["llm.load_state_dict()<br/>llm.to(device).eval()"]
    LoadFlow --> ProcessFlow["flow.load_state_dict()<br/>flow.to(device).eval()"]
    LoadHiFT --> ProcessHiFT["处理HiFT权重键名<br/>移除'generator.'前缀<br/>hift.load_state_dict()"]
    
    %% 优化选项检查
    ProcessHiFT --> OptimizationCheck{"优化选项检查"}
    OptimizationCheck --> JITCheck{"load_jit=True?"}
    OptimizationCheck --> TRTCheck{"load_trt=True?"}
    OptimizationCheck --> VLLMCheck{"load_vllm=True?<br/>(仅CosyVoice2)"}
    
    %% JIT优化加载
    JITCheck -->|是| LoadJIT["加载JIT编译模型"]
    LoadJIT --> JITFiles["torch.jit.load()<br/>llm.text_encoder.{fp16/fp32}.zip<br/>llm.llm.{fp16/fp32}.zip<br/>flow.encoder.{fp16/fp32}.zip"]
    JITFiles --> JITReplace["替换对应模型组件"]
    
    %% TensorRT优化加载
    TRTCheck -->|是| LoadTRT["加载TensorRT引擎"]
    LoadTRT --> TRTEngine["加载.plan引擎文件<br/>或从ONNX转换生成"]
    TRTEngine --> TRTReplace["替换flow.decoder.estimator<br/>为TrtContextWrapper"]
    
    %% vLLM优化加载
    VLLMCheck -->|是| LoadVLLM["加载vLLM模型"]
    LoadVLLM --> VLLMExport["export_cosyvoice2_vllm()<br/>导出vLLM格式"]
    VLLMExport --> VLLMEngine["初始化vLLM LLMEngine<br/>替换llm.vllm"]
    
    %% 完成初始化
    JITReplace --> InitComplete["初始化完成"]
    TRTReplace --> InitComplete
    VLLMEngine --> InitComplete
    OptimizationCheck -->|否| InitComplete
    
    %% 清理配置
    InitComplete --> CleanupConfig["del configs<br/>清理配置对象"]
    CleanupConfig --> Ready["模型就绪"]
    
    %% 样式定义
    classDef configClass fill:#e1f5fe
    classDef coreClass fill:#e8f5e8
    classDef frontendClass fill:#f3e5f5
    classDef optimizeClass fill:#fff3e0
    classDef completeClass fill:#e8f5e8
    
    class ConfigLoad,LoadV1Config,LoadV2Config,ParseV1,ParseV2 configClass
    class LoadCoreWeights,LoadLLM,LoadFlow,LoadHiFT,ProcessLLM,ProcessFlow,ProcessHiFT coreClass
    class InitFrontend,LoadTokenizer,LoadCampplus,LoadSpeechTokenizer,LoadSpkInfo frontendClass
    class JITCheck,TRTCheck,VLLMCheck,LoadJIT,LoadTRT,LoadVLLM optimizeClass
    class InitComplete,Ready completeClass
```

### 模型文件依赖关系图

![[Untitled diagram _ Mermaid Chart-2025-09-26-033150.png|422x800]]

```
flowchart LR
    %% 配置文件
    ConfigFile["cosyvoice.yaml<br/>或<br/>cosyvoice2.yaml"] --> ModelInit["模型初始化"]
    
    %% 核心权重文件
    LLMWeight["llm.pt<br/>(语言模型权重)"] --> CoreModels["核心模型"]
    FlowWeight["flow.pt<br/>(Flow声学模型权重)"] --> CoreModels
    HiFTWeight["hift.pt<br/>(HiFT声码器权重)"] --> CoreModels
    
    %% 前端ONNX模型
    CampPlus["campplus.onnx<br/>(说话人特征提取)"] --> Frontend["前端处理"]
    SpeechTokenizer["speech_tokenizer_v1/v2.onnx<br/>(语音tokenizer)"] --> Frontend
    SpkInfo["spk2info.pt<br/>(说话人信息字典)"] --> Frontend
    
    %% 优化文件
    JITFiles["JIT编译文件<br/>*.zip"] --> Optimization["性能优化"]
    TRTFiles["TensorRT引擎<br/>*.plan"] --> Optimization
    VLLMDir["vLLM模型目录<br/>(仅CosyVoice2)"] --> Optimization
    
    %% 汇总
    ModelInit --> CosyVoice["CosyVoice系统"]
    CoreModels --> CosyVoice
    Frontend --> CosyVoice
    Optimization --> CosyVoice
    
    %% 样式
    classDef configType fill:#e1f5fe
    classDef coreType fill:#e8f5e8
    classDef frontendType fill:#f3e5f5
    classDef optimizeType fill:#fff3e0
    
    class ConfigFile configType
    class LLMWeight,FlowWeight,HiFTWeight,CoreModels coreType
    class CampPlus,SpeechTokenizer,SpkInfo,Frontend frontendType
    class JITFiles,TRTFiles,VLLMDir,Optimization optimizeType
```

### 加载顺序时序图

![[Untitled diagram _ Mermaid Chart-2025-09-26-053912.png]]

```
sequenceDiagram
    participant Init as 初始化方法
    participant FS as 文件系统
    participant Config as 配置解析器
    participant Frontend as CosyVoiceFrontEnd
    participant Model as CosyVoiceModel
    participant Optimizer as 优化器
    
    Init->>FS: 检查模型目录
    alt 目录不存在
        Init->>FS: snapshot_download()下载模型
    end
    
    Init->>FS: 读取cosyvoice.yaml/cosyvoice2.yaml
    Init->>Config: hyperpyyaml.load_hyperpyyaml()
    Config-->>Init: 返回配置字典
    
    Init->>Frontend: 初始化CosyVoiceFrontEnd
    Frontend->>FS: 加载campplus.onnx
    Frontend->>FS: 加载speech_tokenizer_v*.onnx
    Frontend->>FS: 加载spk2info.pt
    Frontend-->>Init: 前端初始化完成
    
    Init->>Model: 初始化CosyVoiceModel/CosyVoice2Model
    Model->>FS: torch.load('llm.pt')
    Model->>FS: torch.load('flow.pt')
    Model->>FS: torch.load('hift.pt')
    Model->>Model: 加载状态字典到GPU/CPU
    Model-->>Init: 核心模型加载完成
    
    alt load_jit=True
        Init->>Optimizer: 加载JIT编译模型
        Optimizer->>FS: torch.jit.load(*.zip文件)
        Optimizer->>Model: 替换模型组件
    end
    
    alt load_trt=True
        Init->>Optimizer: 加载TensorRT引擎
        Optimizer->>FS: 加载.plan文件或从ONNX转换
        Optimizer->>Model: 替换decoder组件
    end
    
    alt load_vllm=True (CosyVoice2)
        Init->>Optimizer: 加载vLLM模型
        Optimizer->>FS: 导出/加载vLLM格式
        Optimizer->>Model: 初始化vLLM引擎
    end
    
    Init->>Init: del configs清理配置
    Init-->>Init: 模型就绪
```

