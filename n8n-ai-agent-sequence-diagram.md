# n8n AI Agent Node - Detailed Sequence Diagram

This document provides a comprehensive analysis of how the n8n AI Agent node processes user questions, interacts with tools (including MCP tools), and manages the LLM conversation flow.

## Overview

The n8n AI Agent node supports multiple agent types, with **Tools Agent** being the recommended implementation. The flow involves:

1. **Question Processing** - User input validation and preparation
2. **Tool Discovery** - Gathering available tools (n8n tools + MCP tools)
3. **LLM System Prompt** - Embedding tool descriptions into system prompt
4. **LLM Response Analysis** - Checking for tool calling requests
5. **Tool Execution** - Calling requested tools if needed
6. **Iterative LLM Calls** - Continuing conversation until completion

## Detailed Sequence Diagram

```mermaid
sequenceDiagram
    participant User
    participant AgentNode as AI Agent Node
    participant ToolsAgent as Tools Agent Executor
    participant LLM as Language Model
    participant ToolConnector as Tool Connector
    participant MCPClient as MCP Client
    participant MCPServer as MCP Server
    participant N8nTool as N8n Tool
    participant OutputParser as Output Parser
    participant Memory as Memory Store

    Note over User, Memory: QUESTION PROCESSING PHASE
    User->>AgentNode: User Question Input
    AgentNode->>AgentNode: Validate Input Parameters
    AgentNode->>AgentNode: Determine Agent Type (toolsAgent)
    AgentNode->>ToolsAgent: Execute with Input

    Note over ToolsAgent, Memory: INITIALIZATION PHASE
    ToolsAgent->>LLM: getChatModel() - Validate LLM supports tool calling
    ToolsAgent->>Memory: getOptionalMemory() - Load conversation history
    ToolsAgent->>OutputParser: getOptionalOutputParser() - Load output format requirements

    Note over ToolsAgent, Memory: TOOL DISCOVERY PHASE
    ToolsAgent->>ToolConnector: getConnectedTools()
    
    ToolConnector->>N8nTool: Discover N8n Tools
    N8nTool-->>ToolConnector: Return N8n Tool Definitions
    
    ToolConnector->>MCPClient: Discover MCP Tools
    MCPClient->>MCPServer: listTools() - Get available tools
    MCPServer-->>MCPClient: Return MCP Tool Definitions
    MCPClient->>MCPClient: mcpToolToDynamicTool() - Convert to LangChain format
    MCPClient-->>ToolConnector: Return Converted MCP Tools
    
    ToolConnector->>ToolConnector: Validate Unique Tool Names
    ToolConnector->>ToolConnector: Convert N8nTool to DynamicTool
    ToolConnector-->>ToolsAgent: Return All Available Tools

    Note over ToolsAgent, Memory: OUTPUT PARSER TOOL INJECTION
    alt Output Parser Connected
        ToolsAgent->>ToolsAgent: Create format_final_json_response tool
        ToolsAgent->>ToolsAgent: Add to tools list with schema validation
    end

    Note over ToolsAgent, Memory: PROMPT PREPARATION PHASE
    ToolsAgent->>ToolsAgent: prepareMessages() - Build prompt template
    ToolsAgent->>ToolsAgent: Include system message + tool descriptions
    ToolsAgent->>ToolsAgent: Add chat history placeholder
    ToolsAgent->>ToolsAgent: Add user input placeholder
    ToolsAgent->>ToolsAgent: Add agent scratchpad placeholder

    Note over ToolsAgent, Memory: AGENT CREATION PHASE
    ToolsAgent->>LLM: createToolCallingAgent() - Bind tools to LLM
    ToolsAgent->>ToolsAgent: Create RunnableSequence with parsers
    ToolsAgent->>ToolsAgent: Create AgentExecutor with tools and memory

    Note over ToolsAgent, Memory: LLM INTERACTION LOOP
    ToolsAgent->>LLM: Initial invoke() with user question + system prompt
    
    loop Until Final Answer or Max Iterations
        LLM->>LLM: Process input with tool descriptions in context
        LLM-->>ToolsAgent: Return response (text or tool calls)
        
        alt LLM Requests Tool Call
            ToolsAgent->>ToolsAgent: Parse tool call request
            
            alt N8n Tool Call
                ToolsAgent->>N8nTool: Execute tool with parameters
                N8nTool->>N8nTool: Validate input schema (Zod)
                N8nTool->>N8nTool: Execute n8n workflow/function
                N8nTool-->>ToolsAgent: Return tool result
            else MCP Tool Call
                ToolsAgent->>MCPClient: Execute MCP tool
                MCPClient->>MCPServer: callTool() with parameters
                MCPServer->>MCPServer: Execute tool logic
                MCPServer-->>MCPClient: Return tool result
                MCPClient-->>ToolsAgent: Return formatted result
            end
            
            ToolsAgent->>LLM: Send tool results back to LLM
        else LLM Provides Final Answer
            break Exit Loop
        end
    end

    Note over ToolsAgent, Memory: OUTPUT PROCESSING PHASE
    alt Output Parser Required
        ToolsAgent->>ToolsAgent: Check for format_final_json_response tool usage
        ToolsAgent->>OutputParser: Parse structured output
        OutputParser-->>ToolsAgent: Return validated structured data
    else No Output Parser
        ToolsAgent->>ToolsAgent: handleAgentFinishOutput() - Normalize response
    end

    Note over ToolsAgent, Memory: MEMORY STORAGE PHASE
    alt Memory Connected
        ToolsAgent->>Memory: Save conversation context
        Memory->>Memory: Store user question + agent response
    end

    Note over ToolsAgent, Memory: RESPONSE FINALIZATION
    ToolsAgent->>ToolsAgent: Clean up internal fields
    ToolsAgent-->>AgentNode: Return final response
    AgentNode-->>User: Return processed answer

    Note over User, Memory: ERROR HANDLING
    alt Tool Execution Error
        N8nTool-->>ToolsAgent: Error message
        ToolsAgent->>LLM: Send error to LLM for handling
    else MCP Connection Error
        MCPClient-->>ToolsAgent: Connection/execution error
        ToolsAgent->>LLM: Send error to LLM for handling
    else LLM Error
        LLM-->>ToolsAgent: Processing error
        ToolsAgent-->>AgentNode: Return error response
    end
```

## Key Components Breakdown

### Question Processing
- **Input Validation**: Checks if user input is provided and valid
- **Agent Type Selection**: Determines which agent implementation to use (Tools Agent recommended)
- **Parameter Extraction**: Gets system message, max iterations, and other options

### Tool Discovery & Integration

#### N8n Tools
- Connected via `ai_tool` input connections
- Converted from `N8nTool` to `DynamicStructuredTool`
- Schema validation using Zod
- Direct execution within n8n workflow context

#### MCP Tools
- Connected via MCP Client Tool node
- Uses SSE (Server-Sent Events) transport for communication
- Tool discovery via `listTools()` MCP protocol method
- Real-time tool execution via `callTool()` MCP protocol method
- Automatic conversion to LangChain-compatible format

### 3. LLM System Prompt Construction
```
System Message: "You are a helpful assistant"
+ Tool Descriptions: Embedded automatically from connected tools
+ Output Format Instructions: If output parser connected
+ Chat History: Previous conversation context
+ Current Input: User's question
+ Agent Scratchpad: Tool execution results
```

### 4. Tool Calling Flow

#### Tool Call Detection
- LLM response parsed for tool calling requests
- Supports both structured tool schemas and function calling
- Validates tool names against available tools

#### Tool Execution
- **N8n Tools**: Direct execution with schema validation
- **MCP Tools**: Remote execution via MCP protocol
- **Error Handling**: Graceful error recovery and reporting to LLM

#### Result Processing
- Tool results formatted and sent back to LLM
- LLM continues reasoning with tool outputs
- Iterative process until final answer or max iterations reached

### 5. Output Processing

#### Structured Output (with Output Parser)
- Special `format_final_json_response` tool injected
- LLM must use this tool for final response
- Schema validation ensures correct format
- Supports complex nested JSON structures

#### Standard Output (no Output Parser)
- Direct text response from LLM
- Automatic formatting and cleanup
- Handles different model output formats (OpenAI vs Anthropic)

### 6. Memory Management
- Optional conversation history storage
- Maintains context across multiple interactions
- Supports different memory implementations
- Automatic cleanup of internal fields

## Error Handling Strategies

1. **Tool Connection Errors**: Graceful fallback, error reporting to LLM
2. **Tool Execution Errors**: Error message passed to LLM for handling
3. **LLM Errors**: Proper error propagation with context
4. **Schema Validation Errors**: Detailed error messages for debugging
5. **MCP Protocol Errors**: Connection retry and error reporting

## Performance Optimizations

1. **Tool Caching**: Tools discovered once per execution
2. **Schema Validation**: Efficient Zod-based validation
3. **Memory Optimization**: Optional memory usage
4. **Streaming Support**: Disabled for stability in tool calling scenarios
5. **Connection Pooling**: MCP client connection reuse

## Security Considerations

1. **Tool Access Control**: Only explicitly connected tools available
2. **Input Validation**: All tool inputs validated against schemas
3. **MCP Authentication**: Support for header and bearer token auth
4. **Error Sanitization**: Sensitive information filtered from error messages
5. **Resource Limits**: Max iterations and timeout controls

This implementation provides a robust, extensible framework for AI agents that can seamlessly integrate both local n8n tools and remote MCP tools while maintaining conversation context and providing structured outputs.
