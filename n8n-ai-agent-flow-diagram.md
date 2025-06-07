# n8n AI Agent Node - Flow Analysis with Code References

## Complete Process Flow

```mermaid
graph TD
    A[User Question] --> B[AI Agent Node]
    B --> C[Validate Input]
    C --> D[Select Tools Agent]
    D --> E[Initialize Components]
    
    E --> F[Get Chat Model]
    E --> G[Get Memory]
    E --> H[Get Output Parser]
    
    F --> I[Discover Tools]
    G --> I
    H --> I
    
    I --> J[N8n Tools]
    I --> K[MCP Tools]
    
    J --> L[Convert to Dynamic Tools]
    K --> M[Connect via SSE]
    M --> N[List MCP Tools]
    N --> O[Convert MCP to Dynamic Tools]
    
    L --> P[Validate Tool Names]
    O --> P
    
    P --> Q[Create System Prompt]
    Q --> R[Embed Tool Descriptions]
    R --> S[Add Chat History]
    S --> T[Create Agent]
    
    T --> U[Send Question to LLM]
    U --> V{LLM Response}
    
    V -->|Tool Call Needed| W[Parse Tool Request]
    V -->|Final Answer| Z[Process Output]
    
    W --> X{Tool Type}
    X -->|N8n Tool| Y1[Execute N8n Tool]
    X -->|MCP Tool| Y2[Execute MCP Tool]
    
    Y1 --> AA[Tool Result]
    Y2 --> AA
    
    AA --> BB[Send Result to LLM]
    BB --> V
    
    Z --> CC{Output Parser?}
    CC -->|Yes| DD[Format Structured Output]
    CC -->|No| EE[Clean Response]
    
    DD --> FF[Save to Memory]
    EE --> FF
    
    FF --> GG[Return Final Answer]
    GG --> HH[User Receives Response]

    %% Internal links to code sections
    click B "#main-agent-node" "Main Agent Node Implementation"
    click D "#tools-agent-executor" "Tools Agent Executor"
    click F "#get-chat-model" "Get Chat Model Function"
    click G "#get-optional-memory" "Get Optional Memory Function"
    click H "#get-output-parser" "Get Output Parser Function"
    click I "#discover-tools" "Discover Tools Function"
    click J "#n8n-tools" "N8n Tool Implementation"
    click K "#mcp-tools" "MCP Client Tool"
    click L "#convert-n8n-tools" "Convert N8n Tools"
    click M "#mcp-sse-connection" "MCP SSE Connection"
    click N "#list-mcp-tools" "List MCP Tools"
    click O "#convert-mcp-tools" "Convert MCP Tools"
    click P "#validate-tool-names" "Validate Tool Names"
    click Q "#prepare-messages" "Prepare Messages"
    click R "#system-message" "System Message"
    click T "#create-agent" "Create Tool Calling Agent"
    click U "#agent-executor" "Agent Executor Invoke"
    click W "#parse-tool-request" "Parse Tool Request"
    click Y1 "#execute-n8n-tool" "Execute N8n Tool"
    click Y2 "#execute-mcp-tool" "Execute MCP Tool"
    click DD "#format-structured-output" "Format Structured Output"
    click EE "#clean-response" "Clean Response"
```

## Code Implementation Details

### Main Agent Node
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/Agent.node.ts`*

```typescript
export class Agent implements INodeType {
	async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
		const agentType = this.getNodeParameter('agent', 0, '') as string;
		const nodeVersion = this.getNode().typeVersion;

		if (agentType === 'conversationalAgent') {
			return await conversationalAgentExecute.call(this, nodeVersion);
		} else if (agentType === 'toolsAgent') {
			return await toolsAgentExecute.call(this);
		} else if (agentType === 'openAiFunctionsAgent') {
			return await openAiFunctionsAgentExecute.call(this, nodeVersion);
		} else if (agentType === 'reActAgent') {
			return await reActAgentAgentExecute.call(this, nodeVersion);
		} else if (agentType === 'sqlAgent') {
			return await sqlAgentAgentExecute.call(this);
		} else if (agentType === 'planAndExecuteAgent') {
			return await planAndExecuteAgentExecute.call(this, nodeVersion);
		}

		throw new NodeOperationError(this.getNode(), `The agent type "${agentType}" is not supported`);
	}
}
```

### Tools Agent Executor
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export async function toolsAgentExecute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
	this.logger.debug('Executing Tools Agent');

	const returnData: INodeExecutionData[] = [];
	const items = this.getInputData();
	const outputParser = await getOptionalOutputParser(this);
	const tools = await getTools(this, outputParser);

	for (let itemIndex = 0; itemIndex < items.length; itemIndex++) {
		try {
			const model = await getChatModel(this);
			const memory = await getOptionalMemory(this);

			const input = getPromptInputByType({
				ctx: this,
				i: itemIndex,
				inputKey: 'text',
				promptTypeKey: 'promptType',
			});

			// Create the agent and execute
			const agent = createToolCallingAgent({
				llm: model,
				tools,
				prompt,
				streamRunnable: false,
			});

			const executor = AgentExecutor.fromAgentAndTools({
				agent: runnableAgent,
				memory,
				tools,
				returnIntermediateSteps: options.returnIntermediateSteps === true,
				maxIterations: options.maxIterations ?? 10,
			});

			const response = await executor.invoke({
				input,
				system_message: options.systemMessage ?? SYSTEM_MESSAGE,
			});

			returnData.push({ json: response });
		} catch (error) {
			// Error handling logic
		}
	}

	return [returnData];
}
```

### Get Chat Model
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export async function getChatModel(ctx: IExecuteFunctions): Promise<BaseChatModel> {
	const model = await ctx.getInputConnectionData(NodeConnectionTypes.AiLanguageModel, 0);
	if (!isChatInstance(model) || !model.bindTools) {
		throw new NodeOperationError(
			ctx.getNode(),
			'Tools Agent requires Chat Model which supports Tools calling',
		);
	}
	return model;
}
```

### Get Optional Memory
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export async function getOptionalMemory(
	ctx: IExecuteFunctions,
): Promise<BaseChatMemory | undefined> {
	return (await ctx.getInputConnectionData(NodeConnectionTypes.AiMemory, 0)) as
		| BaseChatMemory
		| undefined;
}
```

### Get Output Parser
*Reference: `packages/@n8n/nodes-langchain/utils/output_parsers/N8nOutputParser.ts`*

```typescript
export async function getOptionalOutputParser(
	ctx: IExecuteFunctions,
): Promise<N8nOutputParser | undefined> {
	let outputParser: N8nOutputParser | undefined;

	if (ctx.getNodeParameter('hasOutputParser', 0, true) === true) {
		outputParser = (await ctx.getInputConnectionData(
			NodeConnectionTypes.AiOutputParser,
			0,
		)) as N8nOutputParser;
	}

	return outputParser;
}
```

### Discover Tools
*Reference: `packages/@n8n/nodes-langchain/utils/helpers.ts`*

```typescript
export const getConnectedTools = async (
	ctx: IExecuteFunctions | IWebhookFunctions,
	enforceUniqueNames: boolean,
	convertStructuredTool: boolean = true,
	escapeCurlyBrackets: boolean = false,
) => {
	const connectedTools = (
		((await ctx.getInputConnectionData(NodeConnectionTypes.AiTool, 0)) as Array<Toolkit | Tool>) ??
		[]
	).flatMap((toolOrToolkit) => {
		if (toolOrToolkit instanceof Toolkit) {
			return toolOrToolkit.getTools() as Tool[];
		}
		return toolOrToolkit;
	});

	if (!enforceUniqueNames) return connectedTools;

	const seenNames = new Set<string>();
	const finalTools: Tool[] = [];

	for (const tool of connectedTools) {
		const { name } = tool;
		if (seenNames.has(name)) {
			throw new NodeOperationError(
				ctx.getNode(),
				`You have multiple tools with the same name: '${name}', please rename them to avoid conflicts`,
			);
		}
		seenNames.add(name);

		if (convertStructuredTool && tool instanceof N8nTool) {
			finalTools.push(tool.asDynamicTool());
		} else {
			finalTools.push(tool);
		}
	}

	return finalTools;
};
```

### N8n Tools
*Reference: `packages/@n8n/nodes-langchain/utils/N8nTool.ts`*

```typescript
export class N8nTool extends DynamicStructuredTool {
	constructor(
		private context: ISupplyDataFunctions,
		fields: DynamicStructuredToolInput,
	) {
		super(fields);
	}

	asDynamicTool(): DynamicTool {
		const { name, func, schema, context, description } = this;
		const parser = new StructuredOutputParser(schema);

		const wrappedFunc = async function (query: string) {
			let parsedQuery: object;

			try {
				parsedQuery = await parser.parse(query);
			} catch (e) {
				// Graceful error handling for malformed input
				let dataFromModel;
				try {
					dataFromModel = jsonParse<IDataObject>(query, { acceptJSObject: true });
				} catch (error) {
					if (Object.keys(schema.shape).length === 1) {
						const parameterName = Object.keys(schema.shape)[0];
						dataFromModel = { [parameterName]: query };
					} else {
						throw new NodeOperationError(
							context.getNode(),
							`Input is not a valid JSON: ${error.message}`,
						);
					}
				}
				parsedQuery = schema.parse(dataFromModel);
			}

			try {
				const result = await func(parsedQuery);
				return result;
			} catch (e) {
				const { index } = context.addInputData(NodeConnectionTypes.AiTool, [[{ json: { query } }]]);
				void context.addOutputData(NodeConnectionTypes.AiTool, index, e);
				return e.toString();
			}
		};

		return new DynamicTool({
			name,
			description: prepareFallbackToolDescription(description, schema),
			func: wrappedFunc,
		});
	}
}
```

### MCP Tools
*Reference: `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/McpClientTool.node.ts`*

```typescript
async supplyData(this: ISupplyDataFunctions, itemIndex: number): Promise<ToolsAgentAction[]> {
	const credentials = await this.getCredentials<McpCredentials>('mcpApi');
	const { authentication, sseEndpoint } = credentials;

	const { headers } = await getAuthHeaders(this, authentication);
	const client = await connectMcpClient({
		sseEndpoint,
		headers,
		name: 'n8n-mcp-client',
		version: 1,
	});

	if (!client.ok) {
		this.logger.error('McpClientTool: Failed to connect to MCP Server', {
			error: client.error,
		});
		// Error handling logic
	}

	const allTools = await getAllTools(client.result);
	const mcpTools = getSelectedTools({
		tools: allTools,
		mode: this.getNodeParameter('toolsToInclude', itemIndex) as McpToolIncludeMode,
		includeTools: this.getNodeParameter('includeTools', itemIndex, []) as string[],
		excludeTools: this.getNodeParameter('excludeTools', itemIndex, []) as string[],
	});

	const tools = mcpTools.map((tool) =>
		mcpToolToDynamicTool(
			tool,
			createCallTool(tool.name, client.result, (error) => {
				this.logger.error(`McpClientTool: Error executing tool ${tool.name}`, { error });
				return `Error: ${error}`;
			}),
		),
	);

	return new McpToolkit(tools);
}
```

### Convert N8n Tools
*Reference: `packages/@n8n/nodes-langchain/utils/N8nTool.ts`*

```typescript
asDynamicTool(): DynamicTool {
	const { name, func, schema, context, description } = this;
	const parser = new StructuredOutputParser(schema);

	const wrappedFunc = async function (query: string) {
		let parsedQuery: object;
		
		// Parse and validate input
		try {
			parsedQuery = await parser.parse(query);
		} catch (e) {
			// Handle parsing errors gracefully
		}

		// Execute the tool function
		try {
			const result = await func(parsedQuery);
			return result;
		} catch (e) {
			return e.toString();
		}
	};

	return new DynamicTool({
		name,
		description: prepareFallbackToolDescription(description, schema),
		func: wrappedFunc,
	});
}
```

### MCP SSE Connection
*Reference: `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/utils.ts`*

```typescript
export async function connectMcpClient({
	headers,
	sseEndpoint,
	name,
	version,
}: {
	sseEndpoint: string;
	headers?: Record<string, string>;
	name: string;
	version: number;
}): Promise<Result<Client, ConnectMcpClientError>> {
	try {
		const endpoint = normalizeAndValidateUrl(sseEndpoint);

		if (!endpoint.ok) {
			return createResultError({ type: 'invalid_url', error: endpoint.error });
		}

		const transport = new SSEClientTransport(endpoint.result, {
			eventSourceInit: {
				fetch: async (url, init) =>
					await fetch(url, {
						...init,
						headers: {
							...headers,
							Accept: 'text/event-stream',
						},
					}),
			},
			requestInit: { headers },
		});

		const client = new Client(
			{ name, version: version.toString() },
			{ capabilities: { tools: {} } },
		);

		await client.connect(transport);
		return createResultOk(client);
	} catch (error) {
		return createResultError({ type: 'connection', error });
	}
}
```

### List MCP Tools
*Reference: `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/utils.ts`*

```typescript
export async function getAllTools(client: Client, cursor?: string): Promise<McpTool[]> {
	const { tools, nextCursor } = await client.listTools({ cursor });

	if (nextCursor) {
		return (tools as McpTool[]).concat(await getAllTools(client, nextCursor));
	}

	return tools as McpTool[];
}
```

### Convert MCP Tools
*Reference: `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/utils.ts`*

```typescript
export function mcpToolToDynamicTool(
	tool: McpTool,
	onCallTool: DynamicStructuredToolInput['func'],
) {
	return new DynamicStructuredTool({
		name: tool.name,
		description: tool.description ?? '',
		schema: convertJsonSchemaToZod(tool.inputSchema),
		func: onCallTool,
		metadata: { isFromToolkit: true },
	});
}
```

### Validate Tool Names
*Reference: `packages/@n8n/nodes-langchain/utils/helpers.ts`*

```typescript
const seenNames = new Set<string>();
const finalTools: Tool[] = [];

for (const tool of connectedTools) {
	const { name } = tool;
	if (seenNames.has(name)) {
		throw new NodeOperationError(
			ctx.getNode(),
			`You have multiple tools with the same name: '${name}', please rename them to avoid conflicts`,
		);
	}
	seenNames.add(name);
	
	if (convertStructuredTool && tool instanceof N8nTool) {
		finalTools.push(tool.asDynamicTool());
	} else {
		finalTools.push(tool);
	}
}
```

### Prepare Messages
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export async function prepareMessages(
	ctx: IExecuteFunctions,
	itemIndex: number,
	options: {
		systemMessage?: string;
		passthroughBinaryImages?: boolean;
		outputParser?: N8nOutputParser;
	},
): Promise<BaseMessagePromptTemplateLike[]> {
	const useSystemMessage = options.systemMessage ?? ctx.getNode().typeVersion < 1.9;

	const messages: BaseMessagePromptTemplateLike[] = [];

	if (useSystemMessage) {
		messages.push([
			'system',
			`{system_message}${options.outputParser ? '\n\n{formatting_instructions}' : ''}`,
		]);
	} else if (options.outputParser) {
		messages.push(['system', '{formatting_instructions}']);
	}

	messages.push(['placeholder', '{chat_history}'], ['human', '{input}']);

	// Add binary message if present
	const hasBinaryData = ctx.getInputData()?.[itemIndex]?.binary !== undefined;
	if (hasBinaryData && options.passthroughBinaryImages) {
		const binaryMessage = await extractBinaryMessages(ctx, itemIndex);
		messages.push(binaryMessage);
	}

	messages.push(['placeholder', '{agent_scratchpad}']);
	return messages;
}
```

### System Message
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/prompt.ts`*

```typescript
export const SYSTEM_MESSAGE = 'You are a helpful assistant';
```

### Create Agent
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
// Create the base agent that calls tools
const agent = createToolCallingAgent({
	llm: model,
	tools,
	prompt,
	streamRunnable: false,
});

agent.streamRunnable = false;

// Wrap the agent with parsers and fixes
const runnableAgent = RunnableSequence.from([
	agent,
	getAgentStepsParser(outputParser, memory),
	fixEmptyContentMessage,
]);
```

### Agent Executor
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
const executor = AgentExecutor.fromAgentAndTools({
	agent: runnableAgent,
	memory,
	tools,
	returnIntermediateSteps: options.returnIntermediateSteps === true,
	maxIterations: options.maxIterations ?? 10,
});

// Invoke the executor with the given input and system message
const response = await executor.invoke(
	{
		input,
		system_message: options.systemMessage ?? SYSTEM_MESSAGE,
		formatting_instructions:
			'IMPORTANT: For your response to user, you MUST use the `format_final_json_response` tool with your complete answer formatted according to the required schema.',
	},
	{ signal: this.getExecutionCancelSignal() },
);
```

### Parse Tool Request
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export const getAgentStepsParser =
	(outputParser?: N8nOutputParser, memory?: BaseChatMemory) =>
	async (steps: AgentFinish | AgentAction[]): Promise<AgentFinish | AgentAction[]> => {
		// Check if the steps contain the 'format_final_json_response' tool invocation
		if (Array.isArray(steps)) {
			const responseParserTool = steps.find((step) => step.tool === 'format_final_json_response');
			if (responseParserTool && outputParser) {
				const toolInput = responseParserTool.toolInput;
				const parserInput = toolInput instanceof Object ? JSON.stringify(toolInput) : toolInput;
				const returnValues = (await outputParser.parse(parserInput)) as Record<string, unknown>;
				return handleParsedStepOutput(returnValues, memory);
			}
		}

		return handleAgentFinishOutput(steps);
	};
```

### Execute N8n Tool
*Reference: `packages/@n8n/nodes-langchain/utils/N8nTool.ts`*

```typescript
const wrappedFunc = async function (query: string) {
	let parsedQuery: object;

	// Parse input with error handling
	try {
		parsedQuery = await parser.parse(query);
	} catch (e) {
		// Handle malformed input gracefully
	}

	try {
		// Call tool function with parsed query
		const result = await func(parsedQuery);
		return result;
	} catch (e) {
		const { index } = context.addInputData(NodeConnectionTypes.AiTool, [[{ json: { query } }]]);
		void context.addOutputData(NodeConnectionTypes.AiTool, index, e);
		return e.toString();
	}
};
```

### Execute MCP Tool
*Reference: `packages/@n8n/nodes-langchain/nodes/mcp/McpClientTool/utils.ts`*

```typescript
export const createCallTool =
	(name: string, client: Client, onError: (error: string | undefined) => void) =>
	async (args: IDataObject) => {
		let result: Awaited<ReturnType<Client['callTool']>>;
		try {
			result = await client.callTool({ name, arguments: args }, CompatibilityCallToolResultSchema);
		} catch (error) {
			return onError(getErrorDescriptionFromToolCall(error));
		}

		if (result.isError) {
			return onError(getErrorDescriptionFromToolCall(result));
		}

		if (result.toolResult !== undefined) {
			return result.toolResult;
		}

		if (result.content !== undefined) {
			return result.content;
		}

		return result;
	};
```

### Format Structured Output
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export function handleParsedStepOutput(
	output: Record<string, unknown>,
	memory?: BaseChatMemory,
): { returnValues: Record<string, unknown>; log: string } {
	return {
		returnValues: memory ? { output: JSON.stringify(output) } : output,
		log: 'Final response formatted',
	};
}
```

### Clean Response
*Reference: `packages/@n8n/nodes-langchain/nodes/agents/Agent/agents/ToolsAgent/execute.ts`*

```typescript
export function handleAgentFinishOutput(
	steps: AgentFinish | AgentAction[],
): AgentFinish | AgentAction[] {
	const agentFinishSteps = steps as AgentMultiOutputFinish | AgentFinish;

	if (agentFinishSteps.returnValues) {
		const isMultiOutput = Array.isArray(agentFinishSteps.returnValues?.output);
		if (isMultiOutput) {
			const multiOutputSteps = agentFinishSteps.returnValues.output as Array<{
				index: number;
				type: string;
				text: string;
			}>;
			const isTextOnly = multiOutputSteps.every((output) => 'text' in output);
			if (isTextOnly) {
				agentFinishSteps.returnValues.output = multiOutputSteps
					.map((output) => output.text)
					.join('\n')
					.trim();
			}
			return agentFinishSteps;
		}
	}

	return agentFinishSteps;
}
```

## Flow Summary

This comprehensive flow shows how the n8n AI Agent processes user questions through multiple phases:

1. **Input Processing** - Validates and prepares user input
2. **Component Initialization** - Sets up LLM, memory, and output parser
3. **Tool Discovery** - Finds both N8n and MCP tools, converts them to compatible format
4. **Agent Creation** - Builds LangChain agent with tools and system prompt
5. **Iterative Execution** - LLM processes input, calls tools as needed, continues until completion
6. **Output Processing** - Formats response according to requirements and saves to memory

The implementation seamlessly integrates local N8n tools with remote MCP tools, providing a unified interface for AI agents to interact with various external services and workflows.
