# OpenAI API 集成完整技术文档

> **版本**: v1.0 | **日期**: 2025-01-25 | **作者**: Snow AI | **项目**: [New-API](https://github.com/Calcium-Ion/new-api)

---

## 📋 目录

- [概述](#概述)
- [架构设计](#架构设计)
- [核心组件](#核心组件)
- [请求处理流程](#请求处理流程)
- [响应处理流程](#响应处理流程)
- [高级功能](#高级功能)
- [多渠道支持](#多渠道支持)
- [完整调用流程](#完整调用流程)
- [配置与部署](#配置与部署)

---

## 🎯 概述

new-api 项目通过**适配器模式**实现对 OpenAI API 的完整支持，作为整个系统的基础架构模式。OpenAI Adaptor 不仅支持标准 OpenAI API，还兼容 Azure OpenAI、OpenRouter 等多种衍生服务。

### 主要特性

✅ **多格式兼容** - OpenAI / Claude / Gemini 格式互转  
✅ **全模态支持** - 文本、语音、图像、实时通信  
✅ **流式处理** - SSE 流式响应 + WebSocket 实时通信  
✅ **推理模型** - O1/O3/GPT-5 系列推理模型支持  
✅ **多渠道架构** - 负载均衡、自动故障转移、重试机制  
✅ **缓存计费** - Cache Billing (Azure/DeepSeek/Qwen)  
✅ **企业级功能** - 配额管理、多租户、审计日志

### 支持的 API 类型

| API 类型 | 端点 | 说明 |
|---------|------|-----|
| Chat Completions | `/v1/chat/completions` | 聊天补全（流式/非流式） |
| Embeddings | `/v1/embeddings` | 向量嵌入 |
| Audio Speech | `/v1/audio/speech` | 文本转语音 (TTS) |
| Audio Transcriptions | `/v1/audio/transcriptions` | 语音转文本 (STT) |
| Audio Translations | `/v1/audio/translations` | 语音翻译 |
| Realtime | `/v1/realtime` | WebSocket 实时通信 |
| Images | `/v1/images/generations` | 图像生成 |
| Moderations | `/v1/moderations` | 内容审核 |

---

## 🏗️ 架构设计

### 整体架构流程

```
Client Request
       ↓
[Router Layer] - router/relay-router.go
  ├─ /v1/chat/completions
  ├─ /v1/embeddings
  ├─ /v1/audio/*
  └─ /v1/realtime
       ↓
[Middleware Stack]
  ├─ CORS
  ├─ TokenAuth (API密钥验证)
  ├─ ModelRequestRateLimit (限流)
  └─ Distribute (渠道选择)
       ↓
[Controller] - controller/relay.go
  ├─ Relay(c *gin.Context, relayFormat)
  ├─ 解析请求体
  ├─ 验证Token/配额
  └─ 调用 relayHandler()
       ↓
[Relay Handler]
  ├─ GetAdaptor(apiType) - 获取适配器
  ├─ 初始化 RelayInfo 上下文
  └─ 路由到特定模式处理器
       ↓
[OpenAI Adaptor] - relay/channel/openai/adaptor.go
  ├─ ConvertOpenAIRequest() - 请求转换
  ├─ GetRequestURL() - 构建目标URL
  ├─ SetupRequestHeader() - 添加认证头
  └─ DoRequest() - 执行HTTP调用
       ↓
[External OpenAI API]
       ↓
[Response Handler] - relay/channel/openai/relay-openai.go
  ├─ DoResponse() - 路由到响应处理器
  ├─ 处理流式/非流式响应
  ├─ Token计数（计费）
  └─ 返回格式化响应
       ↓
Client receives response
```

### 文件结构

```
relay/channel/openai/
├── adaptor.go              # 核心适配器（请求转换、URL构建、认证）
├── relay-openai.go         # 响应处理器（流式/非流式）
├── audio.go                # 音频API处理（TTS/STT）
├── realtime.go             # WebSocket实时通信
└── token.go                # Token计数工具

controller/
└── relay.go                # 主控制器（请求分发、渠道选择）

middleware/
├── auth.go                 # Token认证中间件
├── distributor.go          # 渠道分配与负载均衡
└── rate-limit.go           # 限流中间件

router/
└── relay-router.go         # 路由定义

service/
├── convert.go              # 格式转换（OpenAI ↔ Claude ↔ Gemini）
└── token_counter.go        # Token计数服务（tiktoken）

model/
├── channel.go              # 渠道数据模型
└── token.go                # Token数据模型

dto/
├── openai.go               # OpenAI请求/响应DTO
├── claude.go               # Claude DTO
└── gemini.go               # Gemini DTO
```

---

## 🔧 核心组件

### 1. Adaptor 结构体

**文件**: `relay/channel/openai/adaptor.go`

```go
type Adaptor struct {
    ChannelType    int    // 渠道类型（OpenAI/Azure/OpenRouter）
    ResponseFormat string // 响应格式强制转换
}
```

**核心功能**:
- 实现 `channel.Adaptor` 接口
- 支持多种请求格式（OpenAI/Claude/Gemini）自动识别与转换
- 处理不同模型的特殊需求（O系列、GPT-4、Azure特定配置）
- 统一的错误处理与重试逻辑

### 2. 核心方法详解

#### 2.1 Init() - 初始化

```go
func (a *Adaptor) Init(info *relaycommon.RelayInfo) {}
```

**功能**:
- 设置渠道类型（OpenAI、Azure、OpenRouter等）
- 根据渠道配置初始化响应格式转换选项

---

#### 2.2 ConvertOpenAIRequest() - 请求转换

**文件**: `relay/channel/openai/adaptor.go`

```go
func (a *Adaptor) ConvertOpenAIRequest(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest) error
```

**转换逻辑**:

##### A. O系列模型特殊处理（O1/O3/GPT-5）

```go
if strings.HasPrefix(info.UpstreamModelName, "o") {
    // 1. max_tokens → max_completion_tokens
    if request.MaxCompletionTokens == 0 && request.MaxTokens != 0 {
        request.MaxCompletionTokens = request.MaxTokens
        request.MaxTokens = 0
    }
    
    // 2. 禁用temperature（O系列不支持）
    request.Temperature = nil
    
    // 3. system → developer（较新O系列模型）
    if request.Messages[0].Role == "system" {
        request.Messages[0].Role = "developer"
    }
    
    // 4. 解析推理强度（从模型名后缀）
    // 例如: "o1-mini-high" → reasoning_effort: "high"
    if strings.Contains(info.UpstreamModelName, "-high") {
        request.ReasoningEffort = "high"
    } else if strings.Contains(info.UpstreamModelName, "-medium") {
        request.ReasoningEffort = "medium"
    } else if strings.Contains(info.UpstreamModelName, "-low") {
        request.ReasoningEffort = "low"
    }
}
```

##### B. Stream Options 配置

```go
// OpenAI/Azure: 添加 stream_options 以获取使用统计
if request.Stream && a.ChannelType == constant.ChannelTypeOpenAI {
    if request.StreamOptions == nil {
        request.StreamOptions = &dto.StreamOptions{}
    }
    request.StreamOptions.IncludeUsage = true
    info.SupportStreamOptions = true
}
```

##### C. Claude/Gemini 格式自动转换

```go
// 检测请求格式（通过结构或Header）
if isClaudeFormat(request) {
    claudeReq := parseClaudeRequest(request)
    request = service.ClaudeToOpenAIRequest(claudeReq, info)
}

if isGeminiFormat(request) {
    geminiReq := parseGeminiRequest(request)
    request = service.GeminiToOpenAIRequest(geminiReq, info)
}
```

##### D. OpenRouter 特殊处理

```go
// OpenRouter的-thinking后缀模型转换为推理API
if a.ChannelType == constant.ChannelTypeOpenRouter {
    if strings.HasSuffix(info.UpstreamModelName, "-thinking") {
        info.UpstreamModelName = strings.TrimSuffix(info.UpstreamModelName, "-thinking")
        // 添加推理相关参数
    }
}
```

---

#### 2.3 GetRequestURL() - URL构建

**文件**: `relay/channel/openai/adaptor.go`

```go
func (a *Adaptor) GetRequestURL(info *relaycommon.RelayInfo) (string, error)
```

**支持的URL格式**:

##### A. Azure OpenAI

```go
// 格式: https://{base-url}/openai/deployments/{model}/chat/completions?api-version={version}
fullRequestURL := fmt.Sprintf("%s/openai/deployments/%s/%s?api-version=%s",
    baseURL,
    info.UpstreamModelName,
    requestPath,
    info.ApiVersion, // 例如: 2024-02-15-preview
)
```

**注意事项**:
- 需要配置 `api-version`（通过渠道设置）
- 模型名作为 deployment name
- 支持自定义端点

##### B. 标准 OpenAI

```go
// 格式: https://api.openai.com/v1/{endpoint}
fullRequestURL := fmt.Sprintf("%s/%s", baseURL, requestPath)
// 例如: https://api.openai.com/v1/chat/completions
```

##### C. 自定义渠道（支持占位符）

```go
// 渠道配置支持 {model} 占位符替换
if strings.Contains(baseURL, "{model}") {
    fullRequestURL = strings.Replace(baseURL, "{model}", info.UpstreamModelName, -1)
}
```

##### D. WebSocket 实时API

```go
// wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview
fullRequestURL := fmt.Sprintf("wss://%s/v1/realtime?model=%s",
    strings.TrimPrefix(baseURL, "https://"),
    info.UpstreamModelName,
)
```

---

#### 2.4 SetupRequestHeader() - 请求头设置

**文件**: `relay/channel/openai/adaptor.go`

```go
func (a *Adaptor) SetupRequestHeader(c *gin.Context, header *http.Header, info *relaycommon.RelayInfo) error
```

**认证头配置**:

##### A. Azure OpenAI 认证

```go
if info.ChannelType == constant.ChannelTypeAzure {
    header.Set("api-key", info.ApiKey)  // Azure专用header
}
```

##### B. 标准 OpenAI 认证

```go
else {
    header.Set("Authorization", "Bearer " + info.ApiKey)
    
    // 可选: OpenAI Organization
    if info.Organization != "" {
        header.Set("OpenAI-Organization", info.Organization)
    }
}
```

##### C. 其他标准头

```go
header.Set("Content-Type", "application/json")
header.Set("Accept", "application/json")

// User-Agent
header.Set("User-Agent", "new-api/1.0")
```

---

#### 2.5 DoRequest() - 执行HTTP请求

**文件**: `relay/channel/openai/adaptor.go`

```go
func (a *Adaptor) DoRequest(c *gin.Context, info *relaycommon.RelayInfo, requestBody io.Reader) (*http.Response, error)
```

**请求执行流程**:

```go
// 1. 构建HTTP请求
req, err := http.NewRequest("POST", fullRequestURL, requestBody)

// 2. 设置请求头
a.SetupRequestHeader(c, &req.Header, info)

// 3. 超时控制
client := &http.Client{
    Timeout: time.Duration(info.Timeout) * time.Second,
}

// 4. 执行请求
resp, err := client.Do(req)

// 5. 错误检查
if err != nil {
    return nil, fmt.Errorf("request failed: %w", err)
}

return resp, nil
```

---

#### 2.6 DoResponse() - 响应处理分发

**文件**: `relay/channel/openai/relay-openai.go`

```go
func (a *Adaptor) DoResponse(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo) (usage *dto.Usage, err *types.NewAPIError)
```

**响应处理路由**:

```go
switch info.RelayMode {
case constant.RelayModeChatCompletions:
    if info.IsStream {
        // 流式响应
        usage, err = OaiStreamHandler(c, resp, info, a.ResponseFormat)
    } else {
        // 非流式响应
        usage, err = OpenaiHandler(c, resp, info, a.ResponseFormat)
    }

case constant.RelayModeAudioSpeech:
    // TTS: 返回音频流
    usage, err = OpenaiTTSHandler(c, resp, info)

case constant.RelayModeAudioTranscription:
    // STT: 返回文本
    usage, err = OpenaiSTTHandler(c, resp, info)

case constant.RelayModeImagesGenerations:
    // 图像生成
    usage, err = OpenaiHandlerWithUsage(c, resp, info)

case constant.RelayModeRealtime:
    // WebSocket实时通信（在其他地方处理）
    return nil, nil
}
```

---

## 📨 请求处理流程

### 1. 路由层（Router）

**文件**: `router/relay-router.go`

```go
func SetRelayRouter(router *gin.Engine) {
    relayV1Router := router.Group("/v1")
    
    // Chat Completions
    relayV1Router.POST("/chat/completions", func(c *gin.Context) {
        controller.Relay(c, types.RelayFormatOpenAI)
    })
    
    // Embeddings
    relayV1Router.POST("/embeddings", func(c *gin.Context) {
        controller.Relay(c, types.RelayFormatEmbedding)
    })
    
    // Audio APIs
    audioRouter := relayV1Router.Group("/audio")
    {
        audioRouter.POST("/speech", func(c *gin.Context) {
            controller.Relay(c, types.RelayFormatAudioSpeech)
        })
        
        audioRouter.POST("/transcriptions", func(c *gin.Context) {
            controller.Relay(c, types.RelayFormatAudioTranscription)
        })
        
        audioRouter.POST("/translations", func(c *gin.Context) {
            controller.Relay(c, types.RelayFormatAudioTranslation)
        })
    }
    
    // Realtime (WebSocket)
    relayV1Router.GET("/realtime", func(c *gin.Context) {
        controller.Relay(c, types.RelayFormatOpenAIRealtime)
    })
    
    // Images
    relayV1Router.POST("/images/generations", func(c *gin.Context) {
        controller.Relay(c, types.RelayFormatImagesGenerations)
    })
}
```

### 2. 中间件栈（Middleware）

#### 2.1 Token认证

**文件**: `middleware/auth.go`

```go
func TokenAuth() func(c *gin.Context) {
    return func(c *gin.Context) {
        // 1. 提取Token
        key := c.Request.Header.Get("Authorization")
        key = strings.TrimPrefix(key, "Bearer ")
        
        // 2. 数据库验证
        token := model.ValidateUserToken(key)
        if token == nil {
            c.JSON(401, gin.H{"error": "Invalid token"})
            c.Abort()
            return
        }
        
        // 3. 检查配额
        if token.RemainQuota <= 0 {
            c.JSON(429, gin.H{"error": "Quota exceeded"})
            c.Abort()
            return
        }
        
        // 4. 设置上下文
        SetupContextForToken(c, token)
        c.Next()
    }
}
```

#### 2.2 渠道分配（Channel Distribution）

**文件**: `middleware/distributor.go`

```go
func Distribute() func(c *gin.Context) {
    return func(c *gin.Context) {
        // 1. 获取可用渠道列表
        channels := model.GetAvailableChannels(modelName)
        
        // 2. 加权随机选择
        channel := selectChannelByWeight(channels)
        
        // 3. 健康检查
        if !channel.IsHealthy() {
            // 尝试下一个渠道
            channel = fallbackChannel(channels)
        }
        
        // 4. 设置渠道信息到上下文
        c.Set("channel", channel)
        c.Next()
    }
}
```

**选择策略**:
- **加权随机**: 根据渠道优先级（Priority字段）加权
- **健康检查**: 过滤掉失败次数过多的渠道
- **自动故障转移**: 请求失败时自动切换到备用渠道

### 3. 控制器层（Controller）

**文件**: `controller/relay.go`

```go
func Relay(c *gin.Context, relayFormat types.RelayFormat) {
    // 1. 解析请求体
    var request dto.GeneralOpenAIRequest
    if err := c.ShouldBindJSON(&request); err != nil {
        c.JSON(400, gin.H{"error": err.Error()})
        return
    }
    
    // 2. 获取Token和渠道信息
    token := c.GetInt("token_id")
    channel := c.MustGet("channel").(model.Channel)
    
    // 3. 构建RelayInfo上下文
    info := buildRelayInfo(c, token, channel, &request)
    
    // 4. 调用relay处理
    usage, err := relayHandler(c, info, &request)
    
    // 5. 扣除配额
    if err == nil {
        deductQuota(token, usage)
    }
    
    // 6. 记录日志
    logRequest(info, usage, err)
}
```

### 4. Relay Handler（核心处理）

**文件**: `controller/relay.go`

```go
func relayHandler(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest) (*dto.Usage, *types.NewAPIError) {
    // 1. 获取适配器
    adaptor := relay.GetAdaptor(info.ChannelType)
    
    // 2. 初始化适配器
    adaptor.Init(info)
    
    // 3. 转换请求
    if err := adaptor.ConvertOpenAIRequest(c, info, request); err != nil {
        return nil, types.NewAPIError(400, err.Error())
    }
    
    // 4. 序列化请求体
    requestBody, _ := json.Marshal(request)
    
    // 5. 执行请求
    resp, err := adaptor.DoRequest(c, info, bytes.NewReader(requestBody))
    if err != nil {
        // 重试逻辑
        if shouldRetry(err) {
            return retryRequest(c, info, request, adaptor)
        }
        return nil, types.NewAPIError(500, err.Error())
    }
    
    // 6. 处理响应
    usage, apiErr := adaptor.DoResponse(c, resp, info)
    
    return usage, apiErr
}
```

**重试策略**:

```go
func shouldRetry(err *types.NewAPIError) bool {
    // 429: Rate limit
    // 500-504: Server errors
    retryableStatusCodes := []int{429, 500, 502, 503, 504}
    return contains(retryableStatusCodes, err.StatusCode)
}

func retryRequest(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest, adaptor channel.Adaptor) (*dto.Usage, *types.NewAPIError) {
    maxRetries := 3
    for i := 0; i < maxRetries; i++ {
        time.Sleep(time.Duration(i+1) * time.Second) // 指数退避
        
        resp, err := adaptor.DoRequest(c, info, getRequestBody(request))
        if err == nil {
            return adaptor.DoResponse(c, resp, info)
        }
    }
    return nil, types.NewAPIError(500, "Max retries exceeded")
}
```

---

## 📤 响应处理流程

### 1. 流式响应处理（Streaming）

**文件**: `relay/channel/openai/relay-openai.go`

```go
func OaiStreamHandler(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo, responseFormat string) (usage *dto.Usage, err *types.NewAPIError) {
    // 1. 设置SSE响应头
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache")
    c.Writer.Header().Set("Connection", "keep-alive")
    
    // 2. 初始化Token计数器
    usage = &dto.Usage{}
    
    // 3. 逐行读取SSE流
    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() {
        line := scanner.Text()
        
        // 跳过空行和注释
        if line == "" || strings.HasPrefix(line, ":") {
            continue
        }
        
        // 提取data内容
        if strings.HasPrefix(line, "data: ") {
            data := strings.TrimPrefix(line, "data: ")
            
            // 结束标记
            if data == "[DONE]" {
                c.Writer.Write([]byte("data: [DONE]\n\n"))
                c.Writer.Flush()
                break
            }
            
            // 4. 解析JSON chunk
            var chunk dto.ChatCompletionsStreamResponse
            if err := json.Unmarshal([]byte(data), &chunk); err != nil {
                continue
            }
            
            // 5. 处理chunk内容
            if len(chunk.Choices) > 0 {
                delta := chunk.Choices[0].Delta
                
                // 累加Token计数
                if delta.Content != "" {
                    usage.CompletionTokens += countTokens(delta.Content)
                }
                
                // 处理推理内容（reasoning_content）
                if delta.ReasoningContent != "" && info.ThinkingToContent {
                    // 转换为<think>标签格式
                    delta.Content = fmt.Sprintf("<think>\n%s\n</think>\n%s", 
                        delta.ReasoningContent, delta.Content)
                    delta.ReasoningContent = ""
                }
                
                // 处理工具调用
                if len(delta.ToolCalls) > 0 {
                    // 累加工具调用Token
                    usage.CompletionTokens += countToolCallTokens(delta.ToolCalls)
                }
            }
            
            // 6. 提取使用统计（最后一个chunk）
            if chunk.Usage != nil {
                usage.PromptTokens = chunk.Usage.PromptTokens
                usage.CompletionTokens = chunk.Usage.CompletionTokens
                usage.TotalTokens = chunk.Usage.TotalTokens
            }
            
            // 7. 格式转换（如需要）
            if responseFormat == "claude" {
                data = convertToClaudeStream(chunk)
            } else if responseFormat == "gemini" {
                data = convertToGeminiStream(chunk)
            }
            
            // 8. 转发给客户端
            c.Writer.Write([]byte("data: " + data + "\n\n"))
            c.Writer.Flush()
        }
    }
    
    return usage, nil
}
```

**特殊处理**:

##### A. 音频模型（Audio Models）

```go
// 音频模型的usage在倒数第二个事件中
if info.IsAudioModel {
    // 缓存最后两个事件
    if secondToLastEvent.Usage != nil {
        usage = secondToLastEvent.Usage
    }
}
```

##### B. 推理内容转换（Thinking to Content）

```go
// O系列模型的推理过程可选转换为普通内容
if info.ThinkingToContent && delta.ReasoningContent != "" {
    delta.Content = fmt.Sprintf("<think>\n%s\n</think>\n%s",
        delta.ReasoningContent,
        delta.Content,
    )
    delta.ReasoningContent = ""
    chunk.Choices[0].Delta = delta
}
```

### 2. 非流式响应处理

**文件**: `relay/channel/openai/relay-openai.go`

```go
func OpenaiHandler(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo, responseFormat string) (usage *dto.Usage, err *types.NewAPIError) {
    // 1. 读取完整响应体
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, types.NewAPIError(500, "Failed to read response")
    }
    
    // 2. 解析JSON
    var openaiResp dto.OpenAITextResponse
    if err := json.Unmarshal(body, &openaiResp); err != nil {
        return nil, types.NewAPIError(500, "Invalid response format")
    }
    
    // 3. 提取使用统计
    usage = openaiResp.Usage
    if usage == nil {
        usage = &dto.Usage{}
    }
    
    // 4. 计算缺失的Token计数
    if usage.CompletionTokens == 0 {
        for _, choice := range openaiResp.Choices {
            usage.CompletionTokens += countTokens(choice.Message.Content)
            // 工具调用Token
            for _, toolCall := range choice.Message.ToolCalls {
                usage.CompletionTokens += countTokens(toolCall.Function.Arguments)
            }
        }
    }
    
    if usage.PromptTokens == 0 && info.PromptTokens > 0 {
        usage.PromptTokens = info.PromptTokens
    }
    
    usage.TotalTokens = usage.PromptTokens + usage.CompletionTokens
    
    // 5. 格式转换
    var responseData interface{} = openaiResp
    
    if responseFormat == "claude" {
        responseData = service.ResponseOpenAI2Claude(&openaiResp, info)
    } else if responseFormat == "gemini" {
        responseData = service.ResponseOpenAI2Gemini(&openaiResp, info)
    }
    
    // 6. 返回响应
    c.JSON(resp.StatusCode, responseData)
    
    return usage, nil
}
```

### 3. 音频响应处理

#### 3.1 TTS (Text-to-Speech)

**文件**: `relay/channel/openai/audio.go`

```go
func OpenaiTTSHandler(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo) (usage *dto.Usage, err *types.NewAPIError) {
    // 1. 设置音频响应头
    c.Writer.Header().Set("Content-Type", resp.Header.Get("Content-Type")) // audio/mpeg
    
    // 2. 流式转发音频数据
    written, err := io.Copy(c.Writer, resp.Body)
    if err != nil {
        return nil, types.NewAPIError(500, "Audio streaming failed")
    }
    
    // 3. 计算Token（基于文本长度估算）
    usage = &dto.Usage{
        PromptTokens:      info.PromptTokens, // 原始文本Token数
        CompletionTokens:  0,
        TotalTokens:       info.PromptTokens,
    }
    
    logger.Info(c, fmt.Sprintf("TTS generated %d bytes audio", written))
    
    return usage, nil
}
```

#### 3.2 STT (Speech-to-Text)

**文件**: `relay/channel/openai/audio.go`

```go
func OpenaiSTTHandler(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo) (usage *dto.Usage, err *types.NewAPIError) {
    // 1. 读取转写结果
    body, _ := io.ReadAll(resp.Body)
    
    var sttResp dto.AudioTranscriptionResponse
    json.Unmarshal(body, &sttResp)
    
    // 2. 计算Token（基于音频时长和输出文本）
    usage = &dto.Usage{
        PromptTokens:     info.AudioDurationTokens,  // 音频时长→Token（按秒计算）
        CompletionTokens: countTokens(sttResp.Text), // 转写文本Token
    }
    usage.TotalTokens = usage.PromptTokens + usage.CompletionTokens
    
    // 3. 返回转写结果
    c.JSON(200, sttResp)
    
    return usage, nil
}
```

### 4. WebSocket实时通信

**文件**: `relay/channel/openai/realtime.go`

```go
func OpenaiRealtimeHandler(c *gin.Context, info *relaycommon.RelayInfo) (usage *dto.Usage, err *types.NewAPIError) {
    // 1. 升级HTTP为WebSocket
    upgrader := websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool { return true },
    }
    
    clientConn, err := upgrader.Upgrade(c.Writer, c.Request, nil)
    if err != nil {
        return nil, types.NewAPIError(500, "WebSocket upgrade failed")
    }
    defer clientConn.Close()
    
    // 2. 连接到OpenAI WebSocket
    targetURL := fmt.Sprintf("wss://api.openai.com/v1/realtime?model=%s", info.UpstreamModelName)
    
    dialer := websocket.Dialer{}
    header := http.Header{}
    header.Set("Authorization", "Bearer "+info.ApiKey)
    header.Set("OpenAI-Beta", "realtime=v1")
    
    targetConn, _, err := dialer.Dial(targetURL, header)
    if err != nil {
        return nil, types.NewAPIError(500, "Failed to connect to OpenAI")
    }
    defer targetConn.Close()
    
    // 3. 双向消息转发
    usage = &dto.Usage{}
    
    // Client → OpenAI
    go func() {
        for {
            messageType, message, err := clientConn.ReadMessage()
            if err != nil {
                break
            }
            
            // 转发消息
            targetConn.WriteMessage(messageType, message)
            
            // 计算Token（音频/文本）
            usage.PromptTokens += estimateTokens(message)
        }
    }()
    
    // OpenAI → Client
    for {
        messageType, message, err := targetConn.ReadMessage()
        if err != nil {
            break
        }
        
        // 转发响应
        clientConn.WriteMessage(messageType, message)
        
        // 计算Token
        usage.CompletionTokens += estimateTokens(message)
    }
    
    usage.TotalTokens = usage.PromptTokens + usage.CompletionTokens
    
    return usage, nil
}
```

---

## 🚀 高级功能

### 1. 推理模型支持（Reasoning Models）

支持 OpenAI 的推理系列模型（O1/O3/GPT-5）的特殊功能。

#### 1.1 推理强度（Reasoning Effort）

**从模型名后缀解析**:

```go
// 支持格式: o1-mini-high, o1-preview-medium, o3-low
func parseReasoningEffort(modelName string) string {
    if strings.HasSuffix(modelName, "-high") {
        return "high"
    } else if strings.HasSuffix(modelName, "-medium") {
        return "medium"
    } else if strings.HasSuffix(modelName, "-low") {
        return "low"
    }
    return "" // 使用默认值
}

// 应用到请求
if effort := parseReasoningEffort(info.UpstreamModelName); effort != "" {
    request.ReasoningEffort = effort
    // 清理模型名后缀
    info.UpstreamModelName = strings.TrimSuffix(info.UpstreamModelName, "-"+effort)
}
```

#### 1.2 推理内容处理（Reasoning Content）

**Streaming模式**:

```go
// SSE流中包含 reasoning_content 字段
{
  "choices": [{
    "delta": {
      "reasoning_content": "Let me think about this step by step...",
      "content": "The answer is..."
    }
  }]
}

// 可选转换为<think>标签（兼容性）
if info.ThinkingToContent {
    convertedContent := fmt.Sprintf("<think>\n%s\n</think>\n%s",
        delta.ReasoningContent,
        delta.Content,
    )
}
```

**Non-Streaming模式**:

```go
// 完整响应包含 reasoning_content
{
  "choices": [{
    "message": {
      "reasoning_content": "My reasoning process...",
      "content": "Final answer..."
    }
  }]
}
```

### 2. 缓存计费（Cache Billing）

支持多家厂商的缓存机制，优化成本。

**支持的厂商**:
- Azure OpenAI
- DeepSeek
- Qwen (通义千问)

**使用统计结构**:

```go
type Usage struct {
    PromptTokens        int               `json:"prompt_tokens"`
    CompletionTokens    int               `json:"completion_tokens"`
    TotalTokens         int               `json:"total_tokens"`
    
    // 缓存相关
    PromptTokensDetails *PromptTokensDetails `json:"prompt_tokens_details,omitempty"`
}

type PromptTokensDetails struct {
    CachedTokens    int `json:"cached_tokens"`     // 缓存命中Token数
    AudioTokens     int `json:"audio_tokens"`      // 音频Token数
}
```

**处理逻辑**:

```go
// 兼容不同厂商的字段名
if usage.PromptTokensDetails != nil {
    // Azure/OpenAI: cached_tokens
    cachedTokens := usage.PromptTokensDetails.CachedTokens
    
    // DeepSeek/Qwen: prompt_cache_hit_tokens
    if cachedTokens == 0 && usage.PromptCacheHitTokens > 0 {
        cachedTokens = usage.PromptCacheHitTokens
    }
    
    // 计算实际计费Token
    billablePromptTokens := usage.PromptTokens - cachedTokens
    
    // 缓存命中计费（通常是正常价格的10%）
    cacheCost := cachedTokens * 0.1 * promptPrice
    normalCost := billablePromptTokens * promptPrice
    
    totalCost := cacheCost + normalCost
}
```

### 3. 多格式转换

支持客户端使用任意格式（OpenAI/Claude/Gemini），服务端自动转换。

#### 3.1 Claude → OpenAI 转换

**文件**: `service/convert.go`

```go
func ClaudeToOpenAIRequest(claudeReq dto.ClaudeRequest, info *RelayInfo) (*dto.GeneralOpenAIRequest, error) {
    openaiReq := &dto.GeneralOpenAIRequest{
        Model:       claudeReq.Model,
        MaxTokens:   claudeReq.MaxTokens,
        Temperature: claudeReq.Temperature,
        TopP:        claudeReq.TopP,
        Stream:      claudeReq.Stream,
    }
    
    // 1. System prompt 转换
    if claudeReq.System != "" {
        openaiReq.Messages = append(openaiReq.Messages, dto.Message{
            Role:    "system",
            Content: claudeReq.System,
        })
    }
    
    // 2. Messages 转换
    for _, msg := range claudeReq.Messages {
        openaiMsg := dto.Message{
            Role: msg.Role,
        }
        
        // 内容格式转换（支持多模态）
        if len(msg.Content) > 0 {
            // Claude 多模态格式 → OpenAI 格式
            for _, content := range msg.Content {
                switch content.Type {
                case "text":
                    openaiMsg.Content += content.Text
                    
                case "image":
                    // 图片转换
                    openaiMsg.MultiContent = append(openaiMsg.MultiContent, dto.Content{
                        Type: "image_url",
                        ImageURL: &dto.ImageURL{
                            URL: fmt.Sprintf("data:%s;base64,%s",
                                content.Source.MediaType,
                                content.Source.Data,
                            ),
                        },
                    })
                    
                case "thinking":
                    // Claude的thinking块转换
                    openaiMsg.Content += fmt.Sprintf("<think>%s</think>", content.Text)
                }
            }
        } else {
            openaiMsg.Content = msg.StringContent
        }
        
        openaiReq.Messages = append(openaiReq.Messages, openaiMsg)
    }
    
    // 3. Tool calls 转换
    if len(claudeReq.Tools) > 0 {
        for _, tool := range claudeReq.Tools {
            openaiReq.Tools = append(openaiReq.Tools, dto.Tool{
                Type: "function",
                Function: dto.Function{
                    Name:        tool.Name,
                    Description: tool.Description,
                    Parameters:  tool.InputSchema,
                },
            })
        }
    }
    
    return openaiReq, nil
}
```

#### 3.2 OpenAI → Claude 转换

```go
func ResponseOpenAI2Claude(openaiResp *dto.OpenAITextResponse, info *RelayInfo) *dto.ClaudeResponse {
    claudeResp := &dto.ClaudeResponse{
        ID:      openaiResp.ID,
        Type:    "message",
        Role:    "assistant",
        Model:   openaiResp.Model,
        Content: []dto.ClaudeContent{},
    }
    
    // 转换choices → content
    if len(openaiResp.Choices) > 0 {
        choice := openaiResp.Choices[0]
        
        // 1. 文本内容
        if choice.Message.Content != "" {
            claudeResp.Content = append(claudeResp.Content, dto.ClaudeContent{
                Type: "text",
                Text: choice.Message.Content,
            })
        }
        
        // 2. 推理内容（如果有）
        if choice.Message.ReasoningContent != "" {
            claudeResp.Content = append(claudeResp.Content, dto.ClaudeContent{
                Type: "thinking",
                Text: choice.Message.ReasoningContent,
            })
        }
        
        // 3. 工具调用
        for _, toolCall := range choice.Message.ToolCalls {
            claudeResp.Content = append(claudeResp.Content, dto.ClaudeContent{
                Type: "tool_use",
                ID:   toolCall.ID,
                Name: toolCall.Function.Name,
                Input: parseJSON(toolCall.Function.Arguments),
            })
        }
        
        // 4. Stop reason映射
        stopReasonMap := map[string]string{
            "stop":         "end_turn",
            "length":       "max_tokens",
            "tool_calls":   "tool_use",
            "content_filter": "stop_sequence",
        }
        claudeResp.StopReason = stopReasonMap[choice.FinishReason]
    }
    
    // 转换Usage
    if openaiResp.Usage != nil {
        claudeResp.Usage = dto.ClaudeUsage{
            InputTokens:  openaiResp.Usage.PromptTokens,
            OutputTokens: openaiResp.Usage.CompletionTokens,
        }
    }
    
    return claudeResp
}
```

### 4. Token 计数（精确计费）

**文件**: `service/token_counter.go`

使用 tiktoken 库进行精确Token计数。

```go
import "github.com/pkoukk/tiktoken-go"

// 模型对应的编码器
var encoderMap = map[string]string{
    "gpt-4":          "cl100k_base",
    "gpt-4-turbo":    "cl100k_base",
    "gpt-3.5-turbo":  "cl100k_base",
    "text-embedding": "cl100k_base",
    "davinci":        "p50k_base",
}

func CountTokens(model string, text string) int {
    encoding := encoderMap[model]
    if encoding == "" {
        encoding = "cl100k_base" // 默认
    }
    
    tke, err := tiktoken.GetEncoding(encoding)
    if err != nil {
        // 降级到估算（1 token ≈ 4 chars）
        return len(text) / 4
    }
    
    tokens := tke.Encode(text, nil, nil)
    return len(tokens)
}

// 消息Token计数（包含格式开销）
func CountMessageTokens(model string, messages []dto.Message) int {
    totalTokens := 0
    
    // 每条消息的固定开销（<|start|>role<|message|>content<|end|>）
    messageOverhead := 4
    
    for _, msg := range messages {
        totalTokens += messageOverhead
        totalTokens += CountTokens(model, msg.Role)
        totalTokens += CountTokens(model, msg.Content)
        
        // 多模态内容
        for _, content := range msg.MultiContent {
            if content.Type == "text" {
                totalTokens += CountTokens(model, content.Text)
            } else if content.Type == "image_url" {
                // 图片Token计算（基于图片尺寸）
                totalTokens += CalculateImageTokens(content.ImageURL.URL)
            }
        }
        
        // 工具调用
        for _, toolCall := range msg.ToolCalls {
            totalTokens += CountTokens(model, toolCall.Function.Name)
            totalTokens += CountTokens(model, toolCall.Function.Arguments)
        }
    }
    
    // 补全开始标记
    totalTokens += 3
    
    return totalTokens
}

// 图片Token计算
func CalculateImageTokens(imageURL string) int {
    // OpenAI 图片Token计算规则:
    // - 低分辨率: 85 tokens
    // - 高分辨率: 170 tokens + 额外tile tokens
    // 
    // Tile计算: ceil(width/512) * ceil(height/512) * 170
    
    // 简化版本（假设高分辨率）
    return 765 // (512x512 → 2x2 tiles = 4*170 + 85)
}
```

---

## 🔀 多渠道支持

### 1. 渠道管理架构

**数据模型**:

**文件**: `model/channel.go`

```go
type Channel struct {
    Id              int       `json:"id"`
    Type            int       `json:"type"`           // 渠道类型（OpenAI/Azure/Claude等）
    Key             string    `json:"key"`            // API密钥（加密存储）
    BaseURL         string    `json:"base_url"`       // 自定义端点
    Models          string    `json:"models"`         // 支持的模型列表（逗号分隔）
    Priority        int       `json:"priority"`       // 优先级（用于加权选择）
    Weight          int       `json:"weight"`         // 权重
    Status          int       `json:"status"`         // 状态（1=启用 2=禁用 3=自动禁用）
    
    // 统计信息
    SuccessCount    int       `json:"success_count"`  // 成功请求数
    FailureCount    int       `json:"failure_count"`  // 失败请求数
    LastUsedTime    int64     `json:"last_used_time"` // 最后使用时间
    
    // 配置
    Config          string    `json:"config"`         // JSON格式的渠道特定配置
    
    CreatedTime     int64     `json:"created_time"`
    TestTime        int64     `json:"test_time"`      // 最后测试时间
}
```

### 2. 渠道选择策略

**文件**: `middleware/distributor.go`

```go
func selectChannel(c *gin.Context, modelName string) (*model.Channel, error) {
    // 1. 获取可用渠道列表
    channels, err := model.GetEnabledChannelsByModel(modelName)
    if err != nil || len(channels) == 0 {
        return nil, errors.New("no available channels")
    }
    
    // 2. 过滤健康的渠道
    healthyChannels := []model.Channel{}
    for _, ch := range channels {
        if isChannelHealthy(&ch) {
            healthyChannels = append(healthyChannels, ch)
        }
    }
    
    if len(healthyChannels) == 0 {
        // 降级: 尝试使用状态不佳的渠道
        healthyChannels = channels
    }
    
    // 3. 加权随机选择
    channel := weightedRandomSelect(healthyChannels)
    
    return channel, nil
}

// 健康检查
func isChannelHealthy(channel *model.Channel) bool {
    // 失败率阈值检查
    totalRequests := channel.SuccessCount + channel.FailureCount
    if totalRequests > 10 {
        failureRate := float64(channel.FailureCount) / float64(totalRequests)
        if failureRate > 0.5 { // 失败率超过50%
            return false
        }
    }
    
    // 连续失败检查
    if channel.FailureCount > 5 {
        // 检查最近是否有成功
        timeSinceLastSuccess := time.Now().Unix() - channel.LastUsedTime
        if timeSinceLastSuccess > 300 { // 5分钟内没有成功
            return false
        }
    }
    
    return true
}

// 加权随机选择
func weightedRandomSelect(channels []model.Channel) *model.Channel {
    // 计算总权重
    totalWeight := 0
    for _, ch := range channels {
        weight := ch.Priority // 或使用 ch.Weight
        if weight <= 0 {
            weight = 1
        }
        totalWeight += weight
    }
    
    // 随机选择
    randValue := rand.Intn(totalWeight)
    currentWeight := 0
    
    for i, ch := range channels {
        weight := ch.Priority
        if weight <= 0 {
            weight = 1
        }
        currentWeight += weight
        
        if randValue < currentWeight {
            return &channels[i]
        }
    }
    
    return &channels[0] // 默认返回第一个
}
```

### 3. 自动故障转移

**文件**: `controller/relay.go`

```go
func relayWithFailover(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest) (*dto.Usage, *types.NewAPIError) {
    maxFailoverAttempts := 3
    originalChannelId := info.ChannelId
    
    for attempt := 0; attempt < maxFailoverAttempts; attempt++ {
        // 执行请求
        usage, err := relayHandler(c, info, request)
        
        if err == nil {
            // 成功: 更新渠道统计
            model.UpdateChannelSuccess(info.ChannelId)
            return usage, nil
        }
        
        // 失败: 记录并尝试切换渠道
        logger.Error(c, fmt.Sprintf("Channel %d failed: %s", info.ChannelId, err.Message))
        model.UpdateChannelFailure(info.ChannelId)
        
        // 判断是否应该切换渠道
        if !shouldFailover(err) {
            return nil, err
        }
        
        // 选择新渠道
        newChannel, failoverErr := selectChannel(c, info.OriginModelName)
        if failoverErr != nil {
            return nil, err // 返回原始错误
        }
        
        // 避免重复选择同一渠道
        if newChannel.Id == originalChannelId && attempt < maxFailoverAttempts-1 {
            continue
        }
        
        // 更新RelayInfo
        info.ChannelId = newChannel.Id
        info.ChannelType = newChannel.Type
        info.ApiKey = newChannel.Key
        info.ChannelBaseUrl = newChannel.BaseURL
        
        logger.Info(c, fmt.Sprintf("Failover to channel %d (attempt %d)", newChannel.Id, attempt+1))
    }
    
    return nil, types.NewAPIError(503, "All channels failed")
}

// 判断是否应该故障转移
func shouldFailover(err *types.NewAPIError) bool {
    // 可重试的错误码
    failoverStatusCodes := []int{
        429,  // Rate limit
        500,  // Internal server error
        502,  // Bad gateway
        503,  // Service unavailable
        504,  // Gateway timeout
    }
    
    for _, code := range failoverStatusCodes {
        if err.StatusCode == code {
            return true
        }
    }
    
    return false
}
```

### 4. 渠道特定配置

不同渠道可以有自定义配置。

**配置结构** (`dto/channel_settings.go`):

```go
type ChannelSettings struct {
    // 通用配置
    ForceFormat         bool   `json:"force_format"`          // 强制响应格式
    ThinkingToContent   bool   `json:"thinking_to_content"`   // 推理内容转换
    
    // Azure特定
    ApiVersion          string `json:"api_version"`           // Azure API版本
    
    // 参数覆盖
    MaxTokens           int    `json:"max_tokens"`            // 覆盖max_tokens
    Temperature         float64 `json:"temperature"`          // 覆盖temperature
    
    // 自定义头
    CustomHeaders       map[string]string `json:"custom_headers"` // 额外HTTP头
    
    // 超时配置
    Timeout             int    `json:"timeout"`               // 请求超时（秒）
    
    // 模型映射
    ModelMapping        map[string]string `json:"model_mapping"` // 模型名映射
}
```

**应用配置**:

```go
// 从渠道配置JSON解析
var settings dto.ChannelSettings
json.Unmarshal([]byte(channel.Config), &settings)

// 应用配置
if settings.ForceFormat {
    info.ResponseFormat = "openai"
}

if settings.ThinkingToContent {
    info.ThinkingToContent = true
}

// 模型名映射
if mappedModel, ok := settings.ModelMapping[info.OriginModelName]; ok {
    info.UpstreamModelName = mappedModel
}

// 自定义超时
if settings.Timeout > 0 {
    info.Timeout = settings.Timeout
}
```

---

## 📊 完整调用流程示例

### 场景：客户端调用 Chat Completions API

#### 1. 客户端请求

```bash
curl https://api.your-gateway.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-xxxxxxxxxxxxx" \
  -d '{
    "model": "gpt-4",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is 2+2?"}
    ],
    "stream": true
  }'
```

#### 2. 路由层处理

```
POST /v1/chat/completions
  ↓
router.POST("/chat/completions", func(c *gin.Context) {
    controller.Relay(c, types.RelayFormatOpenAI)
})
```

#### 3. 中间件处理

##### A. CORS中间件
```go
// 允许跨域请求
c.Header("Access-Control-Allow-Origin", "*")
```

##### B. TokenAuth中间件
```go
// 1. 提取Token: "Bearer sk-xxxxxxxxxxxxx"
token := extractToken(c)

// 2. 数据库验证
dbToken := model.ValidateUserToken(token)
if dbToken == nil {
    c.JSON(401, gin.H{"error": "Invalid token"})
    return
}

// 3. 检查配额
if dbToken.RemainQuota <= 0 {
    c.JSON(429, gin.H{"error": "Quota exceeded"})
    return
}

// 4. 设置上下文
c.Set("token_id", dbToken.Id)
c.Set("user_id", dbToken.UserId)
c.Set("remain_quota", dbToken.RemainQuota)
```

##### C. RateLimit中间件
```go
// 限流检查
if isRateLimited(dbToken.UserId, "gpt-4") {
    c.JSON(429, gin.H{"error": "Rate limit exceeded"})
    return
}
```

##### D. Distribute中间件
```go
// 选择渠道
channels := model.GetEnabledChannelsByModel("gpt-4")
// 假设有3个OpenAI渠道:
// - Channel 1: Priority 10, BaseURL: api.openai.com
// - Channel 2: Priority 5,  BaseURL: api.openai.com (备用)
// - Channel 3: Priority 2,  BaseURL: custom-proxy.com

// 加权随机选择 → 选中 Channel 1
selectedChannel := weightedRandomSelect(channels)

c.Set("channel", selectedChannel)
```

#### 4. 控制器处理

```go
func Relay(c *gin.Context, relayFormat types.RelayFormat) {
    // 1. 解析请求体
    var request dto.GeneralOpenAIRequest
    c.ShouldBindJSON(&request)
    // request.Model = "gpt-4"
    // request.Messages = [...]
    // request.Stream = true
    
    // 2. 获取上下文信息
    tokenId := c.GetInt("token_id")
    userId := c.GetInt("user_id")
    channel := c.MustGet("channel").(model.Channel)
    
    // 3. 构建RelayInfo
    info := &relaycommon.RelayInfo{
        TokenId:           tokenId,
        UserId:            userId,
        ChannelId:         channel.Id,
        ChannelType:       channel.Type,      // constant.ChannelTypeOpenAI
        ApiKey:            channel.Key,        // sk-xxxxxxxxxxxxxxx
        ChannelBaseUrl:    channel.BaseURL,   // https://api.openai.com
        OriginModelName:   "gpt-4",
        UpstreamModelName: "gpt-4",
        IsStream:          true,
        RelayMode:         constant.RelayModeChatCompletions,
    }
    
    // 4. 调用relay处理
    usage, err := relayHandler(c, info, &request)
    
    // 5. 扣除配额（请求结束后）
    if err == nil {
        cost := calculateCost(usage, "gpt-4")
        model.DeductQuota(tokenId, cost)
    }
    
    // 6. 记录日志
    logRequest(info, usage, err)
}
```

#### 5. Relay Handler处理

```go
func relayHandler(c *gin.Context, info *relaycommon.RelayInfo, request *dto.GeneralOpenAIRequest) (*dto.Usage, *types.NewAPIError) {
    // 1. 获取适配器
    adaptor := relay.GetAdaptor(info.ChannelType)
    // → 返回 &openai.Adaptor{}
    
    // 2. 初始化适配器
    adaptor.Init(info)
    // → adaptor.ChannelType = constant.ChannelTypeOpenAI
    
    // 3. 转换请求
    err := adaptor.ConvertOpenAIRequest(c, info, request)
    // → 添加 stream_options.include_usage = true
    // → 计算 prompt tokens
    
    // 4. 序列化请求体
    requestBody, _ := json.Marshal(request)
    
    // 5. 执行HTTP请求
    resp, err := adaptor.DoRequest(c, info, bytes.NewReader(requestBody))
    // → URL: https://api.openai.com/v1/chat/completions
    // → Header: Authorization: Bearer sk-xxxxxxx
    // → Body: {"model":"gpt-4","messages":[...],"stream":true,...}
    
    if err != nil {
        return nil, types.NewAPIError(500, err.Error())
    }
    
    // 6. 处理响应
    usage, apiErr := adaptor.DoResponse(c, resp, info)
    // → 调用 OaiStreamHandler (因为 IsStream=true)
    
    return usage, apiErr
}
```

#### 6. OpenAI Adaptor处理

##### A. DoRequest执行

```go
func (a *Adaptor) DoRequest(c *gin.Context, info *relaycommon.RelayInfo, requestBody io.Reader) (*http.Response, error) {
    // 1. 构建URL
    fullRequestURL, _ := a.GetRequestURL(info)
    // → "https://api.openai.com/v1/chat/completions"
    
    // 2. 创建HTTP请求
    req, _ := http.NewRequest("POST", fullRequestURL, requestBody)
    
    // 3. 设置请求头
    a.SetupRequestHeader(c, &req.Header, info)
    // → Authorization: Bearer sk-xxxxxxx
    // → Content-Type: application/json
    
    // 4. 执行请求
    client := &http.Client{Timeout: 180 * time.Second}
    resp, err := client.Do(req)
    
    return resp, err
}
```

##### B. DoResponse处理（流式）

```go
func (a *Adaptor) DoResponse(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo) (*dto.Usage, *types.NewAPIError) {
    // 因为 IsStream=true，路由到 OaiStreamHandler
    return OaiStreamHandler(c, resp, info, a.ResponseFormat)
}
```

#### 7. 流式响应处理

```go
func OaiStreamHandler(c *gin.Context, resp *http.Response, info *relaycommon.RelayInfo, responseFormat string) (*dto.Usage, *types.NewAPIError) {
    // 1. 设置SSE响应头
    c.Header("Content-Type", "text/event-stream")
    c.Header("Cache-Control", "no-cache")
    c.Header("Connection", "keep-alive")
    
    // 2. 初始化usage
    usage := &dto.Usage{}
    
    // 3. 逐行读取SSE流
    scanner := bufio.NewScanner(resp.Body)
    for scanner.Scan() {
        line := scanner.Text()
        
        if strings.HasPrefix(line, "data: ") {
            data := strings.TrimPrefix(line, "data: ")
            
            if data == "[DONE]" {
                c.Writer.Write([]byte("data: [DONE]\n\n"))
                c.Writer.Flush()
                break
            }
            
            // 4. 解析chunk
            var chunk dto.ChatCompletionsStreamResponse
            json.Unmarshal([]byte(data), &chunk)
            // chunk = {
            //   "id": "chatcmpl-xxxxx",
            //   "choices": [{
            //     "delta": {"content": "4"},
            //     "index": 0
            //   }]
            // }
            
            // 5. 累加token计数
            if len(chunk.Choices) > 0 {
                delta := chunk.Choices[0].Delta
                if delta.Content != "" {
                    usage.CompletionTokens += countTokens(delta.Content)
                }
            }
            
            // 6. 提取最终usage（最后一个chunk）
            if chunk.Usage != nil {
                usage.PromptTokens = chunk.Usage.PromptTokens
                usage.CompletionTokens = chunk.Usage.CompletionTokens
                usage.TotalTokens = chunk.Usage.TotalTokens
            }
            
            // 7. 转发给客户端
            c.Writer.Write([]byte("data: " + data + "\n\n"))
            c.Writer.Flush()
        }
    }
    
    // 8. 返回usage用于计费
    return usage, nil
}
```

#### 8. 客户端接收响应

```
data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{"role":"assistant"},"index":0}]}

data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{"content":"The"},"index":0}]}

data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{"content":" answer"},"index":0}]}

data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{"content":" is"},"index":0}]}

data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{"content":" 4"},"index":0}]}

data: {"id":"chatcmpl-xxxxx","choices":[{"delta":{},"index":0,"finish_reason":"stop"}],"usage":{"prompt_tokens":25,"completion_tokens":5,"total_tokens":30}}

data: [DONE]
```

#### 9. 计费与日志

```go
// 计算成本
// GPT-4 价格: $0.03 / 1K prompt tokens, $0.06 / 1K completion tokens
promptCost := (usage.PromptTokens / 1000.0) * 0.03
completionCost := (usage.CompletionTokens / 1000.0) * 0.06
totalCost := promptCost + completionCost

// 扣除配额
model.DeductQuota(tokenId, totalCost)

// 记录日志
model.CreateLog(&model.Log{
    UserId:           userId,
    ChannelId:        channelId,
    TokenId:          tokenId,
    Model:            "gpt-4",
    PromptTokens:     usage.PromptTokens,
    CompletionTokens: usage.CompletionTokens,
    Cost:             totalCost,
    CreatedAt:        time.Now().Unix(),
})
```

---

## ⚙️ 配置与部署

### 1. 环境变量配置

**文件**: `.env`

```bash
# 数据库配置
DB_TYPE=mysql
DB_HOST=localhost
DB_PORT=3306
DB_NAME=new_api
DB_USER=root
DB_PASSWORD=password

# Redis配置（可选，用于缓存和限流）
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# 服务配置
PORT=3000
GIN_MODE=release

# 日志配置
LOG_LEVEL=info
LOG_DIR=./logs

# 安全配置
SESSION_SECRET=your-secret-key-here
API_KEY_SALT=your-salt-here
```

### 2. 渠道配置示例

**通过管理界面或API配置渠道**:

#### OpenAI官方渠道

```json
{
  "type": 1,
  "name": "OpenAI Official",
  "base_url": "https://api.openai.com",
  "key": "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "models": "gpt-4,gpt-4-turbo,gpt-3.5-turbo,text-embedding-ada-002",
  "priority": 10,
  "status": 1,
  "config": {
    "force_format": false,
    "thinking_to_content": false,
    "timeout": 180
  }
}
```

#### Azure OpenAI渠道

```json
{
  "type": 8,
  "name": "Azure OpenAI",
  "base_url": "https://your-resource.openai.azure.com",
  "key": "your-azure-api-key",
  "models": "gpt-4,gpt-35-turbo",
  "priority": 8,
  "status": 1,
  "config": {
    "api_version": "2024-02-15-preview",
    "model_mapping": {
      "gpt-4": "gpt-4-deployment-name",
      "gpt-3.5-turbo": "gpt-35-turbo-deployment"
    }
  }
}
```

#### 自定义OpenAI兼容渠道

```json
{
  "type": 1,
  "name": "Custom Proxy",
  "base_url": "https://custom-proxy.com/v1",
  "key": "custom-api-key",
  "models": "gpt-4,gpt-3.5-turbo",
  "priority": 5,
  "status": 1,
  "config": {
    "custom_headers": {
      "X-Custom-Header": "value"
    }
  }
}
```

### 3. 数据库Schema

**渠道表** (`channels`):

```sql
CREATE TABLE `channels` (
  `id` int NOT NULL AUTO_INCREMENT,
  `type` int NOT NULL,
  `name` varchar(255) NOT NULL,
  `key` text NOT NULL,
  `base_url` varchar(255) DEFAULT NULL,
  `models` text,
  `priority` int DEFAULT '0',
  `weight` int DEFAULT '0',
  `status` int DEFAULT '1',
  `success_count` int DEFAULT '0',
  `failure_count` int DEFAULT '0',
  `last_used_time` bigint DEFAULT NULL,
  `config` text,
  `created_time` bigint NOT NULL,
  `test_time` bigint DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_type` (`type`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**Token表** (`tokens`):

```sql
CREATE TABLE `tokens` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `key` varchar(48) NOT NULL UNIQUE,
  `name` varchar(255) DEFAULT NULL,
  `remain_quota` bigint DEFAULT '0',
  `unlimited_quota` tinyint(1) DEFAULT '0',
  `status` int DEFAULT '1',
  `created_time` bigint NOT NULL,
  `accessed_time` bigint DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_key` (`key`),
  KEY `idx_user_id` (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 4. 部署方式

#### A. Docker部署

```bash
# 使用官方镜像
docker run -d \
  --name new-api \
  -p 3000:3000 \
  -e DB_TYPE=mysql \
  -e DB_HOST=your-db-host \
  -e DB_USER=root \
  -e DB_PASSWORD=password \
  -e DB_NAME=new_api \
  calciumion/new-api:latest

# 使用docker-compose
docker-compose up -d
```

#### B. 源码部署

```bash
# 安装依赖
go mod download

# 编译
go build -o new-api main.go

# 运行
./new-api
```

#### C. Systemd服务

```ini
[Unit]
Description=New API Service
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/new-api
ExecStart=/opt/new-api/new-api
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## 📚 参考资源

- [OpenAI API Official Documentation](https://platform.openai.com/docs/api-reference)
- [Azure OpenAI Documentation](https://learn.microsoft.com/azure/ai-services/openai/)
- [New-API GitHub Repository](https://github.com/Calcium-Ion/new-api)
- [Tiktoken - OpenAI Tokenizer](https://github.com/openai/tiktoken)

---

## 📝 总结

### OpenAI 集成核心要点

1. **适配器模式**: 统一接口，支持多种衍生服务（OpenAI/Azure/OpenRouter）
2. **多格式兼容**: 自动识别并转换 OpenAI/Claude/Gemini 格式
3. **推理模型支持**: 完整支持 O系列推理模型的特殊功能
4. **流式处理**: 高效的 SSE 流式响应和 WebSocket 实时通信
5. **多渠道架构**: 负载均衡、故障转移、健康检查
6. **精确计费**: 基于 tiktoken 的精确 Token 计数
7. **企业级功能**: 配额管理、多租户、审计日志

### 架构优势

✅ **高可用性**: 多渠道冗余 + 自动故障转移  
✅ **高性能**: 流式处理 + 连接池优化  
✅ **高扩展性**: 插件化适配器架构  
✅ **成本优化**: 缓存计费 + 精确Token计数  
✅ **开发友好**: 统一API接口 + 多格式兼容

### 后续扩展方向

- 支持更多 OpenAI 兼容服务（Anthropic、Google AI Studio等）
- 增强推理模型功能（思维链可视化、推理过程控制）
- 性能优化（请求批处理、响应缓存、连接复用）
- 高级监控（实时metrics、链路追踪、性能分析）

---

**文档版本**: v1.0  
**最后更新**: 2025-11-25  
**许可证**: MIT
