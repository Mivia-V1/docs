# 🤖 OpenAI Assistant Integration PRD

# 🛠️ OpenAI Assistant Integration - Technical Implementation Specification

**MiVia Chat Feature Enhancement - Developer Implementation Guide**

---

## 📋 Overview

This document provides technical implementation requirements for integrating OpenAI Assistant API with MiVia's existing chat feature while maintaining current UI/UX and leveraging existing Supabase schema.

**Key Constraints:**

- Preserve existing Swift chat UI/UX completely
- Utilize current Supabase schema (`journal_ai_chats`, `user_assistant_threads`, etc.)
- Replace simulated AI responses with real OpenAI API calls
- Implement via Supabase Edge Functions for security and scalability

---

## 🗄️ Current Architecture Analysis

### Existing Supabase Tables

### `journal_ai_chats` (Primary Message Store)

```sql
CREATE TABLE journal_ai_chats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    journal_entry_id UUID,
    message TEXT NOT NULL,           -- User's input message
    response TEXT,                   -- AI's response (currently simulated)
    message_type TEXT NOT NULL,      -- 'user' | 'ai'
    chat_type TEXT,                  -- 'general' | 'pillar_health' | etc.
    goal_id UUID,                    -- Associated user goal
    created_at TIMESTAMP DEFAULT NOW()
);

```

### `user_assistant_threads` (Thread Management)

```sql
CREATE TABLE user_assistant_threads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    assistant_type TEXT NOT NULL,    -- 'general_coach' | 'pillar_coach' | etc.
    thread_id TEXT NOT NULL,         -- OpenAI Thread ID (currently generated locally)
    created_at TIMESTAMP DEFAULT NOW(),
    last_interaction_at TIMESTAMP DEFAULT NOW()
);

```

### Related Tables for Context

- `pillars` - Wellness framework categories
- `user_goals` - User-specific goals per pillar
- profiles - User profiles and preferences

### Current Swift Implementation Points

### `SupabaseChatViewModel.swift`

```swift
// Current method that needs OpenAI integration
private func getAIResponse(for userMessage: JournalAIChat) async {
    isLoading = true

    // TODO: Replace this simulated response with OpenAI API call
    let response = generateAIResponse(to: userMessage.message, chatType: userMessage.chatType)
    let aiMessage = JournalAIChat.createAIResponse(to: userMessage, response: response)

    messages.append(aiMessage.toChatMessage())

    if SupabaseManager.isConfigured {
        sessionManager.addMessage(aiMessage)
    }

    isLoading = false
}

```

### `ChatRepository.swift`

```swift
// Thread management methods that need OpenAI integration
func getOrCreateThread(for userId: UUID, assistantType: UserAssistantThread.AssistantType) async throws -> UserAssistantThread
func saveThread(_ thread: UserAssistantThread) async throws
func updateThread(_ thread: UserAssistantThread) async throws

```

---

## 🔧 Suggested Supabase Edge Functions

### 1. `openai-chat-completion` Edge Function

**Purpose**: Handle OpenAI API calls securely server-side

**Endpoint**: `POST /functions/v1/openai-chat-completion`

**Request Body**:

```tsx
interface ChatCompletionRequest {
  user_id: string;
  message: string;
  chat_type?: string;
  goal_id?: string;
  thread_id?: string;
}

```

**Response**:

```tsx
interface ChatCompletionResponse {
  response: string;
  thread_id: string;
  tokens_used: number;
  model_used: string;
  cost_estimate: number;
}

```

**Implementation**:

```tsx
import { serve } from "<https://deno.land/std@0.168.0/http/server.ts>"
import { createClient } from '<https://esm.sh/@supabase/supabase-js@2>'

const openai = new OpenAI({
  apiKey: Deno.env.get('OPENAI_API_KEY'),
})

serve(async (req) => {
  try {
    const { user_id, message, chat_type, goal_id, thread_id } = await req.json()

    // 1. Get or create OpenAI thread
    const activeThreadId = thread_id || await createOpenAIThread(user_id, chat_type)

    // 2. Build user context from database
    const context = await buildUserContext(user_id, chat_type, goal_id)

    // 3. Get appropriate assistant ID
    const assistantId = getAssistantId(chat_type)

    // 4. Send message to OpenAI
    await openai.beta.threads.messages.create(activeThreadId, {
      role: "user",
      content: message
    })

    // 5. Run assistant with context
    const run = await openai.beta.threads.runs.create(activeThreadId, {
      assistant_id: assistantId,
      additional_instructions: context
    })

    // 6. Wait for completion and get response
    const response = await waitForRunCompletion(activeThreadId, run.id)

    // 7. Save to database and return
    await saveMessageToDatabase(user_id, message, response, chat_type, goal_id, activeThreadId)

    return new Response(JSON.stringify({
      response,
      thread_id: activeThreadId,
      tokens_used: run.usage?.total_tokens || 0,
      model_used: "gpt-4-turbo-preview",
      cost_estimate: calculateCost(run.usage?.total_tokens || 0)
    }), {
      headers: { "Content-Type": "application/json" },
    })

  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    })
  }
})

```

### 2. `user-context-builder` Edge Function

**Purpose**: Aggregate user context for AI conversations

**Endpoint**: `POST /functions/v1/user-context-builder`

**Implementation**:

```tsx
async function buildUserContext(userId: string, chatType?: string, goalId?: string): Promise<string> {
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  // Get user's current pillar ratings (latest check-in)
  const { data: pillarRatings } = await supabase
    .from('pillar_ratings')
    .select('*')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(1)

  // Get active goals for context
  const { data: goals } = await supabase
    .from('user_goals')
    .select('*, pillars(display_name)')
    .eq('user_id', userId)
    .eq('is_active', true)

  // Get recent conversation history (last 10 messages)
  const { data: recentMessages } = await supabase
    .from('journal_ai_chats')
    .select('message, response, chat_type, created_at')
    .eq('user_id', userId)
    .order('created_at', { ascending: false })
    .limit(10)

  // Get specific goal if provided
  let specificGoal = null
  if (goalId) {
    const { data } = await supabase
      .from('user_goals')
      .select('*, pillars(display_name)')
      .eq('id', goalId)
      .single()
    specificGoal = data
  }

  // Build context string
  const context = `
User Context for AI Coaching:

CURRENT PILLAR RATINGS (Latest Check-in):
${pillarRatings?.map(r => `- ${r.pillar}: ${r.rating}%`).join('\\n') || 'No recent ratings'}

ACTIVE GOALS:
${goals?.map(g => `- ${g.pillars.display_name}: ${g.title} (${g.focus})`).join('\\n') || 'No active goals'}

${specificGoal ? `SPECIFIC GOAL FOCUS: ${specificGoal.title} - ${specificGoal.focus}` : ''}

RECENT CONVERSATION THEMES:
${recentMessages?.slice(0, 5).map(m => `- User asked about: ${m.message.substring(0, 100)}...`).join('\\n') || 'No recent conversations'}

CONVERSATION TYPE: ${chatType || 'general'}

Please provide coaching that:
1. References their current pillar ratings when relevant
2. Connects to their active goals
3. Maintains conversation continuity
4. Adapts tone to their previous interactions
5. Focuses on the ${chatType} pillar if specified
  `.trim()

  return context
}

```

### 3. `thread-manager` Edge Function

**Purpose**: Manage OpenAI threads and database synchronization

**Endpoint**: `POST /functions/v1/thread-manager`

**Implementation**:

```tsx
serve(async (req) => {
  const { action, user_id, assistant_type, thread_id } = await req.json()

  switch (action) {
    case 'get_or_create':
      return await getOrCreateThread(user_id, assistant_type)
    case 'update_interaction':
      return await updateThreadInteraction(thread_id)
    case 'get_threads':
      return await getUserThreads(user_id)
  }
})

async function getOrCreateThread(userId: string, assistantType: string): Promise<Response> {
  const supabase = createClient(...)

  // Check for existing thread
  const { data: existingThread } = await supabase
    .from('user_assistant_threads')
    .select('*')
    .eq('user_id', userId)
    .eq('assistant_type', assistantType)
    .single()

  if (existingThread && existingThread.thread_id) {
    // Verify thread exists in OpenAI
    try {
      await openai.beta.threads.retrieve(existingThread.thread_id)
      return new Response(JSON.stringify(existingThread))
    } catch {
      // Thread doesn't exist in OpenAI, create new one
    }
  }

  // Create new OpenAI thread
  const openaiThread = await openai.beta.threads.create()

  // Save or update in database
  const threadData = {
    user_id: userId,
    assistant_type: assistantType,
    thread_id: openaiThread.id,
    last_interaction_at: new Date().toISOString()
  }

  const { data, error } = await supabase
    .from('user_assistant_threads')
    .upsert(threadData)
    .select()
    .single()

  return new Response(JSON.stringify(data))
}

```

---

## 🔄 Swift Integration Changes

### 1. Enhanced `SupabaseChatViewModel.swift`

**Replace the `getAIResponse` method:**

```swift
private func getAIResponse(for userMessage: JournalAIChat) async {
    isLoading = true

    do {
        // 1. Call Supabase Edge Function instead of generating response locally
        let response = try await callOpenAIChatCompletion(for: userMessage)

        // 2. Create AI message with real OpenAI response
        let aiMessage = JournalAIChat.createAIResponse(to: userMessage, response: response.response)

        // 3. Update UI
        await MainActor.run {
            messages.append(aiMessage.toChatMessage())

            // Trigger scroll to bottom after AI message
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                self.scrollToBottomTrigger.toggle()
            }
        }

        // 4. Save to database if configured
        if SupabaseManager.isConfigured {
            sessionManager.addMessage(aiMessage)
        }

    } catch {
        // Fallback to simulated response on error
        print("❌ OpenAI API call failed: \\(error)")
        let fallbackResponse = generateAIResponse(to: userMessage.message, chatType: userMessage.chatType)
        let aiMessage = JournalAIChat.createAIResponse(to: userMessage, response: fallbackResponse)

        await MainActor.run {
            messages.append(aiMessage.toChatMessage())
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                self.scrollToBottomTrigger.toggle()
            }
        }
    }

    await MainActor.run {
        isLoading = false
    }
}

private func callOpenAIChatCompletion(for userMessage: JournalAIChat) async throws -> OpenAIChatResponse {
    let requestBody: [String: Any] = [
        "user_id": userMessage.userId.uuidString,
        "message": userMessage.message,
        "chat_type": userMessage.chatType?.rawValue ?? "general",
        "goal_id": userMessage.goalId?.uuidString as Any,
        "thread_id": currentThread?.threadId as Any
    ]

    let response = try await SupabaseManager.shared
        .functions
        .invoke("openai-chat-completion", parameters: requestBody)

    let data = try JSONSerialization.data(withJSONObject: response)
    return try JSONDecoder().decode(OpenAIChatResponse.self, from: data)
}

struct OpenAIChatResponse: Codable {
    let response: String
    let threadId: String
    let tokensUsed: Int
    let modelUsed: String
    let costEstimate: Double

    enum CodingKeys: String, CodingKey {
        case response
        case threadId = "thread_id"
        case tokensUsed = "tokens_used"
        case modelUsed = "model_used"
        case costEstimate = "cost_estimate"
    }
}

```

### 2. Enhanced `ChatRepository.swift`

**Update thread management methods:**

```swift
// MARK: - Thread Management (Enhanced for OpenAI)

func getOrCreateThread(
    for userId: UUID,
    assistantType: UserAssistantThread.AssistantType
) async throws -> UserAssistantThread {
    let requestBody: [String: Any] = [
        "action": "get_or_create",
        "user_id": userId.uuidString,
        "assistant_type": assistantType.rawValue
    ]

    let response = try await supabase
        .functions
        .invoke("thread-manager", parameters: requestBody)

    let data = try JSONSerialization.data(withJSONObject: response)
    return try JSONDecoder().decode(UserAssistantThread.self, from: data)
}

func updateThread(_ thread: UserAssistantThread) async throws {
    let requestBody: [String: Any] = [
        "action": "update_interaction",
        "thread_id": thread.threadId
    ]

    _ = try await supabase
        .functions
        .invoke("thread-manager", parameters: requestBody)
}

func fetchThreads(for userId: UUID) async throws -> [UserAssistantThread] {
    let requestBody: [String: Any] = [
        "action": "get_threads",
        "user_id": userId.uuidString
    ]

    let response = try await supabase
        .functions
        .invoke("thread-manager", parameters: requestBody)

    let data = try JSONSerialization.data(withJSONObject: response)
    return try JSONDecoder().decode([UserAssistantThread].self, from: data)
}

```

---

## 🗃️ Database Schema Enhancements

### 1. Add OpenAI-specific columns to existing tables

```sql
-- Enhance user_assistant_threads table
ALTER TABLE user_assistant_threads
ADD COLUMN openai_assistant_id TEXT,
ADD COLUMN total_tokens_used INTEGER DEFAULT 0,
ADD COLUMN last_cost_estimate DECIMAL(10,6) DEFAULT 0;

-- Enhance journal_ai_chats table for OpenAI tracking
ALTER TABLE journal_ai_chats
ADD COLUMN openai_thread_id TEXT,
ADD COLUMN tokens_used INTEGER,
ADD COLUMN model_used TEXT,
ADD COLUMN response_time_ms INTEGER;

```

### 2. Create OpenAI usage tracking table

```sql
CREATE TABLE openai_usage_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    thread_id TEXT NOT NULL,
    message_id UUID REFERENCES journal_ai_chats(id) ON DELETE CASCADE,
    prompt_tokens INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    total_tokens INTEGER NOT NULL,
    model_used TEXT NOT NULL,
    cost_estimate DECIMAL(10,6),
    response_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index for cost tracking and analytics
CREATE INDEX idx_openai_usage_user_date ON openai_usage_logs(user_id, created_at);
CREATE INDEX idx_openai_usage_cost ON openai_usage_logs(cost_estimate, created_at);

```

### 3. Create context cache table (optional optimization)

```sql
CREATE TABLE user_context_cache (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    context_hash TEXT NOT NULL,
    context_data TEXT NOT NULL,
    chat_type TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL
);

-- Index for fast context lookup
CREATE UNIQUE INDEX idx_user_context_hash ON user_context_cache(user_id, context_hash);

```

---

## 🔗 API Integration Details

### Assistant Configuration

```tsx
// Edge Function configuration
const ASSISTANT_CONFIGS = {
  'assistant_coach': 'asst_general123',      // Created in OpenAI dashboard
  'onboarding_coach': 'asst_pillar456',       // Created in OpenAI dashboard
}

function getAssistantId(chatType: string): string {
  switch (chatType) {
    case 'pillar_health':
    case 'pillar_emotion':
    case 'pillar_purpose':
    case 'pillar_connection':
      return ASSISTANT_CONFIGS.pillar_coach
    case 'journal_reflection':
      return ASSISTANT_CONFIGS.journal_coach
    default:
      return ASSISTANT_CONFIGS.general_coach
  }
}

```

### Error Handling Strategy

```swift
// In SupabaseChatViewModel.swift
private func callOpenAIChatCompletion(for userMessage: JournalAIChat) async throws -> OpenAIChatResponse {
    let maxRetries = 3
    var lastError: Error?

    for attempt in 1...maxRetries {
        do {
            return try await performOpenAICall(for: userMessage)
        } catch {
            lastError = error
            print("❌ OpenAI call attempt \\(attempt) failed: \\(error)")

            // Exponential backoff
            if attempt < maxRetries {
                try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(attempt)) * 1_000_000_000))
            }
        }
    }

    throw lastError ?? OpenAIError.maxRetriesExceeded
}

enum OpenAIError: Error {
    case maxRetriesExceeded
    case invalidResponse
    case rateLimited
}

```

---

## 🎯 User Experience Requirements

### 1. Seamless Integration

- **No UI Changes**: Current chat interface remains identical
- **Loading States**: Existing loading indicators work unchanged
- **Error Handling**: Graceful fallback to simulated responses
- **Performance**: Response time should be < 5 seconds average

### 2. Progressive Enhancement

- **Feature Flag**: Add configuration to enable/disable OpenAI per user
- **A/B Testing**: Support for comparing OpenAI vs simulated responses
- **Rollback**: Easy way to disable OpenAI and revert to simulated responses

### 3. Monitoring & Observability

- **Usage Tracking**: Log all OpenAI API calls for cost monitoring
- **Performance Metrics**: Track response times and success rates
- **Quality Metrics**: Enable user feedback on response quality

```swift
// Add to SupabaseConfig.plist
<key>OpenAIEnabled</key>
<boolean>true</boolean>
<key>OpenAIFallbackEnabled</key>
<boolean>true</boolean>
<key>OpenAIMaxResponseTime</key>
<integer>5000</integer>

```

---

## 🚀 Implementation Checklist

### Phase 1: Foundation

- [ ]  Create OpenAI assistants in OpenAI dashboard
- [ ]  Set up Edge Functions in Supabase project
- [ ]  Add database schema enhancements
- [ ]  Configure environment variables

### Phase 2: Edge Functions

- [ ]  Implement `openai-chat-completion` function
- [ ]  Implement `user-context-builder` function
- [ ]  Implement `thread-manager` function
- [ ]  Add error handling and logging

### Phase 3: Swift Integration

- [ ]  Update `SupabaseChatViewModel.getAIResponse()`
- [ ]  Update `ChatRepository` thread methods
- [ ]  Add OpenAI response models
- [ ]  Implement error handling and fallbacks

### Phase 4: Testing & Deployment

- [ ]  Test with real OpenAI API
- [ ]  Verify fallback mechanisms
- [ ]  Monitor performance and costs
- [ ]  Deploy with feature flags

---

## 📊 Testing Strategy

### Unit Tests

```swift
class OpenAIChatIntegrationTests: XCTestCase {
    func testOpenAIResponseIntegration() async {
        // Test successful OpenAI API call
        // Test fallback to simulated response on error
        // Test thread management
    }

    func testContextBuilding() async {
        // Test user context aggregation
        // Test different chat types
        // Test goal-specific context
    }
}

```

### Integration Tests

- Test Edge Functions locally with Supabase CLI
- Test full user conversation flow
- Test error scenarios and fallbacks
- Test cost limits and rate limiting

---

This technical specification provides developers with concrete implementation requirements while preserving your existing UI/UX and database architecture. The OpenAI integration will be transparent to users and provide intelligent responses through your current chat interface.