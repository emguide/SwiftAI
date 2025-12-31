# OpenAI-Compatible Backend

This backend supports any provider that implements the OpenAI Chat Completions API specification.

## Checking Provider Capabilities

Use `ProviderCapabilities` to check what features a provider reliably supports before making requests:

```swift
let provider = OpenAICompatibleLLM.Provider.gemini()

// Check capabilities programmatically
if !provider.capabilities.supportsToolsWithStructuredOutput {
    // Use tools OR structured output, not both
}

if provider.capabilities.minimumTokens > requestedTokens {
    // Increase token limit to avoid freezing
}

// Available capability flags:
// - supportsToolsWithStructuredOutput
// - supportsPreseededToolHistory
// - supportsMultiTurnToolLoops
// - supportsMultiToolSelection
// - supportsGuideConstraints
// - minimumTokens
```

Untested providers (Groq, Together, Custom) use `ProviderCapabilities.conservative` defaults.

## Provider-Aware Errors

When a request fails due to provider limitations, SwiftAI throws descriptive errors with actionable suggestions:

```swift
do {
    let reply = try await gemini.reply(to: prompt, returning: T.self, tools: tools)
} catch let error as LLMError {
    switch error {
    case .unsupportedConfiguration(let provider, let feature, let suggestion):
        // provider: "Gemini"
        // feature: "tools + structured output"
        // suggestion: "Use tools OR structured output, not both"
        print(error.localizedDescription)  // "Gemini does not support tools + structured output"
        print(error.recoverySuggestion!)   // "Use tools OR structured output, not both"

    case .minimumTokensRequired(let provider, let minimum, let requested):
        // provider: "DeepSeek"
        // minimum: 16
        // requested: 1
        print(error.localizedDescription)  // "DeepSeek requires at least 16 tokens (requested: 1)"

    default:
        break
    }
}
```

## Supported Providers

| Provider | Environment Variable | Base URL |
|----------|---------------------|----------|
| Gemini | `GEMINI_API_KEY` | `https://generativelanguage.googleapis.com/v1beta/openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `https://api.deepseek.com/v1` |
| Grok | `XAI_API_KEY` | `https://api.x.ai/v1` |
| Groq | `GROQ_API_KEY` | `https://api.groq.com/openai/v1` |
| Together | `TOGETHER_API_KEY` | `https://api.together.xyz/v1` |
| Custom | (explicit) | (explicit) |

## Provider Limitations

### Gemini (Tested)

Gemini's OpenAI-compatible endpoint has several limitations compared to native OpenAI:

| Feature | Status | Notes |
|---------|--------|-------|
| Basic text generation | ✅ Works | |
| Streaming | ✅ Works | May emit duplicate final partials |
| Structured output | ✅ Works | |
| Tool calling | ✅ Works | Basic single-tool and multi-tool selection |
| Tools + Structured output | ❌ Avoid | Causes infinite tool call loops |
| Multi-turn tool loops | ⚠️ Unreliable | May not call all required tools in sequence |
| Complex pre-seeded history | ❌ Avoid | Returns empty response for histories with tool calls |
| Max tokens = 1 | ❌ Avoid | Provider may require minimum tokens |

**Recommendations for Gemini:**
- Do not combine `tools` with structured output (`returning:` parameter)
- For multi-step tool workflows, consider breaking into separate requests
- Avoid pre-seeding conversation history with tool call/output messages
- Use `.relaxed` parsing options (automatically applied)

### DeepSeek (Tested)

DeepSeek's OpenAI-compatible endpoint has good compatibility but some limitations:

| Feature | Status | Notes |
|---------|--------|-------|
| Basic text generation | ✅ Works | |
| Streaming | ✅ Works | May emit duplicate partials |
| Structured output | ✅ Works | Uses `json_object` mode (not `json_schema`) |
| Tool calling | ✅ Works | Basic single-tool selection |
| Multiple tool selection | ⚠️ Unreliable | May not select correct tool |
| Multi-turn tool loops | ✅ Works | |
| Tools + Structured output | ✅ Works | |
| @Guide constraints | ⚠️ Unreliable | May not respect array count constraints |
| Max tokens = 1 | ❌ Avoid | May freeze |

**Technical Notes:**
- DeepSeek only supports `json_object` response format, not `json_schema`
- The backend automatically injects a system message with schema when using structured output
- DeepSeek requires "json" keyword in the prompt for `json_object` mode (handled automatically)

**Recommendations for DeepSeek:**
- Use single tools when possible; multi-tool selection may be unreliable
- Don't rely on strict @Guide constraints for array counts
- Avoid very low max token values

### Grok (Tested)

Grok's OpenAI-compatible endpoint has good compatibility with some limitations:

| Feature | Status | Notes |
|---------|--------|-------|
| Basic text generation | ✅ Works | |
| Streaming | ✅ Works | May emit duplicate partials |
| Structured output | ✅ Works | |
| Tool calling | ✅ Works | Basic single-tool and multi-tool selection |
| Tools + Structured output | ❌ Avoid | Returns 400 error |
| Multi-turn tool loops | ✅ Works | |
| Complex pre-seeded history | ❌ Avoid | Returns 400 error for histories with tool calls |
| @Guide constraints | ⚠️ Unreliable | May not respect array count constraints |

**Recommendations for Grok:**
- Do not combine `tools` with structured output (`returning:` parameter)
- Avoid pre-seeding conversation history with tool call/output messages
- Don't rely on strict @Guide constraints for array counts


## API Patterns to Avoid

### 1. Tools + Structured Output (Gemini)

```swift
// ❌ AVOID with Gemini - causes infinite loop
let reply = try await gemini.reply(
    to: "Calculate 10 * 5",
    returning: CalculationResult.self,  // Structured output
    tools: [calculatorTool]              // + Tools = infinite loop
)

// ✅ OK - Use tools without structured output
let reply = try await gemini.reply(
    to: "Calculate 10 * 5",
    tools: [calculatorTool]
)

// ✅ OK - Use structured output without tools
let reply = try await gemini.reply(
    to: "Format this as JSON",
    returning: MyStruct.self
)
```

### 2. Pre-seeded Tool History (Gemini)

```swift
// ❌ AVOID with Gemini - returns empty response
let messages: [Message] = [
    .user(.init(text: "Calculate 5 + 3")),
    .ai(.init(chunks: [...], toolCalls: [...])),  // Pre-seeded tool call
    .toolOutput(.init(...)),                       // Pre-seeded tool output
    .user(.init(text: "Now do something else"))
]
let reply = try await gemini.reply(to: messages, returning: T.self)

// ✅ OK - Let the model generate tool calls naturally
let session = gemini.makeSession(tools: [calculatorTool])
let reply = try await gemini.reply(to: "Calculate 5 + 3", in: session)
```

### 3. Very Low Max Tokens

```swift
// ❌ AVOID - Provider may require minimum tokens
let options = LLMReplyOptions(maximumTokens: 1)

// ✅ OK - Use reasonable minimum (16+ for most providers)
let options = LLMReplyOptions(maximumTokens: 50)
```

## Rate Limits

### Gemini Free Tier

Gemini free tier has a 15 RPM (requests per minute) limit. For testing, enable delays:

```swift
// In test file
private let slowTestsForFreeTier = true
private let freeTierDelaySeconds: UInt64 = 5
```

## Adding New Providers

When adding support for a new provider:

1. Add the provider case to `OpenAICompatibleLLM.Provider`
2. Configure `baseURL`, `apiKey`, `name`, and `parsingOptions`
3. Run the full test suite against the provider
4. Update `capabilities` in `OpenAICompatibleLLM.swift` based on test results
5. Add tests to `ProviderCapabilitiesTests.swift` for the new provider
6. Document any limitations in this file
7. Disable failing tests with descriptive messages
