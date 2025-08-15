
```python
@router.post("/v1/audio/transcriptions")
@with_cancellation
@load_aware_call
async def create_transcriptions(request: Annotated[TranscriptionRequest,
                                                   Form()],
                                raw_request: Request):
    handler = transcription(raw_request)
    if handler is None:
        return base(raw_request).create_error_response(
            message="The model does not support Transcriptions API")

    audio_data = await request.file.read()
    generator = await handler.create_transcription(audio_data, request,
                                                   raw_request)

    if isinstance(generator, ErrorResponse):
        return JSONResponse(content=generator.model_dump(),
                            status_code=generator.code)

    elif isinstance(generator, TranscriptionResponse):
        return JSONResponse(content=generator.model_dump())

    return StreamingResponse(content=generator, media_type="text/event-stream") 
```

```python
# vllm/entrypoints/openai/api_server.py

def transcription(request: Request) -> OpenAIServingTranscription:
    return request.app.state.openai_serving_transcription

async def init_app_state():
    ...
    state.openai_serving_transcription = OpenAIServingTranscription(
        engine_client,
        model_config,
        state.openai_serving_models,
        request_logger=request_logger,
    ) if model_config.runner_type == "transcription" else None

async def run_server():
    ...
    async with build_async_engine_client(args) as engine_client:
        app = build_app(args)

        vllm_config = await engine_client.get_vllm_config()
        await init_app_state(engine_client, vllm_config, app.state, args)
        ...
```