

light-agent 

首先我们复现 base-action 模型

然后探究如何将这些 base-action 注入到大模型提示词中（疑似是通过 toolparser ，然后会被用于聚合器，聚合器会安排历史记录和提示词，非常关键）

探究完之后，我们需要做了一个简单的 agent （使用 api 好像 lagent 中有 GPTAPI 类），然后注入一个 websearch （这里怎么使用 api 还是很关键的），然后测试一下。

上面都搞同步的罗

后面深入了我们才搞异步和流式。