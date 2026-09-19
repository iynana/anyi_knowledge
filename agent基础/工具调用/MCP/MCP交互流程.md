+ MCP Client（第一次连接MCP Server需要进行 Tool Discovery）（嵌入大模型宿主）接收用户请求后，按 MCP 格式打包 “问题 + 工具列表” 发给大模型；
+ 大模型按 MCP 规范生成工具调用指令（格式全一致，不再区分 OpenAI/Claude）；
+ MCP Client 解析指令，调用 MCP Server 上的工具；
+ 工具执行结果按 MCP 标准格式返回给大模型，生成最终回答。