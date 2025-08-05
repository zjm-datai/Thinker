## 项目目标

使用 langggraph 框架，使用下述基类，实现抽象方法并适当扩展，得到满足业务要求的问诊智能体。

### 上下文资料

需要继承的 langgraph 的基类是：

```python
"""This file contains the LangGraph Agent/workflow and interactions with the LLMs."""

import logging
from abc import ABC, abstractmethod
from typing import Any, Optional, Dict

from langchain_openai import ChatOpenAI
from langgraph.graph.state import CompiledStateGraph
from psycopg_pool import AsyncConnectionPool

from core.agents.tools import tools
from configs import config

logger = logging.getLogger(__name__)

class GraphAgent(ABC):
    """Mananges the LangGraph Agent/Workflow and interactions with the LLM.
    
    This class handles the creation and management of the LangGraph workflow,
    including LLM interactions, database connections, and response processing.
    """
    
    def __init__(self):
        """Initialize the langgraph agent with necessary components."""

        # use environment-specific LLM model
        self.llm = ChatOpenAI(
            model=config.LLM_MODEL,
            temperature=config.DEFAULT_LLM_TEMPERATURE,
            api_key=config.LLM_API_KEY,
            base_url=config.LLM_BASE_URL,
            max_tokens=config.MAX_TOKENS,
            # **self._get_model_kwargs(),
        ).bind_tools(tools)

        self.tools_by_name = {
            tool.name for tool in tools
        }
        self._connection_pool: Optional[AsyncConnectionPool] = None
        self._graph: Optional[CompiledStateGraph] = None

        logger.info(f"llm_initialized - model= {config.LLM_MODEL}")

    def _get_model_kwargs(self) -> Dict[str, Any]:
        pass

    @abstractmethod
    async def create_graph(self) -> Any:
        """Construct and compile the workflow graph."""

        raise NotImplementedError("Subclasses must implement create_graph method")
    
    @abstractmethod
    def get_response():
        pass

    @abstractmethod
    def get_stream_response():
        pass
```

实现样例：

```python

# digital_person/backend/core/agents/normal_agent.py


import logging
import re
from typing import Annotated, List, Literal, Optional, AsyncGenerator

from langchain_core.language_models.chat_models import BaseChatModel
from langchain_core.messages import trim_messages as _trim_message
from langchain_core.messages import (
    BaseMessage,
    ToolMessage,
    AIMessage,
    convert_to_openai_messages
)
from langgraph.graph.message import add_messages
from langgraph.graph.state import CompiledStateGraph, StateGraph
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
from langgraph.graph import END
from openai import OpenAIError

from psycopg_pool import AsyncConnectionPool
from pydantic import Field, field_validator, BaseModel

from core.agents.graph_agent_base import GraphAgent
from core.metrics import llm_inference_duration_seconds
from core.prompts import SYSTEM_PROMPT

from configs import config, Environment
from schemas.chat import Message

logger = logging.getLogger(__name__)

def dump_messages(messages: list[Message]) -> list[dict]:
    """Dump the messages to a list of dictionaries.

    Args:
        messages (list[Message]): The messages to dump.

    Returns:
        list[dict]: The dumped messages.
    """
    return [message.model_dump() for message in messages]

def prepare_message(messages: List[Message], llm: BaseChatModel, system_prompt: str) -> List[dict]:
    """Prepare the messages for the LLM."""

    # 对于 trim_message 这个方法来说要么给他 langchain 官方的消息类型的列表
    # 要么给它字典列表，但是每个字典要有相应的键，role content 都要有
    # trimmed = trim_messages(
    #     [
    #         {"role": "system", "content": "你好"},
    #         {"role": "user", "content": "今天天气怎样?"}
    #     ],
    #     token_counter=...,
    #     max_tokens=100
    # )

    # from transformers import AutoTokenizer

    # tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen1.5-30B")

    # def count_tokens(msgs: list[dict]) -> int:
    #     text = "".join([m["content"] for m in msgs if "content" in m])
    #     return len(tokenizer.encode(text))

    def mock_token_counter(msgs: list[dict]) -> int:
        # 粗略估算：每 4 个字符 = 1 token（英文）或 每 1 个汉字 = 1 token
        total_chars = sum(len(m["content"]) for m in msgs if "content" in m)
        return total_chars  # 或者 total_chars // 2 做个近似


    trimmed_messages = _trim_message(
        dump_messages(messages),
        strategy="last",
        token_counter=mock_token_counter,
        max_tokens=config.MAX_TOKENS,
        start_on="human",
        include_system=False,
        allow_partial=False
    )

    return [Message(role="system", content=system_prompt)] + trimmed_messages

class GraphState(BaseModel):
    """state definition for the langgraph agent/worlflow."""

    messages: Annotated[list, add_messages] = Field(
        default_factory=list, description="The messages in the conversation"
    )
    session_id: str = Field(..., description="The unique identifier for the conversation session")

    @field_validator("session_id")
    @classmethod
    def validate_session_id(cls, v: str) -> str:
        """Validate that the session ID is valid UUID or follows safe pattern.
        
        Args:
            v: The thread ID to validate

        Returns:
            str: The validated session ID

        Raises:
            ValueError: If the session ID is not valid
        """
        try:
            pass
        except ValueError:
            # If not a UUID, check for safe characters only
            if not re.match(r"^[a-zA-Z0-9_\-]+$", v):
                raise ValueError("Session ID must contain only alphanumeric characters, underscores, and hyphens")
            return v

class NormalAgent(GraphAgent):
    """This is simple implement of base agent."""

    def __init__(self):
        super().__init__()

    async def create_graph(self) -> Optional[CompiledStateGraph]:
        """Create and configure the LangGraph workflow."""

        if self._graph is None:
            try:

                graph_builder = StateGraph(GraphState)
                graph_builder.add_node("chat", self._chat)
                graph_builder.add_node("tool_call", self._tool_call)
                graph_builder.add_conditional_edges(
                    "chat",
                    self._should_continue,
                    {
                        "continue": "tool_call",
                        "end": END
                    }
                )
                graph_builder.add_edge("tool_call", "chat")
                graph_builder.set_entry_point("chat")
                graph_builder.set_finish_point("chat")

                # Get connection pool (may be None in production if DB unavailable)
                connection_pool = await self._get_connection_pool()
                if connection_pool:
                    checkpointer = AsyncPostgresSaver(connection_pool)
                    await checkpointer.setup()
                else:
                    # In production, proceed without checkpointer if needed
                    checkpointer = None
                    if config.ENVIRONMENT != Environment.PRODUCTION:
                        raise Exception("Connection pool initialization failed!")
                    
                self._graph = graph_builder.compile(
                    checkpointer=checkpointer, name=f"normal-agent-{config.ENVIRONMENT.value}"
                )

            except Exception as e:
                logger.error(f"graph_creation_failed - error={str(e)} - environment={config.ENVIRONMENT.value}")
                # In production, we don't want to crash the app
                if config.ENVIRONMENT == Environment.PRODUCTION:
                    logger.warning("continuing_without_graph")
                    return None
                raise e
            
    async def _chat(self, state: GraphState):
        """Process the chat state and generate a response."""

        messages = prepare_message(state.messages, self.llm, SYSTEM_PROMPT)
        llm_calls_num = 0

        # Configure retry attempts based on env
        max_retries = config.MAX_LLM_CALL_RETRIES

        for attempt in range(max_retries):
            try:
                with llm_inference_duration_seconds.labels(model=self.llm.model_name).time():
                    generated_state = {
                        "messages": [await self.llm.ainvoke(dump_messages(messages))]
                    }

                return generated_state
            except OpenAIError as e:

                llm_calls_num += 1
                continue

        raise Exception(f"Failed to get a response from the LLM after {max_retries} attempts")
        
    # tool node
    async def _tool_call(self, state: GraphState) -> GraphState:
        """Process tool calls from the last message."""

        outputs = []
        for tool_call in state.messages[-1].tool_calls:
            tool_result = await self.tools_by_name[tool_call["name"]].ainvoke(tool_call["args"])
            outputs.append(
                ToolMessage(
                    content=tool_result,
                    name=tool_call["name"],
                    tool_call_id=tool_call["id"]
                )
            )
        
        return {
            "messages": outputs
        }

    def _should_continue(self, state: GraphState) -> Literal["end", "continue"]:
        """Determine if the agent should continue or end based on the last message."""

        messages = state.messages
        last_message = messages[-1]

        if isinstance(last_message, AIMessage):
            if getattr(last_message, "tool_calls", None):
                return "continue"

        return "end"
        
    async def _get_connection_pool(self):
        
        if self._connection_pool is None:
            try:
                # Configure pool size based on environment
                max_size = config.DATABASE_POOL_SIZE

                self._connection_pool = AsyncConnectionPool(
                    config.DATABASE_URL,
                    open=False,
                    max_size=max_size,
                    kwargs={
                        "autocommit": True,
                        "connect_timeout": 60,
                        "prepare_threshold": None,
                    },
                )
                await self._connection_pool.open()
            except Exception as e:
                
                # In production, we might want to degrade gracefully
                if config.ENVIRONMENT == Environment.PRODUCTION:
                    return None
                raise e
        return self._connection_pool

    def __process_messages(self, messages: list[BaseMessage]) -> list[Message]:
        openai_style_messages = convert_to_openai_messages(messages)
        # keep just assistant and user messages
        return [
            Message(**message)
            for message in openai_style_messages
            if message["role"] in ["assistant", "user"] and message["content"]
        ]

    async def get_response(
        self,
        messages: list[Message],
        session_id: str,
        user_id: Optional[str] = None
    ) -> list[dict]:
        """Get a response from the LLM.
        
        Args:
            messages (list[Message]): The messages to send to the LLM.
            session_id (str): The session ID for Langfuse tracking.
            user_id (Optional[str]): The user ID for Langfuse tracking.

        Returns:
            list[dict]: The response from the LLM.
        """
        if self._graph is None:
            await self.create_graph()

        graph_config = {
            "configurable": {"thread_id": session_id}
        }
        try:
            response = await self._graph.ainvoke(
                {
                    "messages": dump_messages(messages),
                    "session_id": session_id
                },
                graph_config
            )
            return self.__process_messages(response["messages"])
        except Exception as e:
            logger.error(f"Error getting response: {str(e)}")
            raise e
        
    async def get_stream_response(
        self, messages: list[Message], session_id: str, user_id: Optional[str] = None
    ) -> AsyncGenerator[str, None]:
        """Get a stream response from the LLM.

        Args:
            messages (list[Message]): The messages to send to the LLM.
            session_id (str): The session ID for the conversation.
            user_id (Optional[str]): The user ID for the conversation.

        Yields:
            str: Tokens of the LLM response.
        """
        config = {
            "configurable": {"thread_id": session_id},
        }
        if self._graph is None:
            await self.create_graph()

        try:
            async for token, _ in self._graph.astream(
                {"messages": dump_messages(messages), "session_id": session_id}, config, stream_mode="messages"
            ):
                try:
                    yield token.content
                except Exception as token_error:
                    logger.error(f"Error processing token - error={str(token_error)} - session_id={session_id}")
                    # Continue with next token even if current one fails
                    continue
        except Exception as stream_error:
            logger.error(f"Error in stream processing - error={str(stream_error)} - session_id={session_id}")
            raise stream_error
```

## 业务需求

我们要实现一个问诊机器人，他可以针对患者传入的基本信息，动态生成问题集进行提问，并根据患者的回答状态进行阶段处理。

### 病历实体抽取

首先其目标是对用户的基本信息（格式如下）：

```
{
	"patientIdentity": {
		"patientId": "PID1234562",
		"idType": "01",
		"idNumber": "110101199001011234",
		"patientName": "王女士"
	},
	"visitInfo": {
		"department": "消化内科",
		"visitNumber": "V0000123"
	},
	"latestMedicalRecord": {
		"basicInfo": {
			"gender": "女",
			"birthday": "1985-05-20",
			"aboBloodType": "A",
			"rhBloodType": "+"
		},
		"vitalSigns": {
			"systolicPressure": 115,
			"diastolicPressure": 75,
			"height": 165,
			"weight": 61
		},
		"marriageChildInfo": {
			"marriageStatus": "已婚",
			"fullTermCount": 1,
			"prematureCount": 0,
			"abortionCount": 0,
			"livingChildrenCount": 1
		},
		"pastHistory": {
			"personalHistory": "无特殊",
			"bloodTransfusionHistory": "无",
			"diseaseHistory": "高血压",
			"epidemiologicalHistory": "无",
			"menstrualHistory": {
				"menarcheAge": 13,
				"intervalDays": 28,
				"durationDays": 5,
				"isSterilization": False,
				"lastMenstrualDate": "2024-09-25"
			},
			"surgeryHistory": "阑尾炎手术史",
			"familyHistory": "父母健康"
		},
		"allergyHistory": "青霉素过敏",
		"childGrowthInfo": None
	},
	"revisitInfo": {
		"isRevisit": 1,
		"lastRecord": {
			"chiefComplaint": "腹痛难耐，同时恶心呕吐，持续三天",
			"presentIllness": (
				"目前接受抗生素治疗，疼痛中度，近一周内症状加重，"
				"食欲减退，偏好清淡食物，夜间睡眠质量差，"
				"大便次数增多但不成形，小便正常"
			),
			"menstrualHistory": {
				"menarcheAge": 13,
				"intervalDays": 28,
				"durationDays": 5,
				"isSterilization": False,
				"lastMenstrualDate": "2024-09-25"
			},
			"tcmFourExams": {
				"inspection": "面色苍白",
				"inquiry": "有胃脘隐痛史",
				"listeningAndSmelling": "口气稍重",
				"palpation": "脉沉细"
			},
			"physicalExam": "心肺听诊正常",
			"auxiliaryExam": "无异常",
			"diagnosis": {
				"tcmDiagnosisName": "头痛",
				"tcmDiagnosisCode": "TCM-001",
				"tcmSyndromeName": "肝阳上亢",
				"westernDiagnosisName": "神经性头痛",
				"westernDiagnosisCode": "G44.2"
			},
			"treatmentPrinciple": "平肝潜阳，活血通络",
			"treatmentAdvice": "注意休息，避免劳累",
			"prescription": {
				"prescriptionName": "平肝潜阳汤",
				"herbs": ["天麻", "钩藤", "栀子"]
			}
		}
	}
}
```

解析出来基本的信息，然后维护三张字段表如下：（相当于实体抽取上述信息进入下面这些字段中，如果有就填入，没有就置空）

condition_fields

```
[{"field_content":null,"field_name_cn":"基本症状","field_name_eg":"condition"},{"field_content":null,"field_name_cn":"症状持续时间","field_name_eg":"condition_duration"},{"field_content":null,"field_name_cn":"症状部位","field_name_eg":"condition_location"},{"field_content":null,"field_name_cn":"症状严重程度","field_name_eg":"condition_severity"},{"field_content":null,"field_name_cn":"伴随症状","field_name_eg":"associated_symptoms"},{"field_content":null,"field_name_cn":"有无就诊治疗过","field_name_eg":"treatment_visited"},{"field_content":null,"field_name_cn":"就诊诊断","field_name_eg":"treatment_diagnosis"},{"field_content":null,"field_name_cn":"有无自行使用过药物","field_name_eg":"treatment_self_medication"},{"field_content":null,"field_name_cn":"药物名称","field_name_eg":"medication_name"}]
```

history_fields

```
[{"field_content":null,"field_name_cn":"有无过敏食物或药物","field_name_eg":"allergy_present"},{"field_content":null,"field_name_cn":"过敏食物或药物名称","field_name_eg":"allergy_foodordrug_name"},{"field_content":null,"field_name_cn":"有无长期用药史","field_name_eg":"long_term_medication_present"},{"field_content":null,"field_name_cn":"长期用药名称","field_name_eg":"long_term_medication_name"}]
```

personal_fields

```
[{"field_content":null,"field_name_cn":"是否有不良生活习惯","field_name_eg":"personal_bad_habits"},{"field_content":null,"field_name_cn":"抽烟频率","field_name_eg":"personal_smoking_frequency"},{"field_content":null,"field_name_cn":"喝酒频率","field_name_eg":"personal_drinking_frequency"},{"field_content":null,"field_name_cn":"饮食与口味状况","field_name_eg":"dietary_status"},{"field_content":null,"field_name_cn":"睡眠状况","field_name_eg":"sleep_status"},{"field_content":null,"field_name_cn":"大便状况","field_name_eg":"bowel_movement"},{"field_content":null,"field_name_cn":"小便状况","field_name_eg":"urine_status"},{"field_content":null,"field_name_cn":"月经周期","field_name_eg":"menstrual_cycle"},{"field_content":null,"field_name_cn":"经期天数","field_name_eg":"menstrual_duration"},{"field_content":null,"field_name_cn":"末次月经时间","field_name_eg":"last_menstrual_period"},{"field_content":null,"field_name_cn":"经量","field_name_eg":"menstrual_flow"},{"field_content":null,"field_name_cn":"经色","field_name_eg":"menstrual_color"},{"field_content":null,"field_name_cn":"经质","field_name_eg":"menstrual_quality"},{"field_content":null,"field_name_cn":"婚育史（18周岁以上）","field_name_eg":"marital_reproductive_history"},{"field_content":null,"field_name_cn":"足月生产个数","field_name_eg":"full_term_birth_count"},{"field_content":null,"field_name_cn":"早产个数","field_name_eg":"preterm_birth_count"},{"field_content":null,"field_name_cn":"流产个数","field_name_eg":"miscarriage_count"},{"field_content":null,"field_name_cn":"现存子女数量","field_name_eg":"living_children_count"},{"field_content":null,"field_name_cn":"已育子女数量","field_name_eg":"children_count"}]
```

### 按字段提问

- **遍历字段表**，依次对每个字段检查是否已有值
    
- 若字段值为空，调用大模型生成「针对该字段」的单一封闭或开放式问题
    
- **用户回答后**：
    
    - 验证格式或语义是否符合预期（例如：时间型字段需含日期/时长）
        
    - **符合**：填充字段，进入下一字段
        
    - **不符合**：继续提问，最多 **3 轮**
        
- 若三轮仍不符合，则记录该字段为 “未获取” 并跳过
    
- **所有字段完成后**：调用大模型生成**结构化病历总结**

## 注意事项

问题需简洁明了，避免歧义