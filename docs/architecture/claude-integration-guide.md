# Claude API 集成完整技术文档

> **版本**: v1.0 | **日期**: 2024-11-23 | **作者**: Snow AI | **项目**: [New-API](https://github.com/Calcium-Ion/new-api)

---

## 📋 目录

- [概述](#概述)
- [架构设计](#架构设计)
- [核心组件](#核心组件)
- [请求转换流程](#请求转换流程)
- [响应处理流程](#响应处理流程)
- [高级功能](#高级功能)
- [支持的模型](#支持的模型)
- [完整调用流程](#完整调用流程)

---

## 🎯 概述

new-api 通过**适配器模式**实现对 Anthropic Claude API 的完整支持,能够将 OpenAI 格式请求转换为 Claude API 格式,并将 Claude 响应转换回 OpenAI 格式。

### 主要特性

✅ **双向格式转换** - OpenAI ↔ Claude  
✅ **流式/非流式** - SSE流式 + 标准响应  
✅ **多模态支持** - 文本、图片、工具调用  
✅ **思维模式** - Extended Thinking  
✅ **缓存机制** - Prompt Caching  
✅ **工具调用** - Function Calling + Web Search  
✅ **版本兼容** - Completion API + Messages API

---

## 🏗️ 架构设计

### 整体架构流程

```
Client (OpenAI Format)
       ↓
Claude Adaptor
 ├─ Init() - 初始化,选择API模式
 ├─ ConvertOpenAIRequest() - 请求转换
 ├─ GetRequestURL() - 构建URL
 ├─ SetupRequestHeader() - 设置请求头
 ├─ DoRequest() - 发送HTTP请求
 └─ DoResponse() - 处理响应
       ├─ Stream → ClaudeStreamHandler()
       └─ Non-Stream → ClaudeHandler()
       ↓
Claude API (Anthropic)
 ├─ /v1/messages (Messages API)
 └─ /v1/complete (Completion API)
       ↓
Response Conversion
 ├─ ResponseClaude2OpenAI()
 └─ StreamResponseClaude2OpenAI()
       ↓
Client (OpenAI Format)
```

### 文件结构

```
relay/channel/claude/
├── adaptor.go          # 适配器核心逻辑
├── relay-claude.go     # 请求/响应转换
├── constants.go        # 常量定义

dto/
└── claude.go           # Claude API数据结构
```

---

## 🔧 核心组件

### 1. Adaptor 结构体

```go
type Adaptor struct {
    RequestMode int  // 1=Completion, 2=Message
}
```

| 模式 | 值 | API端点 | 适用模型 |
|------|---|--------|----------|
| RequestModeCompletion | 1 | `/v1/complete` | claude-2.x, claude-instant |
| RequestModeMessage | 2 | `/v1/messages` | claude-3.x及以上 |

### 2. 核心方法

#### Init() - 初始化

```go
func (a *Adaptor) Init(info *relaycommon.RelayInfo) {
    if strings.HasPrefix(info.UpstreamModelName, "claude-2") || 
       strings.HasPrefix(info.UpstreamModelName, "claude-instant") {
        a.RequestMode = RequestModeCompletion
    } else {
        a.RequestMode = RequestModeMessage
    }
}
```

**作用**: 根据模型名称自动选择API模式

#### GetRequestURL() - 构建请求URL

```go
func (a *Adaptor) GetRequestURL(info *relaycommon.RelayInfo) (string, error) {
    baseURL := ""
    if a.RequestMode == RequestModeMessage {
        baseURL = fmt.Sprintf("%s/v1/messages", info.ChannelBaseUrl)
    } else {
        baseURL = fmt.Sprintf("%s/v1/complete", info.ChannelBaseUrl)
    }
    if info.IsClaudeBetaQuery {
        baseURL = baseURL + "?beta=true"
    }
    return baseURL, nil
}
```

**URL示例**:
- Messages: `https://api.anthropic.com/v1/messages`
- Completion: `https://api.anthropic.com/v1/complete`
- Beta: `https://api.anthropic.com/v1/messages?beta=true`

#### SetupRequestHeader() - 设置请求头

| Header | 必需 | 说明 | 示例 |
|--------|------|------|------|
| x-api-key | ✅ | API密钥 | `sk-ant-api03-...` |
| anthropic-version | ✅ | API版本 | `2023-06-01` |
| anthropic-beta | ❌ | Beta功能 | `prompt-caching-2024-07-31` |
| content-type | ✅ | 内容类型 | `application/json` |

---

## 🔄 请求转换流程

### 1. OpenAI → Claude Messages API

**函数**: `RequestOpenAI2ClaudeMessage()`

#### 消息角色映射

| OpenAI | Claude | 处理方式 |
|--------|--------|----------|
| system | system字段 | 提取到system数组 |
| user | user | 直接映射 |
| assistant | assistant | 直接映射 |
| tool | user | 合并到上一个user消息 |

#### 转换示例

**OpenAI格式**:
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [
    {"role": "system", "content": "You are helpful."},
    {"role": "user", "content": "Hello!"}
  ]
}
```

**Claude格式**:
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "system": [{"type": "text", "text": "You are helpful."}],
  "messages": [{"role": "user", "content": "Hello!"}],
  "max_tokens": 4096
}
```

#### 多模态处理

**图片转换**:
```go
// URL图片 → Base64
if strings.HasPrefix(imageUrl.Url, "http") {
    fileData, _ := service.GetFileBase64FromUrl(c, imageUrl.Url, "...")
    claudeMediaMessage.Source = &dto.ClaudeMessageSource{
        Type:      "base64",
        MediaType: fileData.MimeType,
        Data:      fileData.Base64Data,
    }
}
```

**支持格式**: JPEG, PNG, GIF, WebP

#### 工具调用转换

**OpenAI Function → Claude Tool**:
```json
// OpenAI
{"tools": [{
  "type": "function",
  "function": {
    "name": "get_weather",
    "parameters": {"type": "object", "properties": {...}}
  }
}]}

// Claude
{"tools": [{
  "name": "get_weather",
  "input_schema": {"type": "object", "properties": {...}}
}]}
```

#### Tool Choice映射

| OpenAI | Claude | 说明 |
|--------|--------|------|
| "auto" | {"type": "auto"} | 模型自主决定 |
| "required" | {"type": "any"} | 必须调用工具 |
| "none" | {"type": "none"} | 不调用工具 |

### 2. OpenAI → Claude Completion API

**函数**: `RequestOpenAI2ClaudeComplete()`

```go
prompt := ""
for _, message := range textRequest.Messages {
    if message.Role == "user" {
        prompt += fmt.Sprintf("\n\nHuman: %s", message.StringContent())
    } else if message.Role == "assistant" {
        prompt += fmt.Sprintf("\n\nAssistant: %s", message.StringContent())
    } else if message.Role == "system" {
        if prompt == "" {
            prompt = message.StringContent()
        }
    }
}
prompt += "\n\nAssistant:"
```

**格式示例**:
```
You are helpful.  
} 
} 
```

**支持格式**: JPEG, PNG, GIF, WebP

---

## 📤 响应处理流程

### 1. Claude → OpenAI 非流式响应

**函数**: `ResponseClaude2OpenAI()`

#### Stop Reason映射

| Claude | OpenAI | 说明 |
|--------|--------|------|
| stop_sequence | stop | 遇到停止序列 |
| end_turn | stop | 自然结束 |
| max_tokens | length | 达到最大token |
| tool_use | tool_calls | 工具调用 |

#### 响应转换示例

**Claude响应**:
```json
{
  "id": "msg_123",
  "type": "message",
  "model": "claude-3-5-sonnet-20241022",
  "content": [{"type": "text", "text": "Hello!"}],
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 10, "output_tokens": 5}
}
```

**OpenAI响应**:
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "claude-3-5-sonnet-20241022",
  "choices": [{
    "index": 0,
    "message": {"role": "assistant", "content": "Hello!"},
    "finish_reason": "stop"
  }],
  "usage": {"prompt_tokens": 10, "completion_tokens": 5, "total_tokens": 15}
}
```

### 2. Claude → OpenAI 流式响应

**函数**: `StreamResponseClaude2OpenAI()`

#### 流式事件类型

| Claude Event | OpenAI Delta | 说明 |
|-------------|--------------|------|
| message_start | delta with role | 开始消息,设置role |
| content_block_start | delta with content/tool_calls | 开始内容块 |
| content_block_delta | delta with content | 内容增量 |
| message_delta | finish_reason | 消息结束 |
| message_stop | N/A | 流结束(不发送) |

#### 流式转换示例

**Claude流式事件**:
```json
{"type": "message_start", "message": {"id": "msg_123", "model": "claude-3-5-sonnet", "usage": {...}}}
{"type": "content_block_start", "content_block": {"type": "text", "text": ""}}
{"type": "content_block_delta", "delta": {"type": "text_delta", "text": "Hello"}}
{"type": "content_block_delta", "delta": {"type": "text_delta", "text": " World"}}
{"type": "message_delta", "delta": {"stop_reason": "end_turn"}, "usage": {...}}
{"type": "message_stop"}
```

**OpenAI流式响应**:
```json
{"choices": [{"index": 0, "delta": {"role": "assistant", "content": ""}}]}
{"choices": [{"index": 0, "delta": {"content": "Hello"}}]}
{"choices": [{"index": 0, "delta": {"content": " World"}}]}
{"choices": [{"index": 0, "delta": {}, "finish_reason": "stop"}]}
[DONE]
```

---

## 🚀 高级功能

### 1. Extended Thinking (思维模式)

#### 启用方式

**方法1: 模型名称后缀**
```json
{
  "model": "claude-3-7-sonnet-20250219-thinking"
}
```

**方法2: reasoning_effort参数**
```json
{
  "model": "claude-3-7-sonnet-20250219",
  "reasoning_effort": "high"
}
```

**方法3: reasoning参数**
```json
{
  "model": "claude-3-7-sonnet-20250219",
  "reasoning": {"max_tokens": 4096}
}
```

#### 实现逻辑

```go
// 检测 -thinking 后缀
if strings.HasSuffix(textRequest.Model, "-thinking") {
    claudeRequest.Thinking = &dto.Thinking{
        Type:         "enabled",
        BudgetTokens: &budgetTokens,
    }
    // 设置必需参数
    claudeRequest.TopP = 0
    claudeRequest.Temperature = &1.0
}

// reasoning_effort 映射
switch textRequest.ReasoningEffort {
case "low":    budgetTokens = 1280
case "medium": budgetTokens = 2048
case "high":   budgetTokens = 4096
}
```

### 2. Prompt Caching

**启用方式**: 设置 `anthropic-beta` 头
```
anthropic-beta: prompt-caching-2024-07-31
```

**缓存使用情况**:
```go
claudeInfo.Usage.PromptTokensDetails.CachedTokens = response.Usage.CacheReadInputTokens
claudeInfo.Usage.PromptTokensDetails.CachedCreationTokens = response.Usage.CacheCreationInputTokens
```

### 3. Web Search Tool

```go
// OpenAI格式
{
  "web_search_options": {
    "user_location": {"approximate": {"timezone": "UTC", "country": "US"}},
    "search_context_size": "medium"
  }
}

// 转换为Claude格式
{
  "tools": [{
    "type": "web_search_20250305",
    "name": "web_search",
    "max_uses": 5,  // low=1, medium=5, high=10
    "user_location": {
      "type": "approximate",
      "timezone": "UTC",
      "country": "US"
    }
  }]
}
```

---

## 📋 支持的模型

**Messages API模型** (RequestModeMessage):

| 模型系列 | 模型ID |
|---------|--------|
| **Claude 3 Haiku** | claude-3-haiku-20240307 |
| **Claude 3 Sonnet** | claude-3-sonnet-20240229 |
| **Claude 3 Opus** | claude-3-opus-20240229 |
| **Claude 3.5 Haiku** | claude-3-5-haiku-20241022 |
| **Claude 3.5 Sonnet** | claude-3-5-sonnet-20240620<br>claude-3-5-sonnet-20241022 |
| **Claude 3.7 Sonnet** | claude-3-7-sonnet-20250219<br>claude-3-7-sonnet-20250219-thinking |
| **Claude Sonnet 4** | claude-sonnet-4-20250514<br>claude-sonnet-4-20250514-thinking |
| **Claude Opus 4** | claude-opus-4-20250514<br>claude-opus-4-20250514-thinking |
| **Claude Opus 4.1** | claude-opus-4-1-20250805<br>claude-opus-4-1-20250805-thinking |
| **Claude Sonnet 4.5** | claude-sonnet-4-5-20250929<br>claude-sonnet-4-5-20250929-thinking |

**Completion API模型** (RequestModeCompletion):
- claude-instant-1.2
- claude-2, claude-2.0, claude-2.1

---

## 🔍 完整调用流程

### 时序图

```
Client                Adaptor              Claude API
  │                      │                      │
  │─────Request──────────>│                      │
  │  (OpenAI Format)      │                      │
  │                       │                      │
  │                       │────Init()            │
  │                       │   选择API模式         │
  │                       │                      │
  │                       │────ConvertRequest    │
  │                       │   OpenAI→Claude      │
  │                       │                      │
  │                       │────GetRequestURL     │
  │                       │   构建URL            │
  │                       │                      │
  │                       │────SetupHeaders      │
  │                       │   设置请求头          │
  │                       │                      │
  │                       │──────HTTP POST───────>│
  │                       │   (Claude Format)     │
  │                       │                      │
  │                       │<─────Response─────────│
  │                       │   (Claude Format)     │
  │                       │                      │
  │                       │────DoResponse        │
  │                       │   处理响应            │
  │                       │                      │
  │                       │────ConvertResponse   │
  │                       │   Claude→OpenAI      │
  │                       │                      │
  │<────Response──────────│                      │
  │  (OpenAI Format)      │                      │
```

### 代码调用链

```
1. client.Request (OpenAI format)
   ↓
2. adaptor.Init(info)
   - 判断模型类型
   - 设置 RequestMode
   ↓
3. adaptor.ConvertOpenAIRequest(request)
   - RequestModeMessage → RequestOpenAI2ClaudeMessage()
   - RequestModeCompletion → RequestOpenAI2ClaudeComplete()
   ↓
4. adaptor.GetRequestURL(info)
   - Messages: /v1/messages
   - Completion: /v1/complete
   ↓
5. adaptor.SetupRequestHeader()
   - x-api-key
   - anthropic-version
   - anthropic-beta
   ↓
6. adaptor.DoRequest()
   - channel.DoApiRequest()
   ↓
7. Claude API Processing
   ↓
8. adaptor.DoResponse(response)
   - Stream → ClaudeStreamHandler()
     - HandleStreamResponseData()
     - StreamResponseClaude2OpenAI()
   - Non-Stream → ClaudeHandler()
     - HandleClaudeResponseData()
     - ResponseClaude2OpenAI()
   ↓
9. client.Response (OpenAI format)
```

---

## 💡 技术细节

### Token计费

```go
// 输入token
usage.PromptTokens = claudeResponse.Usage.InputTokens

// 缓存相关
usage.PromptTokensDetails.CachedTokens = claudeResponse.Usage.CacheReadInputTokens
usage.PromptTokensDetails.CachedCreationTokens = claudeResponse.Usage.CacheCreationInputTokens
usage.ClaudeCacheCreation5mTokens = claudeResponse.Usage.CacheCreation.Ephemeral5mInputTokens
usage.ClaudeCacheCreation1hTokens = claudeResponse.Usage.CacheCreation.Ephemeral1hInputTokens

// 输出token
usage.CompletionTokens = claudeResponse.Usage.OutputTokens

// 总计
usage.TotalTokens = usage.PromptTokens + usage.CompletionTokens
```

### Web Search计费

```go
// Claude返回搜索请求次数
if claudeResponse.Usage.ServerToolUse != nil {
    c.Set("claude_web_search_requests", claudeResponse.Usage.ServerToolUse.WebSearchRequests)
}
```

### 错误处理

```go
// Claude错误类型
type ClaudeError struct {
    Type    string `json:"type"`
    Message string `json:"message"`
}

// 常见错误类型
- invalid_request_error
- authentication_error
- permission_error
- not_found_error
- rate_limit_error
- api_error
- overloaded_error
```

---

## ✅ 最佳实践

### 1. 模型选择建议

| 场景 | 推荐模型 | 原因 |
|------|---------|------|
| 快速响应 | claude-3-5-haiku | 速度最快,成本最低 |
| 通用对话 | claude-3-5-sonnet | 平衡性能与成本 |
| 复杂推理 | claude-opus-4 | 最强推理能力 |
| 深度思考 | claude-*-thinking | 支持扩展思维 |

### 2. 参数优化

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 4096,        // 推荐: 1024-8192
  "temperature": 1.0,        // 推荐: 0.7-1.0
  "top_p": 0.9,              // 推荐: 0.9-1.0
  "top_k": 40                // 推荐: 10-50 (可选)
}
```

### 3. 缓存优化

- 将常用的system prompt标记为可缓存
- 使用 `anthropic-beta: prompt-caching-2024-07-31`
- 缓存有效期: 5分钟或1小时

### 4. 错误重试策略

```
Rate Limit (429) → 等待后重试
Overloaded (529) → 指数退避重试
API Error (500) → 最多重试3次
Invalid Request (400) → 不重试,修正请求
```

---

## 📝 总结

new-api 的 Claude 集成实现了:

1. **完整的格式转换**: OpenAI ↔ Claude双向转换
2. **智能模式选择**: 自动识别模型版本,选择合适API
3. **丰富的功能支持**: 多模态、工具调用、思维模式、缓存
4. **健壮的错误处理**: 完善的错误映射和重试机制
5. **灵活的配置**: 支持Beta功能、自定义头部

通过适配器模式,new-api 成功将 Claude 的强大能力整合到统一的API接口中,为开发者提供了便捷的多模型切换能力。

---

**相关文档**:
- [Claude API Official Docs](https://docs.anthropic.com/)
- [New-API GitHub](https://github.com/Calcium-Ion/new-api)
- [Gemini集成指南](./gemini-integration-guide.md)
