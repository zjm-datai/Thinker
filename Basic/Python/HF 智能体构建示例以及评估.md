
## 构建 Guestbook Tool

### Create  the Retriever Tool

```python
from langchain_community.retrievers import BM25Retriver
from langchain.tools import Tool

bm25_retriever = BM25Retriver.from_documents(docs)

def extract_text(query: str) -> str:
	"""Retrieves detailed information about gala guests based on their name or relation."""
	results = bm25_retriever.invoke(query)
	if results:
        return "\n\n".join([doc.page_content for doc in results[:3]])
    else:
        return "No matching guest information found."

guest_info_tool = Tool(
    name="guest_info_retriever",
    func=extract_text,
    description="Retrieves detailed information about gala guests based on their name or relation."
)
```

We’re using a BM25Retriever, which is a powerful text retrieval algorithm that doesn’t require embeddings

# Build and interating Tools for Your Agent

## Give Your Agent Access to the Web

我们希望阿尔弗雷德展现出自己作为一名真正的博学多才的主持人的风采，对世界有着深刻的认知。To do so, we need to make sure that Alfred has access to the latest news and information about the world.

Let’s start by creating a web search tool for Alfred!

```python
from langchain_community.tools import DuckDuckGoSearchRun

search_tool = DuckDuckGoSearchRun()
result = search_tool.invoke("Who is the current President of France?")
print(result)
```

