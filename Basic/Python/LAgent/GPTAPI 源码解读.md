## 文件开头和导入




## 同步版 GPTAPI

### 类定义和文档

```python
class GPTAPI(BaseAPILLM):
	"""Model wrapper around OpenAI's models.

    Args:
        model_type (str): The name of OpenAI's model.
        retry (int): Number of retries if the API call fails. Defaults to 2.
        key (str or List[str]): OpenAI key(s)...
        org (str or List[str], optional): ...
        meta_template (Dict, optional): The model's meta prompt...
        api_base (str): ...
        gen_params: Default generation configuration...
    """
    is_api: bool = True
```

这里的 BaseAPILLM 只是多加了一些初始化参数，然后继承自 BaseLLM，这个类才是老大，定义了很多基本的方法，要是没有实现这些方法会 raise NotImplementedError 。

有趣的是这里的 is_api 这个类属性，估计是为了和本地部署大模型类进行一个区分吧。

### 构造函数 






























