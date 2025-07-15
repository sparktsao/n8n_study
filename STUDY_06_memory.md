# Study 007: Simple Memory Node in n8n AI Agents

## Overview

The "Simple Memory" node (technically implemented as `MemoryBufferWindow`) is a crucial component for AI agents in n8n, providing conversation memory management using LangChain's `BufferWindowMemory` class.

## Key Questions Answered

### 1. Does it only keep the previous K runs of chat as memory?

**YES** - The Simple Memory node implements a sliding window memory system that keeps only the last K conversation turns.

**IMPORTANT CLARIFICATION**: The AI Agent node **DOES NOT have memory by default**. Without connecting a memory node, each interaction is completely independent with no chat history.

**How Memory Works:**
- **Without Memory Node**: AI Agent has NO memory - each request is isolated
- **With Simple Memory Node**: AI Agent gets the last K conversation turns as context
- Uses LangChain's `BufferWindowMemory` with a configurable `k` parameter
- Default context window length is 5 interactions (turns)
- Each "turn" consists of a human message + AI response pair
- When the limit is exceeded, older messages are automatically dropped (FIFO - First In, First Out)
- The `k` parameter is exposed as "Context Window Length" in the UI

**Code Evidence:**
```typescript
// AI Agent checks for connected memory
const memory = (await this.getInputConnectionData(NodeConnectionTypes.AiMemory, 0)) as
    | BaseChatMemory
    | undefined;

// If no memory connected, some agents create a default BufferMemory
memory ??
new BufferMemory({
    returnMessages: true,
    memoryKey: 'chat_history',
    inputKey: 'input',
    outputKey: 'output',
});

// Simple Memory provides the sliding window
const memory = await memoryInstance.getMemory(`${workflowId}__${sessionId}`, {
    k: contextWindowLength,  // This is the K parameter
    inputKey: 'input',
    memoryKey: 'chat_history',
    outputKey: 'output',
    returnMessages: true,
});
```

### 2. Does it do anything like MEM0, asking LLM to decide what to memorize?

**NO** - The Simple Memory node does NOT use LLM-based intelligent memory selection like MEM0.

**What it does instead:**
- Simple sliding window approach based purely on recency
- No intelligent filtering, summarization, or importance-based selection
- No LLM involvement in deciding what to keep or discard
- Purely mechanical: keep the last K turns, drop everything older

**Comparison with MEM0:**
- MEM0: Uses LLM to analyze and decide what's important to remember
- Simple Memory: Uses chronological order only - most recent K conversations

### 3. Can we manually define what to store and retrieve?

**YES** - Through the Memory Manager node, you have extensive manual control over memory content.

**Manual Control Capabilities:**

#### Memory Manager Node Features:
- **Insert Messages**: Add specific messages with different types (AI, System, User)
- **Delete Messages**: Remove last N messages or clear all memory
- **Retrieve Messages**: Get messages with various formatting options
- **Override Memory**: Replace existing memory completely
- **Hide Messages**: Keep messages in memory but hide from UI

#### Operations Available:
1. **Get Many Messages**
   - Retrieve chat messages from connected memory
   - Option to simplify output format
   - Group messages or return individually

2. **Insert Messages**
   - Insert mode: Add alongside existing messages
   - Override mode: Replace all existing messages
   - Support for AI, System, and User message types
   - Option to hide messages from UI while keeping in memory

3. **Delete Messages**
   - Delete last N messages
   - Clear all messages from memory

**Code Evidence:**
```typescript
// Message insertion with manual control
for (const message of messagesToInsert) {
    const MessageClass = new templateMapper[message.type](message.message);
    
    if (message.hideFromUI) {
        MessageClass.additional_kwargs.hideFromUI = true;
    }
    
    await memory.chatHistory.addMessage(MessageClass);
}
```

## Architecture and Implementation

### Memory Storage
- **In-Memory Storage**: Data stored in n8n server memory (not persistent across restarts)
- **Session-Based**: Each session gets its own memory buffer
- **Automatic Cleanup**: Stale buffers are cleaned up after 1 hour of inactivity
- **Workflow Isolation**: Memory keys are prefixed with workflow ID to avoid collisions

### Session Management
```typescript
class MemoryChatBufferSingleton {
    private memoryBuffer: Map<string, {
        buffer: BufferWindowMemory;
        created: Date;
        last_accessed: Date;
    }>;
}
```

### Memory Lifecycle
1. **Creation**: New memory instance created per session
2. **Access**: Last accessed timestamp updated on each use
3. **Cleanup**: Automatic cleanup of buffers inactive for >1 hour
4. **Persistence**: Memory lost on n8n restart (in-memory only)

## Configuration Options

### Simple Memory Node
- **Session ID**: How to identify unique conversation sessions
  - From connected Chat Trigger
  - Custom expression/static value
- **Context Window Length**: Number of conversation turns to keep (default: 5)

### Memory Manager Node
- **Operation Mode**: Load, Insert, or Delete messages
- **Insert Mode**: Insert alongside existing or override all
- **Delete Mode**: Delete last N messages or clear all
- **Message Types**: AI, System, User messages
- **Output Options**: Simplified format, grouped messages

## Use Cases and Patterns

### AI Agent WITHOUT Memory (Default)
```
Chat Trigger → AI Agent
```
- **NO conversation history** - each interaction is independent
- AI Agent treats every request as a new conversation
- No context from previous messages
- Suitable for single-shot questions or stateless interactions

### AI Agent WITH Simple Memory
```
Chat Trigger → Simple Memory → AI Agent
```
- Provides conversation context to AI agent
- Maintains last K conversation turns
- Automatic memory management
- Enables multi-turn conversations with context

### Advanced Memory Management
```
Chat Trigger → Simple Memory → Memory Manager → AI Agent
```
- Manual control over memory content
- Custom message insertion/deletion
- System messages for context injection

### Memory Inspection
```
Simple Memory → Memory Manager (Load) → Process Messages
```
- Retrieve and analyze conversation history
- Export memory content for analysis
- Debug conversation flow

## Limitations

1. **No Default Memory**: AI Agent has NO memory without explicitly connecting a memory node
2. **No Persistence**: Memory lost on server restart
3. **No Intelligence**: No smart filtering or summarization
4. **Fixed Window**: Simple K-based sliding window only
5. **Memory Usage**: All stored in server RAM
6. **No Compression**: Full message content stored without optimization

## Comparison with Other Memory Types

| Memory Type | Persistence | Intelligence | Manual Control | Use Case |
|-------------|-------------|--------------|----------------|----------|
| Simple Memory | No | None | Via Manager | Development/Testing |
| Redis Memory | Yes | None | Via Manager | Production |
| Postgres Memory | Yes | None | Via Manager | Production |
| Zep Memory | Yes | Some | Limited | Advanced |
| Motorhead | Yes | Some | Limited | Advanced |

## Best Practices

1. **Development**: Use Simple Memory for testing and development
2. **Production**: Use persistent memory types (Redis, Postgres) for production
3. **Context Window**: Adjust K based on conversation complexity and model context limits
4. **Memory Management**: Use Memory Manager for system messages and context injection
5. **Session Management**: Ensure proper session ID handling for multi-user scenarios

## Event Tracking

The system tracks memory operations for monitoring:
- `ai-messages-retrieved-from-memory`: When messages are loaded
- `ai-message-added-to-memory`: When messages are stored

## Conclusion

**CRITICAL UNDERSTANDING**: AI Agent nodes in n8n have **NO memory by default**. Each interaction is completely independent unless you explicitly connect a memory node.

The Simple Memory node provides a straightforward, K-based sliding window memory system for AI agents. While it lacks the intelligence of systems like MEM0, it offers:

1. **Enables Conversation Memory**: Transforms stateless AI agents into conversational ones
2. **Simplicity**: Easy to understand and configure
3. **Manual Control**: Extensive control via Memory Manager
4. **Session Isolation**: Proper multi-session support
5. **Development Friendly**: No external dependencies

**Key Takeaway**: Without a memory node, your AI agent will not remember previous interactions. To enable conversational AI with context, you MUST connect a memory node like Simple Memory.

For production use cases requiring persistence or intelligent memory management, consider using other memory types like Redis, Postgres, or specialized solutions like Zep.
