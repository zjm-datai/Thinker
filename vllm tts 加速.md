
### vllm_example

```
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# python vllm_example.py 
INFO 08-20 14:01:30 [__init__.py:243] Automatically detected platform cuda.
[2025-08-20 14:01:34,284] [INFO] [real_accelerator.py:203:get_accelerator] Setting ds_accelerator to cuda (auto detect)
df: /root/.triton/autotune: No such file or directory

A module that was compiled using NumPy 1.x cannot be run in
NumPy 2.2.6 as it may crash. To support both 1.x and 2.x
versions of NumPy, modules must be compiled with NumPy 2.0.
Some module may need to rebuild instead e.g. with 'pybind11>=2.12'.

If you are a user of the module, the easiest solution will be to
downgrade to 'numpy<2' or try to upgrade the affected module.
We expect that some modules will need time to support NumPy 2.

Traceback (most recent call last):  File "/data/workspaces/cosyvoice_test/vllm_example.py", line 7, in <module>
    from cosyvoice.cli.cosyvoice import CosyVoice2
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/cosyvoice.py", line 21, in <module>
    from cosyvoice.cli.frontend import CosyVoiceFrontEnd
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/frontend.py", line 17, in <module>
    import onnxruntime
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 23, in <module>
    from onnxruntime.capi._pybind_state import ExecutionMode  # noqa: F401
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/capi/_pybind_state.py", line 32, in <module>
    from .onnxruntime_pybind11_state import *  # noqa
AttributeError: _ARRAY_API not found
Traceback (most recent call last):
  File "/data/workspaces/cosyvoice_test/vllm_example.py", line 7, in <module>
    from cosyvoice.cli.cosyvoice import CosyVoice2
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/cosyvoice.py", line 21, in <module>
    from cosyvoice.cli.frontend import CosyVoiceFrontEnd
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/frontend.py", line 17, in <module>
    import onnxruntime
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 57, in <module>
    raise import_capi_exception
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 23, in <module>
    from onnxruntime.capi._pybind_state import ExecutionMode  # noqa: F401
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/capi/_pybind_state.py", line 32, in <module>
    from .onnxruntime_pybind11_state import *  # noqa
ImportError
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# pip install onnxruntime-gpu
Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple/
Requirement already satisfied: onnxruntime-gpu in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (1.18.0)
Requirement already satisfied: coloredlogs in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (15.0.1)
Requirement already satisfied: flatbuffers in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (25.2.10)
Requirement already satisfied: numpy>=1.21.6 in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (2.2.6)
Requirement already satisfied: packaging in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (24.2)
Requirement already satisfied: protobuf in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (6.32.0)
Requirement already satisfied: sympy in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from onnxruntime-gpu) (1.14.0)
Requirement already satisfied: humanfriendly>=9.1 in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from coloredlogs->onnxruntime-gpu) (10.0)
Requirement already satisfied: mpmath<1.4,>=1.1.0 in /root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages (from sympy->onnxruntime-gpu) (1.3.0)
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# python vllm_example.py 
INFO 08-20 14:02:31 [__init__.py:243] Automatically detected platform cuda.
[2025-08-20 14:02:35,295] [INFO] [real_accelerator.py:203:get_accelerator] Setting ds_accelerator to cuda (auto detect)

A module that was compiled using NumPy 1.x cannot be run in
NumPy 2.2.6 as it may crash. To support both 1.x and 2.x
versions of NumPy, modules must be compiled with NumPy 2.0.
Some module may need to rebuild instead e.g. with 'pybind11>=2.12'.

If you are a user of the module, the easiest solution will be to
downgrade to 'numpy<2' or try to upgrade the affected module.
We expect that some modules will need time to support NumPy 2.

Traceback (most recent call last):  File "/data/workspaces/cosyvoice_test/vllm_example.py", line 7, in <module>
    from cosyvoice.cli.cosyvoice import CosyVoice2
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/cosyvoice.py", line 21, in <module>
    from cosyvoice.cli.frontend import CosyVoiceFrontEnd
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/frontend.py", line 17, in <module>
    import onnxruntime
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 23, in <module>
    from onnxruntime.capi._pybind_state import ExecutionMode  # noqa: F401
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/capi/_pybind_state.py", line 32, in <module>
    from .onnxruntime_pybind11_state import *  # noqa
AttributeError: _ARRAY_API not found
Traceback (most recent call last):
  File "/data/workspaces/cosyvoice_test/vllm_example.py", line 7, in <module>
    from cosyvoice.cli.cosyvoice import CosyVoice2
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/cosyvoice.py", line 21, in <module>
    from cosyvoice.cli.frontend import CosyVoiceFrontEnd
  File "/data/workspaces/cosyvoice_test/cosyvoice/cli/frontend.py", line 17, in <module>
    import onnxruntime
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 57, in <module>
    raise import_capi_exception
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/__init__.py", line 23, in <module>
    from onnxruntime.capi._pybind_state import ExecutionMode  # noqa: F401
  File "/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/onnxruntime/capi/_pybind_state.py", line 32, in <module>
    from .onnxruntime_pybind11_state import *  # noqa
ImportError
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# pip install "numpy<2"
Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple/
Collecting numpy<2
  Using cached https://pypi.tuna.tsinghua.edu.cn/packages/4b/d7/ecf66c1cd12dc28b4040b15ab4d17b773b87fa9d29ca16125de01adb36cd/numpy-1.26.4-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (18.2 MB)
Installing collected packages: numpy
  Attempting uninstall: numpy
    Found existing installation: numpy 2.2.6
    Uninstalling numpy-2.2.6:
      Successfully uninstalled numpy-2.2.6
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
openai-whisper 20231117 requires triton<3,>=2.0.0, but you have triton 3.3.0 which is incompatible.
opencv-python-headless 4.12.0.88 requires numpy<2.3.0,>=2; python_version >= "3.9", but you have numpy 1.26.4 which is incompatible.
Successfully installed numpy-1.26.4
WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# python vllm_example.py 
INFO 08-20 14:03:19 [__init__.py:243] Automatically detected platform cuda.
[2025-08-20 14:03:23,258] [INFO] [real_accelerator.py:203:get_accelerator] Setting ds_accelerator to cuda (auto detect)
failed to import ttsfrd, use wetext instead
Sliding Window Attention is enabled but not implemented for `sdpa`; unexpected results may be encountered.
/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/diffusers/models/lora.py:393: FutureWarning: `LoRACompatibleLinear` is deprecated and will be removed in version 1.0.0. Use of `LoRACompatibleLinear` is deprecated. Please switch to PEFT backend by installing PEFT: `pip install peft`.
  deprecate("LoRACompatibleLinear", "1.0.0", deprecation_message)
2025-08-20 14:03:28,584 INFO input frame rate=25
/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/torch/nn/utils/weight_norm.py:143: FutureWarning: `torch.nn.utils.weight_norm` is deprecated in favor of `torch.nn.utils.parametrizations.weight_norm`.
  WeightNorm.apply(module, name, dim)
2025-08-20 14:03:30.483626956 [E:onnxruntime:Default, provider_bridge_ort.cc:1744 TryGetProviderInfo_CUDA] /onnxruntime_src/onnxruntime/core/session/provider_bridge_ort.cc:1426 onnxruntime::Provider& onnxruntime::ProviderLibrary::Get() [ONNXRuntimeError] : 1 : FAIL : Failed to load library libonnxruntime_providers_cuda.so with error: libcudnn.so.8: cannot open shared object file: No such file or directory

2025-08-20 14:03:30.483659654 [W:onnxruntime:Default, onnxruntime_pybind_state.cc:870 CreateExecutionProviderInstance] Failed to create CUDAExecutionProvider. Please reference https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#requirementsto ensure all dependencies are met.
2025-08-20 14:03:31,420 - modelscope - WARNING - Authentication has expired, please re-login with modelscope login --token "YOUR_SDK_TOKEN" if you need to access private models or datasets.
Downloading Model to directory: /root/.cache/modelscope/hub/pengzhendong/wetext
2025-08-20 14:03:31,421 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:31,812 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/revisions HTTP/1.1" 200 205
2025-08-20 14:03:31,908 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo/files?Revision=master&Recursive=True HTTP/1.1" 200 None
Downloading [configuration.json]:   0%|                                                                                                          | 0.00/36.0 [00:00<?, ?B/s]2025-08-20 14:03:31,912 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:32,698 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=configuration.json HTTP/1.1" 302 344
2025-08-20 14:03:32,701 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:33,202 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/9a/9b/41224519cc2cd5e993d9bd161a4165bb0f7ddc58a91a63f8b0ced41ad4de?filename=configuration.json&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669812-8fccdbd5d307463c9fc9b9bc2bc0175d-0-351af3bd188e9234305e5514db56859e HTTP/1.1" 206 36
Downloading [configuration.json]: 100%|███████████████████████████████████████████████████████████████████████████████████████████████████| 36.0/36.0 [00:01<00:00, 27.9B/s]
Downloading [full_to_half.fst]:   0%|                                                                                                           | 0.00/17.8k [00:00<?, ?B/s]2025-08-20 14:03:33,207 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:33,659 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=full_to_half.fst HTTP/1.1" 302 342
2025-08-20 14:03:33,662 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:34,067 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/66/52/5f2871df378216c443eedc0789e1f884b3156a3b0ba086e52143788e951c?filename=full_to_half.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669813-7146b1cc3814424d9a0505ecaf642a53-0-9d340a95f8fcbf4206a5329714b4ab90 HTTP/1.1" 206 18210
Downloading [full_to_half.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████| 17.8k/17.8k [00:00<00:00, 21.1kB/s]
Downloading [README.md]:   0%|                                                                                                                  | 0.00/1.36k [00:00<?, ?B/s]2025-08-20 14:03:34,072 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:34,698 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=README.md HTTP/1.1" 200 1389
Downloading [README.md]: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████| 1.36k/1.36k [00:00<00:00, 2.21kB/s]
Downloading [remove_interjections.fst]:   0%|                                                                                                   | 0.00/11.1k [00:00<?, ?B/s]2025-08-20 14:03:34,707 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:35,122 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=remove_interjections.fst HTTP/1.1" 302 350
2025-08-20 14:03:35,125 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:36,454 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/70/f3/6aa8071968f72278fb8db9f17a060e8f8ac73a5e4161a009e33c688fedc1?filename=remove_interjections.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669815-9231278758404bc4823ed1d55b8d4daa-0-eb418da190014403b50620d0e4916cee HTTP/1.1" 206 11322
Downloading [remove_interjections.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████| 11.1k/11.1k [00:01<00:00, 6.47kB/s]
Downloading [remove_puncts.fst]:   0%|                                                                                                          | 0.00/16.3k [00:00<?, ?B/s]2025-08-20 14:03:36,459 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:36,850 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=remove_puncts.fst HTTP/1.1" 302 343
2025-08-20 14:03:36,853 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:37,190 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/0d/69/853648848ddad69e0b7a6398e5bc15be39b5559aa0119938aa6e60632538?filename=remove_puncts.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669816-0a4efd1a199e45e396bfec32a8f46f93-0-03f235f4cc77c25c3b6ad88a274875a4 HTTP/1.1" 206 16642
Downloading [remove_puncts.fst]: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████| 16.3k/16.3k [00:00<00:00, 22.7kB/s]
Downloading [tag_oov.fst]:   0%|                                                                                                                 | 0.00/458k [00:00<?, ?B/s]2025-08-20 14:03:37,194 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:37,817 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=tag_oov.fst HTTP/1.1" 302 337
2025-08-20 14:03:37,820 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:37,983 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/7d/58/b861760f691b24f095ca17dcac6ccb79de1871282847895380249f2b034b?filename=tag_oov.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669817-220b21e91c0c465895abef3187febe7c-0-ba2fae482be741084177992b4199392e HTTP/1.1" 206 468802
Downloading [tag_oov.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████| 458k/458k [00:00<00:00, 542kB/s]
Downloading [zh/tn/tagger.fst]:   0%|                                                                                                            | 0.00/793k [00:00<?, ?B/s]2025-08-20 14:03:38,065 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:38,872 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Ftn%2Ftagger.fst HTTP/1.1" 302 336
2025-08-20 14:03:38,875 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:39,340 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/63/f3/abeafe64fc3e9232099ab8ef6efd44c5198319193854af3ac2716d7c27a7?filename=tagger.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669818-117b2a09d2b74741998280b1d4867f0d-0-07e98c3df1d073c7e8ccbebd7c24b2a0 HTTP/1.1" 206 811646
Downloading [zh/tn/tagger.fst]: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████| 793k/793k [00:01<00:00, 596kB/s]
Downloading [zh/itn/tagger.fst]:   0%|                                                                                                          | 0.00/1.12M [00:00<?, ?B/s]2025-08-20 14:03:39,431 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:40,229 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Fitn%2Ftagger.fst HTTP/1.1" 302 336
2025-08-20 14:03:40,233 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:40,541 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/f1/7f/5375f77d9eef4e2a49fd86b286ce6b293b5653d61f9f74760935b726105a?filename=tagger.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669820-c790e9875fa0495788837adc95723777-0-1c504a717d4f258ffbd9b48f66436fb3 HTTP/1.1" 206 1173434
Downloading [zh/itn/tagger.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████| 1.12M/1.12M [00:01<00:00, 971kB/s]
Downloading [en/tn/tagger.fst]:   0%|                                                                                                           | 0.00/4.98M [00:00<?, ?B/s]2025-08-20 14:03:40,644 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:41,610 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=en%2Ftn%2Ftagger.fst HTTP/1.1" 302 336
2025-08-20 14:03:41,613 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:41,948 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/bb/80/4f88030153999f154f01950c18dbfd64de2c4cfc5035e21371d4eb4ead8a?filename=tagger.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669821-be102fafcee142409632f81d5881029d-0-8731ff9fb6032470c03d4a267767eaa3 HTTP/1.1" 206 5220942
Downloading [en/tn/tagger.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████| 4.98M/4.98M [00:01<00:00, 3.14MB/s]
Downloading [zh/itn/tagger_enable_0_to_9.fst]:   0%|                                                                                            | 0.00/1.13M [00:00<?, ?B/s]2025-08-20 14:03:42,320 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:42,775 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Fitn%2Ftagger_enable_0_to_9.fst HTTP/1.1" 302 350
2025-08-20 14:03:42,778 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:43,251 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/61/8c/a71a771179e665959e6306f69070b24a2ba8c20ec169d6572c8f9d0fc823?filename=tagger_enable_0_to_9.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669822-a640c774bfa146ee90f5299b1834892f-0-6387e4010bdacc4b642e08d956f8500e HTTP/1.1" 206 1188502
Downloading [zh/itn/tagger_enable_0_to_9.fst]: 100%|███████████████████████████████████████████████████████████████████████████████████| 1.13M/1.13M [00:01<00:00, 1.13MB/s]
Downloading [traditional_to_simple.fst]:   0%|                                                                                                   | 0.00/476k [00:00<?, ?B/s]2025-08-20 14:03:43,375 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:43,815 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=traditional_to_simple.fst HTTP/1.1" 302 351
2025-08-20 14:03:43,818 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:43,978 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/5b/64/2063048290dbe7dded164230e3885d418464e6987f16a1f35078a9cb6a30?filename=traditional_to_simple.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669823-d8453041d07a49fc81ea0b04c9324b5e-0-132c8e40a16d58fdc7eee9778e3e5853 HTTP/1.1" 206 487346
Downloading [traditional_to_simple.fst]: 100%|████████████████████████████████████████████████████████████████████████████████████████████| 476k/476k [00:00<00:00, 715kB/s]
Downloading [en/tn/verbalizer.fst]:   0%|                                                                                                       | 0.00/1.74M [00:00<?, ?B/s]2025-08-20 14:03:44,061 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:44,432 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=en%2Ftn%2Fverbalizer.fst HTTP/1.1" 302 340
2025-08-20 14:03:44,436 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:44,630 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/33/ad/65460a3e8f7fe313994ff0204a218c0fcb5dec9c09b24aeeaa30ef21d67b?filename=verbalizer.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669824-1d26063b2c93421aaf74c1f0e0697cb5-0-bd78564ad3acee2973d2a088b76f1e3e HTTP/1.1" 206 1822714
Downloading [en/tn/verbalizer.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████| 1.74M/1.74M [00:00<00:00, 2.56MB/s]
Downloading [zh/itn/verbalizer.fst]:   0%|                                                                                                      | 0.00/85.1k [00:00<?, ?B/s]2025-08-20 14:03:44,779 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:45,145 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Fitn%2Fverbalizer.fst HTTP/1.1" 302 340
2025-08-20 14:03:45,148 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:45,283 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/86/c3/5a5772a79bc998149e7eb0344544e52675b6fc85ad362cbad912349c017d?filename=verbalizer.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669825-57448ef8ae3f44c09cfba284e88df2b2-0-eaacd095edbfd5046baeabfbd7c556ee HTTP/1.1" 206 87098
Downloading [zh/itn/verbalizer.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████████████████| 85.1k/85.1k [00:00<00:00, 165kB/s]
Downloading [zh/tn/verbalizer.fst]:   0%|                                                                                                       | 0.00/85.6k [00:00<?, ?B/s]2025-08-20 14:03:45,312 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:45,733 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Ftn%2Fverbalizer.fst HTTP/1.1" 302 340
2025-08-20 14:03:45,736 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:45,879 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/9f/ca/dad76cafddb2e96b92d892d18249f0ba5fbe60e4d4fc51f466cb6bc7ade9?filename=verbalizer.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669825-bd68c92af9f6419b947f5e66b1afc969-0-d4284504c2abb6eb1071049fa0f80037 HTTP/1.1" 206 87702
Downloading [zh/tn/verbalizer.fst]: 100%|███████████████████████████████████████████████████████████████████████████████████████████████| 85.6k/85.6k [00:00<00:00, 148kB/s]
Downloading [zh/tn/verbalizer_remove_erhua.fst]:   0%|                                                                                          | 0.00/85.6k [00:00<?, ?B/s]2025-08-20 14:03:45,907 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:46,281 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo?Revision=master&FilePath=zh%2Ftn%2Fverbalizer_remove_erhua.fst HTTP/1.1" 302 353
2025-08-20 14:03:46,284 DEBUG Starting new HTTPS connection (1): cdn-lfs-cn-1.modelscope.cn:443
2025-08-20 14:03:46,459 DEBUG https://cdn-lfs-cn-1.modelscope.cn:443 "GET /prod/lfs-objects/f6/66/3ebe403344f9d68c4c6c64d3bf17ba755deda41d3386bc010280b96ee315?filename=verbalizer_remove_erhua.fst&namespace=pengzhendong&repository=wetext&revision=master&tag=model&auth_key=1755669826-cb783693f0a04cbc8b3bf56db6675ad2-0-071573f0a3b98dd7bc94b2b3f7dca0e3 HTTP/1.1" 206 87702
Downloading [zh/tn/verbalizer_remove_erhua.fst]: 100%|██████████████████████████████████████████████████████████████████████████████████| 85.6k/85.6k [00:00<00:00, 151kB/s]
Downloading Model to directory: /root/.cache/modelscope/hub/pengzhendong/wetext
2025-08-20 14:03:46,610 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:03:47,133 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/revisions HTTP/1.1" 200 205
2025-08-20 14:03:47,255 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo/files?Revision=master&Recursive=True HTTP/1.1" 200 None
INFO 08-20 14:03:52 [__init__.py:31] Available plugins for group vllm.general_plugins:
INFO 08-20 14:03:52 [__init__.py:33] - lora_filesystem_resolver -> vllm.plugins.lora_resolvers.filesystem_resolver:register_filesystem_resolver
INFO 08-20 14:03:52 [__init__.py:36] All plugins in this group will be loaded. Set `VLLM_PLUGINS` to control which plugins to load.
INFO 08-20 14:03:52 [config.py:793] This model supports multiple tasks: {'generate', 'classify', 'reward', 'score', 'embed'}. Defaulting to 'generate'.
WARNING 08-20 14:03:52 [arg_utils.py:1583] --enable-prompt-embeds is not supported by the V1 Engine. Falling back to V0. 
INFO 08-20 14:03:52 [llm_engine.py:230] Initializing a V0 LLM engine (v0.9.0) with config: model='pretrained_models/CosyVoice2-0.5B/vllm', speculative_config=None, tokenizer='pretrained_models/CosyVoice2-0.5B/vllm', skip_tokenizer_init=True, tokenizer_mode=auto, revision=None, override_neuron_config={}, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=32768, download_dir=None, load_format=auto, tensor_parallel_size=1, pipeline_parallel_size=1, disable_custom_all_reduce=False, quantization=None, enforce_eager=False, kv_cache_dtype=auto,  device_config=cuda, decoding_config=DecodingConfig(backend='auto', disable_fallback=False, disable_any_whitespace=False, disable_additional_properties=False, reasoning_backend=''), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None), seed=0, served_model_name=pretrained_models/CosyVoice2-0.5B/vllm, num_scheduler_steps=1, multi_step_stream_outputs=True, enable_prefix_caching=None, chunked_prefill_enabled=False, use_async_output_proc=True, pooler_config=None, compilation_config={"compile_sizes": [], "inductor_compile_config": {"enable_auto_functionalized_v2": false}, "cudagraph_capture_sizes": [256, 248, 240, 232, 224, 216, 208, 200, 192, 184, 176, 168, 160, 152, 144, 136, 128, 120, 112, 104, 96, 88, 80, 72, 64, 56, 48, 40, 32, 24, 16, 8, 4, 2, 1], "max_capture_size": 256}, use_cached_outputs=False, 
INFO 08-20 14:03:52 [cuda.py:292] Using Flash Attention backend.
[rank0]:[W820 14:03:52.981628124 ProcessGroupGloo.cpp:727] Warning: Unable to resolve hostname to a (local) address. Using the loopback address as fallback. Manually set the network interface to bind to with GLOO_SOCKET_IFNAME. (function operator())
INFO 08-20 14:03:52 [parallel_state.py:1064] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, TP rank 0, EP rank 0
INFO 08-20 14:03:52 [model_runner.py:1170] Starting to load model pretrained_models/CosyVoice2-0.5B/vllm...
Loading safetensors checkpoint shards:   0% Completed | 0/1 [00:00<?, ?it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  7.52it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  7.51it/s]

INFO 08-20 14:03:52 [default_loader.py:280] Loading weights took 0.15 seconds
INFO 08-20 14:03:53 [model_runner.py:1202] Model loading took 0.6946 GiB and 0.215803 seconds
INFO 08-20 14:03:54 [worker.py:291] Memory profiling takes 0.81 seconds
INFO 08-20 14:03:54 [worker.py:291] the current vLLM instance can use total_gpu_memory (23.53GiB) x gpu_memory_utilization (0.20) = 4.71GiB
INFO 08-20 14:03:54 [worker.py:291] model weights take 0.69GiB; non_torch_memory takes 0.02GiB; PyTorch activation peak memory takes 1.12GiB; the rest of the memory reserved for KV Cache is 2.88GiB.
INFO 08-20 14:03:54 [executor_base.py:112] # cuda blocks: 15712, # CPU blocks: 21845
INFO 08-20 14:03:54 [executor_base.py:117] Maximum concurrency for 32768 tokens per request: 7.67x
INFO 08-20 14:03:57 [model_runner.py:1512] Capturing cudagraphs for decoding. This may lead to unexpected consequences if the model is not static. To run the model in eager mode, set 'enforce_eager=True' or use '--enforce-eager' in the CLI. If out-of-memory error occurs during cudagraph capture, consider decreasing `gpu_memory_utilization` or switching to eager mode. You can also reduce the `max_num_seqs` as needed to decrease memory usage.
Capturing CUDA graph shapes: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████| 70/70 [00:46<00:00,  1.52it/s]
INFO 08-20 14:04:43 [model_runner.py:1670] Graph capturing finished in 46 secs, took 0.28 GiB
INFO 08-20 14:04:43 [llm_engine.py:428] init engine (profile, create kv cache, warmup model) took 50.04 seconds
2025-08-20 14:04:44,157 INFO Converting onnx to trt...
[08/20/2025-14:04:44] [TRT] [I] [MemUsageChange] Init CUDA: CPU +2, GPU +0, now: CPU 2521, GPU 6218 (MiB)
[08/20/2025-14:04:46] [TRT] [I] [MemUsageChange] Init builder kernel library: CPU +1773, GPU +314, now: CPU 4429, GPU 6532 (MiB)
2025-08-20 14:04:46,322 DEBUG Starting new HTTPS connection (1): stats.vllm.ai:443
2025-08-20 14:04:47,417 DEBUG https://stats.vllm.ai:443 "POST / HTTP/1.1" 200 None
[08/20/2025-14:04:47] [TRT] [I] Local timing cache in use. Profiling results in this builder pass will not be stored.
[08/20/2025-14:06:59] [TRT] [I] Detected 6 inputs and 1 output network tensors.
[08/20/2025-14:07:01] [TRT] [I] Total Host Persistent Memory: 333616
[08/20/2025-14:07:01] [TRT] [I] Total Device Persistent Memory: 2048
[08/20/2025-14:07:01] [TRT] [I] Total Scratch Memory: 321828352
[08/20/2025-14:07:01] [TRT] [I] [BlockAssignment] Started assigning block shifts. This will take 475 steps to complete.
[08/20/2025-14:07:01] [TRT] [I] [BlockAssignment] Algorithm ShiftNTopDown took 41.1229ms to assign 12 blocks to 475 nodes requiring 358718464 bytes.
[08/20/2025-14:07:01] [TRT] [I] Total Activation Memory: 358716928
[08/20/2025-14:07:01] [TRT] [I] Total Weights Memory: 142606752
[08/20/2025-14:07:01] [TRT] [I] Engine generation completed in 133.836 seconds.
[08/20/2025-14:07:01] [TRT] [I] [MemUsageStats] Peak memory usage of TRT CPU/GPU memory allocators: CPU 77 MiB, GPU 2308 MiB
[08/20/2025-14:07:01] [TRT] [I] [MemUsageStats] Peak memory usage during Engine building and serialization: CPU: 15105 MiB
2025-08-20 14:07:01,692 INFO Succesfully convert onnx to trt...
[08/20/2025-14:07:02] [TRT] [I] Loaded engine size: 159 MiB
[08/20/2025-14:07:02] [TRT] [I] [MS] Running engine with multi stream info
[08/20/2025-14:07:02] [TRT] [I] [MS] Number of aux streams is 1
[08/20/2025-14:07:02] [TRT] [I] [MS] Number of total worker streams is 2
[08/20/2025-14:07:02] [TRT] [I] [MS] The main stream provided by execute/enqueue calls is the first worker stream
[08/20/2025-14:07:03] [TRT] [I] [MemUsageChange] TensorRT-managed allocation in IExecutionContext creation: CPU +0, GPU +342, now: CPU 0, GPU 478 (MiB)
  0%|                                                                                                                                                | 0/10 [00:00<?, ?it/s2025-08-20 14:07:04,245 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
/data/workspaces/cosyvoice_test/cosyvoice/cli/model.py:104: FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  with self.llm_context, torch.cuda.amp.autocast(self.fp16 is True and hasattr(self.llm, 'vllm') is False):
WARNING 08-20 14:07:04 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:07:04 [metrics.py:486] Avg prompt throughput: 1.1 tokens/s, Avg generation throughput: 0.0 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
/data/workspaces/cosyvoice_test/cosyvoice/cli/model.py:286: FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  with torch.cuda.amp.autocast(self.fp16):
2025-08-20 14:07:06,494 INFO yield speech len 12.88, rtf 0.17468568330966167
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.73s/it]
 10%|█████████████▌                                                                                                                          | 1/10 [00:02<00:24,  2.74s/it2025-08-20 14:07:06,839 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:06 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:07:11,142 INFO yield speech len 12.64, rtf 0.3404439438747454
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:04<00:00,  4.62s/it]
 20%|███████████████████████████▏                                                                                                            | 2/10 [00:07<00:30,  3.85s/it2025-08-20 14:07:11,606 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:11 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:07:11 [metrics.py:486] Avg prompt throughput: 42.0 tokens/s, Avg generation throughput: 87.2 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
2025-08-20 14:07:15,735 INFO yield speech len 10.68, rtf 0.38666044281663076
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:04<00:00,  4.59s/it]
 30%|████████████████████████████████████████▊                                                                                               | 3/10 [00:11<00:29,  4.19s/it2025-08-20 14:07:16,044 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:16 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:07:16 [metrics.py:486] Avg prompt throughput: 30.8 tokens/s, Avg generation throughput: 81.5 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
2025-08-20 14:07:20,841 INFO yield speech len 11.72, rtf 0.4093105679079127
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.10s/it]
 40%|██████████████████████████████████████████████████████▍                                                                                 | 4/10 [00:17<00:27,  4.55s/it2025-08-20 14:07:21,245 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:21 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:07:21 [metrics.py:486] Avg prompt throughput: 30.8 tokens/s, Avg generation throughput: 49.4 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
2025-08-20 14:07:22,511 INFO yield speech len 11.64, rtf 0.10879781237992224
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.67s/it]
 50%|████████████████████████████████████████████████████████████████████                                                                    | 5/10 [00:18<00:17,  3.51s/it2025-08-20 14:07:22,798 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:22 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:07:24,053 INFO yield speech len 11.56, rtf 0.10859135112960445
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.54s/it]
 60%|█████████████████████████████████████████████████████████████████████████████████▌                                                      | 6/10 [00:20<00:11,  2.84s/it2025-08-20 14:07:24,332 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:24 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:07:25,551 INFO yield speech len 11.08, rtf 0.11006772302978736
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.50s/it]
 70%|███████████████████████████████████████████████████████████████████████████████████████████████▏                                        | 7/10 [00:21<00:07,  2.40s/it2025-08-20 14:07:25,844 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:25 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:07:26 [metrics.py:486] Avg prompt throughput: 92.4 tokens/s, Avg generation throughput: 193.7 tokens/s, Running: 1 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.1%, CPU KV cache usage: 0.0%.
2025-08-20 14:07:27,165 INFO yield speech len 12.2, rtf 0.10827744593385791
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.61s/it]
 80%|████████████████████████████████████████████████████████████████████████████████████████████████████████████▊                           | 8/10 [00:23<00:04,  2.15s/it2025-08-20 14:07:27,451 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:27 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:07:28,837 INFO yield speech len 12.56, rtf 0.11029799652707045
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.67s/it]
 90%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████▍             | 9/10 [00:25<00:02,  2.00s/it2025-08-20 14:07:29,231 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:07:29 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:07:30,530 INFO yield speech len 10.84, rtf 0.11977098084903731
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:01<00:00,  1.69s/it]
100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 10/10 [00:26<00:00,  2.67s/it]
2025-08-20 14:07:31,016 DEBUG Attempting to acquire lock 140078744562768 on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:07:31,016 DEBUG Lock 140078744562768 acquired on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:07:31,017 DEBUG Attempting to release lock 140078744562768 on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:07:31,022 DEBUG Lock 140078744562768 released on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:07:31,031 DEBUG Attempting to acquire lock 140078744562720 on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:07:31,031 DEBUG Lock 140078744562720 acquired on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:07:31,031 DEBUG Attempting to release lock 140078744562720 on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:07:31,031 DEBUG Lock 140078744562720 released on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
[rank0]:[W820 14:07:32.395730042 ProcessGroupNCCL.cpp:1476] Warning: WARNING: destroy_process_group() was not called before program exit, which can leak resources. For more info, please see https://pytorch.org/docs/stable/distributed.html#shutdown (function operator())
```

### 并发测试

#### 5 并发

测试代码（使用 vllm 加速）

```python
import sys
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
sys.path.append('third_party/Matcha-TTS')
from vllm import ModelRegistry
from cosyvoice.vllm.cosyvoice2 import CosyVoice2ForCausalLM
ModelRegistry.register_model("CosyVoice2ForCausalLM", CosyVoice2ForCausalLM)

from cosyvoice.cli.cosyvoice import CosyVoice2
from cosyvoice.utils.file_utils import load_wav
from cosyvoice.utils.common import set_all_random_seed


def init_model():
    """初始化模型（全局共享一个实例，避免重复加载）"""
    return CosyVoice2(
        'pretrained_models/CosyVoice2-0.5B',
        load_jit=True,
        load_trt=True,
        load_vllm=True,
        fp16=True
    )

def tts_task(cosyvoice, text, seed):
    """单个TTS任务函数"""
    start_time = time.time()
    prompt_speech_16k = load_wav('./asset/zero_shot_prompt.wav', 16000)
    set_all_random_seed(seed)

    # 执行TTS推理
    for _ in cosyvoice.inference_zero_shot(
        text,
        '希望你以后能够做的比我还好呦。',
        prompt_speech_16k,
        stream=False
    ):
        continue

    end_time = time.time()
    return end_time - start_time  # 返回单个任务耗时

def test_concurrency(num_threads=5, num_tasks=20):
    """测试并发性能"""
    # 初始化模型（只加载一次）
    cosyvoice = init_model()
    print(f"开始测试：{num_threads}线程，共{num_tasks}个任务")

    # 准备测试文本（可根据需要修改）
    test_text = '收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。'

    # 记录所有任务的开始时间
    total_start = time.time()

    # 使用线程池执行并发任务
    with ThreadPoolExecutor(max_workers=num_threads) as executor:
        # 提交所有任务
        futures = [
            executor.submit(tts_task, cosyvoice, test_text, i)
            for i in range(num_tasks)
        ]

        # 收集结果
        task_times = []
        for future in as_completed(futures):
            task_time = future.result()
            task_times.append(task_time)
            print(f"完成一个任务，耗时: {task_time:.2f}秒")

    # 计算统计指标
    total_time = time.time() - total_start
    avg_time = sum(task_times) / len(task_times)
    throughput = num_tasks / total_time  # 每秒处理任务数

    # 输出结果
    print("\n===== 并发测试结果 =====")
    print(f"总任务数: {num_tasks}")
    print(f"线程数: {num_threads}")
    print(f"总耗时: {total_time:.2f}秒")
    print(f"平均单任务耗时: {avg_time:.2f}秒")
    print(f"吞吐量: {throughput:.2f}任务/秒")
    print(f"最大耗时: {max(task_times):.2f}秒")
    print(f"最小耗时: {min(task_times):.2f}秒")

if __name__ == '__main__':
    # 可调整参数：线程数（并发数）和总任务数
    test_concurrency(num_threads=5, num_tasks=20)
```

```
(cosyvoice_vllm) root@11110000:/data/workspaces/cosyvoice_test# python con_vllm_example.py 
INFO 08-20 14:13:24 [__init__.py:243] Automatically detected platform cuda.
[2025-08-20 14:13:28,371] [INFO] [real_accelerator.py:203:get_accelerator] Setting ds_accelerator to cuda (auto detect)
failed to import ttsfrd, use wetext instead
Sliding Window Attention is enabled but not implemented for `sdpa`; unexpected results may be encountered.
/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/diffusers/models/lora.py:393: FutureWarning: `LoRACompatibleLinear` is deprecated and will be removed in version 1.0.0. Use of `LoRACompatibleLinear` is deprecated. Please switch to PEFT backend by installing PEFT: `pip install peft`.
  deprecate("LoRACompatibleLinear", "1.0.0", deprecation_message)
2025-08-20 14:13:33,670 INFO input frame rate=25
/root/miniconda3/envs/cosyvoice_vllm/lib/python3.10/site-packages/torch/nn/utils/weight_norm.py:143: FutureWarning: `torch.nn.utils.weight_norm` is deprecated in favor of `torch.nn.utils.parametrizations.weight_norm`.
  WeightNorm.apply(module, name, dim)
2025-08-20 14:13:35.471633131 [E:onnxruntime:Default, provider_bridge_ort.cc:1744 TryGetProviderInfo_CUDA] /onnxruntime_src/onnxruntime/core/session/provider_bridge_ort.cc:1426 onnxruntime::Provider& onnxruntime::ProviderLibrary::Get() [ONNXRuntimeError] : 1 : FAIL : Failed to load library libonnxruntime_providers_cuda.so with error: libcudnn.so.8: cannot open shared object file: No such file or directory

2025-08-20 14:13:35.471667625 [W:onnxruntime:Default, onnxruntime_pybind_state.cc:870 CreateExecutionProviderInstance] Failed to create CUDAExecutionProvider. Please reference https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#requirementsto ensure all dependencies are met.
2025-08-20 14:13:36,190 - modelscope - WARNING - Authentication has expired, please re-login with modelscope login --token "YOUR_SDK_TOKEN" if you need to access private models or datasets.
Downloading Model to directory: /root/.cache/modelscope/hub/pengzhendong/wetext
2025-08-20 14:13:36,191 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:13:36,799 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/revisions HTTP/1.1" 200 205
2025-08-20 14:13:36,901 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo/files?Revision=master&Recursive=True HTTP/1.1" 200 None
Downloading Model to directory: /root/.cache/modelscope/hub/pengzhendong/wetext
2025-08-20 14:13:37,016 DEBUG Starting new HTTPS connection (1): www.modelscope.cn:443
2025-08-20 14:13:37,621 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/revisions HTTP/1.1" 200 205
2025-08-20 14:13:37,727 DEBUG https://www.modelscope.cn:443 "GET /api/v1/models/pengzhendong/wetext/repo/files?Revision=master&Recursive=True HTTP/1.1" 200 None
INFO 08-20 14:13:41 [__init__.py:31] Available plugins for group vllm.general_plugins:
INFO 08-20 14:13:41 [__init__.py:33] - lora_filesystem_resolver -> vllm.plugins.lora_resolvers.filesystem_resolver:register_filesystem_resolver
INFO 08-20 14:13:41 [__init__.py:36] All plugins in this group will be loaded. Set `VLLM_PLUGINS` to control which plugins to load.
INFO 08-20 14:13:41 [config.py:793] This model supports multiple tasks: {'reward', 'generate', 'score', 'embed', 'classify'}. Defaulting to 'generate'.
WARNING 08-20 14:13:41 [arg_utils.py:1583] --enable-prompt-embeds is not supported by the V1 Engine. Falling back to V0. 
INFO 08-20 14:13:41 [llm_engine.py:230] Initializing a V0 LLM engine (v0.9.0) with config: model='pretrained_models/CosyVoice2-0.5B/vllm', speculative_config=None, tokenizer='pretrained_models/CosyVoice2-0.5B/vllm', skip_tokenizer_init=True, tokenizer_mode=auto, revision=None, override_neuron_config={}, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=32768, download_dir=None, load_format=auto, tensor_parallel_size=1, pipeline_parallel_size=1, disable_custom_all_reduce=False, quantization=None, enforce_eager=False, kv_cache_dtype=auto,  device_config=cuda, decoding_config=DecodingConfig(backend='auto', disable_fallback=False, disable_any_whitespace=False, disable_additional_properties=False, reasoning_backend=''), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None), seed=0, served_model_name=pretrained_models/CosyVoice2-0.5B/vllm, num_scheduler_steps=1, multi_step_stream_outputs=True, enable_prefix_caching=None, chunked_prefill_enabled=False, use_async_output_proc=True, pooler_config=None, compilation_config={"compile_sizes": [], "inductor_compile_config": {"enable_auto_functionalized_v2": false}, "cudagraph_capture_sizes": [256, 248, 240, 232, 224, 216, 208, 200, 192, 184, 176, 168, 160, 152, 144, 136, 128, 120, 112, 104, 96, 88, 80, 72, 64, 56, 48, 40, 32, 24, 16, 8, 4, 2, 1], "max_capture_size": 256}, use_cached_outputs=False, 
INFO 08-20 14:13:41 [cuda.py:292] Using Flash Attention backend.
[rank0]:[W820 14:13:42.318048057 ProcessGroupGloo.cpp:727] Warning: Unable to resolve hostname to a (local) address. Using the loopback address as fallback. Manually set the network interface to bind to with GLOO_SOCKET_IFNAME. (function operator())
INFO 08-20 14:13:42 [parallel_state.py:1064] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, TP rank 0, EP rank 0
INFO 08-20 14:13:42 [model_runner.py:1170] Starting to load model pretrained_models/CosyVoice2-0.5B/vllm...
Loading safetensors checkpoint shards:   0% Completed | 0/1 [00:00<?, ?it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  5.44it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:00<00:00,  5.43it/s]

INFO 08-20 14:13:42 [default_loader.py:280] Loading weights took 0.25 seconds
INFO 08-20 14:13:42 [model_runner.py:1202] Model loading took 0.6951 GiB and 0.339293 seconds
INFO 08-20 14:13:43 [worker.py:291] Memory profiling takes 0.73 seconds
INFO 08-20 14:13:43 [worker.py:291] the current vLLM instance can use total_gpu_memory (23.53GiB) x gpu_memory_utilization (0.20) = 4.71GiB
INFO 08-20 14:13:43 [worker.py:291] model weights take 0.70GiB; non_torch_memory takes 0.07GiB; PyTorch activation peak memory takes 1.12GiB; the rest of the memory reserved for KV Cache is 2.82GiB.
INFO 08-20 14:13:43 [executor_base.py:112] # cuda blocks: 15411, # CPU blocks: 21845
INFO 08-20 14:13:43 [executor_base.py:117] Maximum concurrency for 32768 tokens per request: 7.52x
INFO 08-20 14:13:46 [model_runner.py:1512] Capturing cudagraphs for decoding. This may lead to unexpected consequences if the model is not static. To run the model in eager mode, set 'enforce_eager=True' or use '--enforce-eager' in the CLI. If out-of-memory error occurs during cudagraph capture, consider decreasing `gpu_memory_utilization` or switching to eager mode. You can also reduce the `max_num_seqs` as needed to decrease memory usage.
Capturing CUDA graph shapes: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████| 70/70 [00:50<00:00,  1.40it/s]
INFO 08-20 14:14:36 [model_runner.py:1670] Graph capturing finished in 50 secs, took 0.28 GiB
INFO 08-20 14:14:36 [llm_engine.py:428] init engine (profile, create kv cache, warmup model) took 54.22 seconds
2025-08-20 14:14:38,202 DEBUG Starting new HTTPS connection (1): stats.vllm.ai:443
[08/20/2025-14:14:39] [TRT] [I] Loaded engine size: 159 MiB
[08/20/2025-14:14:39] [TRT] [I] [MS] Running engine with multi stream info
[08/20/2025-14:14:39] [TRT] [I] [MS] Number of aux streams is 1
[08/20/2025-14:14:39] [TRT] [I] [MS] Number of total worker streams is 2
[08/20/2025-14:14:39] [TRT] [I] [MS] The main stream provided by execute/enqueue calls is the first worker stream
2025-08-20 14:14:39,453 DEBUG https://stats.vllm.ai:443 "POST / HTTP/1.1" 200 None
[08/20/2025-14:14:42] [TRT] [I] [MemUsageChange] TensorRT-managed allocation in IExecutionContext creation: CPU +0, GPU +342, now: CPU 0, GPU 478 (MiB)
开始测试：5线程，共20个任务
  0%|                                                                                                                                                 | 0/1 [00:00<?, ?it/s2025-08-20 14:14:43,126 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
2025-08-20 14:14:43,128 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。      | 0/1 [00:00<?, ?it/s]
/data/workspaces/cosyvoice_test/cosyvoice/cli/model.py:104: FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.                                                                                                                                            | 0/1 [00:00<?, ?it/s]
  with self.llm_context, torch.cuda.amp.autocast(self.fp16 is True and hasattr(self.llm, 'vllm') is False):
2025-08-20 14:14:43,128 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
2025-08-20 14:14:43,128 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
2025-08-20 14:14:43,128 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:43 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:43 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:43 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:14:43 [metrics.py:486] Avg prompt throughput: 61.3 tokens/s, Avg generation throughput: 0.4 tokens/s, Running: 3 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.2%, CPU KV cache usage: 0.0%.
WARNING 08-20 14:14:43 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:43 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
/data/workspaces/cosyvoice_test/cosyvoice/cli/model.py:286: FutureWarning: `torch.cuda.amp.autocast(args...)` is deprecated. Please use `torch.amp.autocast('cuda', args...)` instead.
  with torch.cuda.amp.autocast(self.fp16):
2025-08-20 14:14:47,450 INFO yield speech len 14.2, rtf 0.30437906023482203
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.12s/it]
完成一个任务，耗时: 5.14秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.12s/it]
                                                                                                                                                                           2025-08-20 14:14:47,481 INFO yield speech len 14.6, rtf 0.29826087494419046                                                                            | 0/1 [00:00<?, ?it/s]
2025-08-20 14:14:47,482 INFO yield speech len 13.2, rtf 0.3298765059673425
2025-08-20 14:14:47,483 INFO yield speech len 11.24, rtf 0.3874462482343789
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.13s/it]
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.13s/it]
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.13s/it]
完成一个任务，耗时: 5.15秒
完成一个任务，耗时: 5.16秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.13s/it]
完成一个任务，耗时: 5.15秒
  0%|                                                                                                                                                 | 0/1 [00:00<?, ?it/s2025-08-20 14:14:47,496 INFO yield speech len 12.64, rtf 0.3455672083021719
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:05<00:00,  5.14s/it]
完成一个任务，耗时: 5.16秒
  0%|                                                                                                                                                 | 0/1 [00:00<?, ?it/s2025-08-20 14:14:47,923 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。 1/1 [00:05<00:00,  5.14s/it]
2025-08-20 14:14:47,923 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
2025-08-20 14:14:47,923 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。      | 0/1 [00:00<?, ?it/s]
2025-08-20 14:14:47,923 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
2025-08-20 14:14:47,926 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:47 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:47 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:47 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:47 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:47 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
INFO 08-20 14:14:48 [metrics.py:486] Avg prompt throughput: 246.3 tokens/s, Avg generation throughput: 390.8 tokens/s, Running: 5 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.5%, CPU KV cache usage: 0.0%.
2025-08-20 14:14:50,199 INFO yield speech len 12.12, rtf 0.18785070664811843
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.71s/it]
完成一个任务，耗时: 2.72秒
                                                                                                                                                                           2025-08-20 14:14:50,487 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:50 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized████████████████████████████████████| 1/1 [00:02<00:00,  2.71s/it]
2025-08-20 14:14:50,732 INFO yield speech len 13.16, rtf 0.21350191719263883
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.25s/it]
完成一个任务，耗时: 3.25秒
  0%|                                                                                                                                                 | 0/1 [00:00<?, ?it/s]2025-08-20 14:14:50,756 INFO yield speech len 13.28, rtf 0.21337017596486105
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.26s/it]
完成一个任务，耗时: 3.26秒
                                                                                                                                                                           2025-08-20 14:14:50,780 INFO yield speech len 12.36, rtf 0.23084718818417646███████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.26s/it]
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.29s/it]
完成一个任务，耗时: 3.29秒                                                                                                                            | 0/1 [00:00<?, ?it/s]
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.29s/it2025-08-20 14:14:50,808 INFO yield speech len 12.76, rtf 0.2260991585292039
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.33s/it]
完成一个任务，耗时: 3.33秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:03<00:00,  3.33s/it]
                                                                                                                                                                           2025-08-20 14:14:51,023 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。       | 0/1 [00:00<?, ?it/s]
WARNING 08-20 14:14:51 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:51,044 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:51 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:51,064 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:51 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:51,122 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:51 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:52,509 INFO yield speech len 12.16, rtf 0.16622776655774368
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.30s/it]
完成一个任务，耗时: 2.31秒
                                                                                                                                                                           2025-08-20 14:14:52,808 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:52 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized████████████████████████████████████| 1/1 [00:02<00:00,  2.30s/it]
2025-08-20 14:14:52,857 INFO yield speech len 11.8, rtf 0.1553733065976935
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.12s/it]
完成一个任务，耗时: 2.12秒
  0%|                                                                                                                                                 | 0/1 [00:00<?, ?it/s]2025-08-20 14:14:53,139 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:53 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:53,255 INFO yield speech len 11.88, rtf 0.17954325836515586
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.45s/it]
完成一个任务，耗时: 2.45秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.45s/it]
                                                                                                                                                                           2025-08-20 14:14:53,275 INFO yield speech len 12.24, rtf 0.18063842081556133                                                                           | 0/1 [00:00<?, ?it/s]
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.49s/it]
完成一个任务，耗时: 2.49秒
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.49s/it2025-08-20 14:14:53,292 INFO yield speech len 12.44, rtf 0.18074152170653512
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.53s/it]
完成一个任务，耗时: 2.53秒
                                                                                                                                                                           INFO 08-20 14:14:53 [metrics.py:486] Avg prompt throughput: 215.4 tokens/s, Avg generation throughput: 580.4 tokens/s, Running: 2 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 0.2%, CPU KV cache usage: 0.0%.
2025-08-20 14:14:53,579 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。      | 0/1 [00:00<?, ?it/s]
2025-08-20 14:14:53,580 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:53 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
WARNING 08-20 14:14:53 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:53,636 INFO synthesis text 收到好友从远方寄来的生日礼物，那份意外的惊喜与深深的祝福让我心中充满了甜蜜的快乐，笑容如花儿般绽放。
WARNING 08-20 14:14:53 [preprocess.py:63] Using None for EOS token id because tokenizer is not initialized
2025-08-20 14:14:54,657 INFO yield speech len 11.92, rtf 0.1551069269244303
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.15s/it]
完成一个任务，耗时: 2.15秒
2025-08-20 14:14:55,104 INFO yield speech len 12.68, rtf 0.15493849474548918
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.25s/it]
完成一个任务，耗时: 2.25秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.15s/it]
2025-08-20 14:14:55,649 INFO yield speech len 11.72, rtf 0.17174832243154478
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.35s/it]
完成一个任务，耗时: 2.36秒
2025-08-20 14:14:55,692 INFO yield speech len 12.04, rtf 0.1753354389406122
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.43s/it]
完成一个任务，耗时: 2.43秒████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.43s/it]
2025-08-20 14:14:55,710 INFO yield speech len 12.48, rtf 0.17072138113853258
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.43s/it]
完成一个任务，耗时: 2.43秒
100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 1/1 [00:02<00:00,  2.43s/it]
===== 并发测试结果 =====
总任务数: 20
线程数: 5
总耗时: 13.39秒
平均单任务耗时: 3.26秒
吞吐量: 1.49任务/秒
最大耗时: 5.16秒
最小耗时: 2.12秒
2025-08-20 14:14:56,105 DEBUG Attempting to acquire lock 140521691385136 on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:14:56,105 DEBUG Lock 140521691385136 acquired on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:14:56,106 DEBUG Attempting to release lock 140521691385136 on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:14:56,106 DEBUG Lock 140521691385136 released on /root/.triton/autotune/Fp16Matmul_2d_kernel.pickle.lock
2025-08-20 14:14:56,114 DEBUG Attempting to acquire lock 140521691385184 on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:14:56,115 DEBUG Lock 140521691385184 acquired on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:14:56,115 DEBUG Attempting to release lock 140521691385184 on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
2025-08-20 14:14:56,115 DEBUG Lock 140521691385184 released on /root/.triton/autotune/Fp16Matmul_4d_kernel.pickle.lock
[rank0]:[W820 14:14:56.214318047 ProcessGroupNCCL.cpp:1476] Warning: WARNING: destroy_process_group() was not called before program exit, which can leak resources. For more info, please see https://pytorch.org/docs/stable/distributed.html#shutdown (function operator())
```

### 10 并发

