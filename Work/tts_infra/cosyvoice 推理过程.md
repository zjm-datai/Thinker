


![[Untitled diagram _ Mermaid Chart-2025-09-26-031226.png]]




```
sequenceDiagram
    participant User as 用户代码
    participant CV as CosyVoice
    participant FE as CosyVoiceFrontEnd
    participant Model as CosyVoiceModel
    participant LLM as LLM线程
    participant Flow as Flow模型
    participant HiFT as HiFT声码器
    
    User->>CV: inference_zero_shot(tts_text, prompt_text, prompt_speech_16k)
    CV->>FE: text_normalize(tts_text)
    FE-->>CV: normalized_texts[]
    
    loop 对每个文本片段
        CV->>FE: frontend_zero_shot(text, prompt_text, prompt_speech_16k)
        FE->>FE: _extract_text_token(text)
        FE->>FE: _extract_speech_token(prompt_speech_16k)
        FE->>FE: _extract_spk_embedding(prompt_speech_16k)
        FE->>FE: _extract_speech_feat(prompt_speech_16k)
        FE-->>CV: model_input dict
        
        CV->>Model: tts(**model_input, stream=stream, speed=speed)
        Model->>Model: 生成UUID，初始化session字典
        
        par LLM推理线程
            Model->>LLM: llm_job(text, prompt_text, tokens, embedding, uuid)
            LLM->>LLM: llm.inference()
            loop 生成token
                LLM->>Model: tts_speech_token_dict[uuid].append(token)
            end
            LLM->>Model: llm_end_dict[uuid] = True
        and 主线程处理
            alt stream=True
                loop 流式处理
                    Model->>Model: 检查token数量
                    Model->>Model: token2wav(tokens, prompt_token, prompt_feat, embedding, uuid)
                    Model->>Flow: flow.inference()
                    Flow-->>Model: mel_spectrogram
                    Model->>HiFT: hift.inference(mel_spectrogram)
                    HiFT-->>Model: audio_chunk
                    Model-->>CV: yield {'tts_speech': audio_chunk}
                end
            else stream=False
                Model->>Model: p.join() 等待LLM完成
                Model->>Model: token2wav(all_tokens, ...)
                Model->>Flow: flow.inference()
                Flow-->>Model: mel_spectrogram
                Model->>HiFT: hift.inference(mel_spectrogram)
                HiFT-->>Model: final_audio
                Model-->>CV: yield {'tts_speech': final_audio}
            end
        end
        
        Model->>Model: 清理session字典
        CV-->>User: yield model_output
    end
```