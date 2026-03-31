# Function Calling 메커니즘 상세 가이드

**작성일**: 2026년 3월 31일  
**프로젝트**: SafeTunes AI Agent  
**주제**: Ollama/Qwen LLM의 Function Calling 작동 원리

---

## 📋 개요

SafeTunes AI Agent는 **OpenAI Function Calling 표준**을 따르는 Ollama API를 사용합니다. 이 문서는 사용자의 자연어 명령이 실제 CLI 실행으로 변환되는 전체 과정을 단계별로 설명합니다.

---

## 🎯 핵심 개념

### Function Calling이란?

LLM이 **외부 도구(Tool/Function)를 호출해야 한다고 판단**하면, 직접 실행하지 않고 **"이 도구를 이렇게 호출하세요"라는 정보**만 반환합니다. 실제 실행은 우리 코드(SafeTunesAgent)가 담당합니다.

### 왜 이렇게 설계되었나?

| 이유 | 설명 |
|------|------|
| 🔒 **보안** | LLM이 시스템 명령을 직접 실행하면 위험 |
| ✅ **검증** | Tool 실행 전후에 우리가 검증 가능 |
| 🛡️ **에러 핸들링** | CLI 실패해도 LLM에게 피드백하여 재시도 가능 |
| 🔄 **확장성** | 새 Tool 추가해도 LLM 재학습 불필요 |

---

## 🔄 전체 Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│  Step 1: 사용자 입력                                              │
│  "C:/data/battery.csv를 MAVD 알고리즘으로 분석해줘"               │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 2: SafeTunesAgent - Tool Definition 준비                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ var tools = ToolRegistry.GetAllTools();                    │  │
│  │ // 5개의 ToolDefinition 객체 생성                          │  │
│  │ // - run_mavd_algorithm                                    │  │
│  │ // - run_rdv_algorithm                                     │  │
│  │ // - run_dsoh_algorithm                                    │  │
│  │ // - list_workspaces                                       │  │
│  │ // - get_workspace_info                                    │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 3: HTTP POST to Ollama API                                │
│  POST http://10.99.238.102:11434/v1/chat/completions            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ {                                                          │  │
│  │   "model": "qwen3-coder:30b",                              │  │
│  │   "messages": [                                            │  │
│  │     {"role": "system", "content": "You are SafeTunes..."}  │  │
│  │     {"role": "user", "content": "battery.csv를 MAVD..."}   │  │
│  │   ],                                                       │  │
│  │   "tools": [ /* 5개 Tool 정의 JSON */ ],                   │  │
│  │   "tool_choice": "auto",                                   │  │
│  │   "temperature": 0.7                                       │  │
│  │ }                                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 4: Qwen LLM 추론                                           │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ [LLM 사고 과정]                                            │  │
│  │ • 사용자가 "MAVD로 분석"이라고 했음                         │  │
│  │ • 사용 가능한 Tool 중 "run_mavd_algorithm"이 적합         │  │
│  │ • csv_path 파라미터는 "C:/data/battery.csv"              │  │
│  │ • 나머지 파라미터는 기본값 사용                            │  │
│  │ → ToolCall 반환 결정                                       │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 5: Ollama 응답 (ToolCall 포함)                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ {                                                          │  │
│  │   "choices": [{                                            │  │
│  │     "message": {                                           │  │
│  │       "role": "assistant",                                 │  │
│  │       "content": null,                                     │  │
│  │       "tool_calls": [{                                     │  │
│  │         "id": "call_abc123",                               │  │
│  │         "type": "function",                                │  │
│  │         "function": {                                      │  │
│  │           "name": "run_mavd_algorithm",                    │  │
│  │           "arguments": "{\"csv_path\":\"C:/data/...\"}"    │  │
│  │         }                                                  │  │
│  │       }]                                                   │  │
│  │     },                                                     │  │
│  │     "finish_reason": "tool_calls"                          │  │
│  │   }]                                                       │  │
│  │ }                                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 6: SafeTunesAgent - ToolCall 파싱 및 실행                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ if (assistantMessage.ToolCalls != null) {                 │  │
│  │   foreach (var toolCall in assistantMessage.ToolCalls) {  │  │
│  │     string toolName = toolCall.Function.Name;             │  │
│  │     string toolArgs = toolCall.Function.Arguments;        │  │
│  │                                                            │  │
│  │     // ⬇️ 실제 Tool 실행                                   │  │
│  │     string result = await ExecuteToolAsync(...);          │  │
│  │   }                                                        │  │
│  │ }                                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 7: CLI 실행                                                │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ CliExecutorService.ExecuteAsync(                           │  │
│  │   "C:/data/battery.csv mavd --cells 96"                   │  │
│  │ )                                                          │  │
│  │                                                            │  │
│  │ → SafeTunes.Cli.exe 프로세스 실행                          │  │
│  │ → stdout: "✓ Success\n96 cells analyzed\n3 faults..."     │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 8: Tool 결과를 대화에 추가 → 다시 LLM 호출                  │
│  POST http://10.99.238.102:11434/v1/chat/completions            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ {                                                          │  │
│  │   "messages": [                                            │  │
│  │     {"role": "system", ...},                              │  │
│  │     {"role": "user", ...},                                │  │
│  │     {"role": "assistant", "tool_calls": [...]},           │  │
│  │     {"role": "tool",                                      │  │
│  │      "tool_call_id": "call_abc123",                       │  │
│  │      "content": "✓ Success\n96 cells analyzed..."}        │  │
│  │   ],                                                       │  │
│  │   "tools": [...]                                           │  │
│  │ }                                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 9: LLM의 최종 응답 (자연어 정리)                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ {                                                          │  │
│  │   "choices": [{                                            │  │
│  │     "message": {                                           │  │
│  │       "role": "assistant",                                 │  │
│  │       "content": "분석이 완료되었습니다!\n\n              │  │
│  │                   📊 결과 요약:\n                          │  │
│  │                   - 총 96개 셀 분석\n                      │  │
│  │                   - 3개의 결함 감지",                      │  │
│  │       "tool_calls": null                                   │  │
│  │     },                                                     │  │
│  │     "finish_reason": "stop"                                │  │
│  │   }]                                                       │  │
│  │ }                                                          │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────┬─────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│  Step 10: 사용자에게 최종 응답 표시                               │
│  "분석이 완료되었습니다! 📊 결과 요약: ..."                       │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔧 코드 레벨 상세 설명

### 1. Tool Definition 생성

**파일**: `Services/AI/ToolRegistry.cs`

```csharp
public static List<ToolDefinition> GetAllTools()
{
    return new List<ToolDefinition>
    {
        new ToolDefinition(
            name: "run_mavd_algorithm",
            description: "Run MAVD (Maximum Allowable Voltage Difference) " +
                        "algorithm on CSV data file. Detects battery cell faults.",
            parameters: new Dictionary<string, ParameterDefinition>
            {
                ["csv_path"] = ParameterDefinition.String(
                    "Absolute path to the CSV data file", 
                    required: true),
                ["voltage_threshold"] = ParameterDefinition.Number(
                    "Voltage threshold for fault detection (default: 3.8V)", 
                    required: false),
                ["cells"] = ParameterDefinition.Integer(
                    "Total number of battery cells (default: auto-detect)", 
                    required: false)
            }
        ),
        // ... 4개 더
    };
}
```

**JSON 직렬화 결과**:
```json
{
  "type": "function",
  "function": {
    "name": "run_mavd_algorithm",
    "description": "Run MAVD (Maximum Allowable Voltage Difference) algorithm on CSV data file. Detects battery cell faults.",
    "parameters": {
      "type": "object",
      "properties": {
        "csv_path": {
          "type": "string",
          "description": "Absolute path to the CSV data file"
        },
        "voltage_threshold": {
          "type": "number",
          "description": "Voltage threshold for fault detection (default: 3.8V)"
        },
        "cells": {
          "type": "integer",
          "description": "Total number of battery cells (default: auto-detect)"
        }
      },
      "required": ["csv_path"]
    }
  }
}
```

---

### 2. LLM 호출 with Tools

**파일**: `Services/AI/SafeTunesAgent.cs`

```csharp
public async Task<AgentResponse> ProcessMessageAsync(
    string userMessage,
    List<ChatMessage>? conversationHistory = null,
    CancellationToken cancellationToken = default)
{
    var messages = new List<ChatMessage>
    {
        ChatMessage.System(SystemPrompt),
        ...conversationHistory,
        ChatMessage.User(userMessage)
    };

    while (iteration < MaxIterations)
    {
        // ⬇️ 여기서 Tools를 LLM에 전달
        var response = await _ollama.CreateChatCompletionAsync(
            messages,
            tools: _availableTools,  // ⬅️ 5개의 ToolDefinition
            temperature: 0.7,
            cancellationToken: cancellationToken
        );

        var assistantMessage = response.Choices[0].Message;
        messages.Add(assistantMessage);

        // ⬇️ ToolCall 체크
        if (assistantMessage.ToolCalls != null && assistantMessage.ToolCalls.Count > 0)
        {
            // Tool 실행 로직...
        }
        else
        {
            // 최종 응답
            return new AgentResponse { Response = assistantMessage.Content };
        }
    }
}
```

---

### 3. HTTP 요청 구성

**파일**: `Services/AI/OllamaService.cs`

```csharp
public async Task<ChatCompletionResponse> CreateChatCompletionAsync(
    List<ChatMessage> messages,
    List<ToolDefinition>? tools = null,
    double temperature = 0.7,
    int? maxTokens = null,
    CancellationToken cancellationToken = default)
{
    var request = new ChatCompletionRequest
    {
        Model = _modelId,              // "qwen3-coder:30b"
        Messages = messages,
        Tools = tools,                 // ⬅️ Tool 정의들
        Temperature = temperature,
        MaxTokens = maxTokens,
        Stream = false
    };

    // ⬇️ Tool이 있으면 "auto" 모드
    if (tools != null && tools.Count > 0)
    {
        request.ToolChoice = "auto";
    }

    string jsonRequest = JsonSerializer.Serialize(request, JsonOptions);
    _logger.Debug("Ollama Request: {Request}", jsonRequest);

    var content = new StringContent(jsonRequest, Encoding.UTF8, "application/json");
    string endpoint = $"{_baseUrl}/chat/completions";

    HttpResponseMessage response = await _httpClient.PostAsync(
        endpoint, content, cancellationToken);
    
    string jsonResponse = await response.Content.ReadAsStringAsync(cancellationToken);
    _logger.Debug("Ollama Response: {Response}", jsonResponse);

    return JsonSerializer.Deserialize<ChatCompletionResponse>(jsonResponse);
}
```

**실제 HTTP 요청**:
```http
POST http://10.99.238.102:11434/v1/chat/completions
Content-Type: application/json
Authorization: Bearer TEST_KEY

{
  "model": "qwen3-coder:30b",
  "messages": [
    {
      "role": "system",
      "content": "You are SafeTunes AI Assistant, an expert in battery management system (BMS) analysis..."
    },
    {
      "role": "user",
      "content": "C:/data/battery.csv를 MAVD 알고리즘으로 분석해줘"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "run_mavd_algorithm",
        "description": "Run MAVD algorithm...",
        "parameters": { ... }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "run_rdv_algorithm",
        "description": "Run RDV algorithm...",
        "parameters": { ... }
      }
    }
    // ... 총 5개
  ],
  "tool_choice": "auto",
  "temperature": 0.7,
  "stream": false
}
```

---

### 4. ToolCall 응답 파싱

**Ollama 응답 예시**:
```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "created": 1711886400,
  "model": "qwen3-coder:30b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "run_mavd_algorithm",
              "arguments": "{\"csv_path\":\"C:/data/battery.csv\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ],
  "usage": {
    "prompt_tokens": 350,
    "completion_tokens": 25,
    "total_tokens": 375
  }
}
```

**C# 파싱 코드**:
```csharp
// SafeTunesAgent.cs
var assistantMessage = response.Choices.FirstOrDefault()?.Message;

if (assistantMessage.ToolCalls != null && assistantMessage.ToolCalls.Count > 0)
{
    foreach (var toolCall in assistantMessage.ToolCalls)
    {
        string toolName = toolCall.Function.Name;           
        // "run_mavd_algorithm"
        
        string toolArgs = toolCall.Function.Arguments;      
        // "{\"csv_path\":\"C:/data/battery.csv\"}"
        
        _logger.Information("Executing tool: {Tool} with args: {Args}", toolName, toolArgs);
        executionSteps.Add($"🔧 Executing: {toolName}");

        // ⬇️ 실제 Tool 실행
        string toolResult = await ExecuteToolAsync(toolName, toolArgs, cancellationToken);
        
        executionSteps.Add($"✓ Result: {toolResult.Substring(0, Math.Min(100, toolResult.Length))}...");

        // ⬇️ Tool 결과를 대화에 추가
        messages.Add(ChatMessage.Tool(toolCall.Id, toolResult));
    }
    
    continue;  // ⬅️ 다시 LLM 호출하기 위해 루프 계속
}
```

---

### 5. Tool 실행

**파일**: `Services/AI/SafeTunesAgent.cs`

```csharp
private async Task<string> ExecuteToolAsync(
    string toolName,
    string argumentsJson,
    CancellationToken cancellationToken)
{
    try
    {
        // JSON 파싱
        var args = JsonSerializer.Deserialize<Dictionary<string, JsonElement>>(argumentsJson);
        
        // Tool 이름으로 분기
        return toolName switch
        {
            "run_mavd_algorithm" => await ExecuteMavdAsync(args, cancellationToken),
            "run_rdv_algorithm" => await ExecuteRdvAsync(args, cancellationToken),
            "run_dsoh_algorithm" => await ExecuteDsohAsync(args, cancellationToken),
            "list_workspaces" => await ExecuteListWorkspacesAsync(cancellationToken),
            "get_workspace_info" => await ExecuteGetWorkspaceInfoAsync(args, cancellationToken),
            _ => $"Error: Unknown tool '{toolName}'"
        };
    }
    catch (Exception ex)
    {
        _logger.Error(ex, "Tool execution failed: {Tool}", toolName);
        return $"Error executing tool: {ex.Message}";
    }
}

private async Task<string> ExecuteMavdAsync(
    Dictionary<string, JsonElement> args,
    CancellationToken cancellationToken)
{
    // 파라미터 추출
    string csvPath = GetStringArg(args, "csv_path") 
        ?? throw new ArgumentException("csv_path is required");

    var options = new Dictionary<string, string>();
    
    if (args.TryGetValue("voltage_threshold", out var vt))
        options["params"] = $"VoltageThreshold={vt.GetDouble()}";
    
    if (args.TryGetValue("workspace_name", out var wn))
        options["workspace"] = wn.GetString() ?? string.Empty;
    
    if (args.TryGetValue("cells", out var cells))
        options["cells"] = cells.GetInt32().ToString();

    // CLI 명령 빌드
    string arguments = CliExecutorService.BuildArguments(csvPath, "mavd", options);
    // 예: "C:/data/battery.csv mavd --cells 96"

    // ⬇️ 실제 CLI 실행
    var result = await _cliExecutor.ExecuteAsync(arguments, cancellationToken);

    return result.ToString();
    // "✓ Success\n[SafeTunes CLI]\nCSV File: ...\n96 cells analyzed\n3 faults detected"
}
```

---

### 6. CLI 실행

**파일**: `Services/AI/CliExecutorService.cs`

```csharp
public async Task<CliExecutionResult> ExecuteAsync(
    string arguments,
    CancellationToken cancellationToken = default)
{
    _logger.Information("Executing CLI: {Executable} {Arguments}", 
        _cliExecutablePath, arguments);

    var startInfo = new ProcessStartInfo
    {
        FileName = _cliExecutablePath,           // "SafeTunes.Cli.exe"
        Arguments = arguments,                   // "C:/data/battery.csv mavd"
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        UseShellExecute = false,
        CreateNoWindow = true
    };

    using var process = new Process { StartInfo = startInfo };
    
    var stdOutput = new StringBuilder();
    var stdError = new StringBuilder();
    
    process.OutputDataReceived += (s, e) => {
        if (e.Data != null) stdOutput.AppendLine(e.Data);
    };
    
    process.ErrorDataReceived += (s, e) => {
        if (e.Data != null) stdError.AppendLine(e.Data);
    };
    
    process.Start();
    process.BeginOutputReadLine();
    process.BeginErrorReadLine();
    
    await process.WaitForExitAsync(cancellationToken);
    
    return new CliExecutionResult
    {
        ExitCode = process.ExitCode,
        StandardOutput = stdOutput.ToString(),
        StandardError = stdError.ToString(),
        Success = process.ExitCode == 0
    };
}
```

---

### 7. Tool 결과를 LLM에 피드백

**다시 LLM 호출**:
```csharp
// messages 배열에 Tool 결과 추가됨
messages.Add(ChatMessage.Tool(toolCall.Id, toolResult));

// ⬇️ 다시 LLM 호출 (while 루프 계속)
var response = await _ollama.CreateChatCompletionAsync(
    messages,
    tools: _availableTools,
    temperature: 0.7,
    cancellationToken: cancellationToken
);
```

**HTTP 요청 (2번째)**:
```json
{
  "model": "qwen3-coder:30b",
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "C:/data/battery.csv를 MAVD로 분석해줘"},
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "run_mavd_algorithm",
          "arguments": "{\"csv_path\":\"C:/data/battery.csv\"}"
        }
      }]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "✓ Success\n[SafeTunes CLI]\nCSV File: C:/data/battery.csv\nAlgorithm: MAVD\n96 cells analyzed\n3 faults detected"
    }
  ],
  "tools": [...],
  "tool_choice": "auto"
}
```

---

### 8. LLM의 최종 응답

**Ollama 응답 (2번째)**:
```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "분석이 완료되었습니다!\n\n📊 결과 요약:\n- 파일: battery.csv\n- 알고리즘: MAVD (Maximum Allowable Voltage Difference)\n- 셀 개수: 96개\n- 감지된 결함: 3개\n\n결함이 발견된 셀을 자세히 확인하시겠습니까?",
        "tool_calls": null
      },
      "finish_reason": "stop"
    }
  ]
}
```

**C# 코드**:
```csharp
// tool_calls가 null이므로 while 루프 탈출
if (assistantMessage.ToolCalls == null)
{
    return new AgentResponse
    {
        Response = assistantMessage.Content,
        ExecutionSteps = executionSteps,
        ConversationHistory = messages,
        ToolCallsExecuted = executionSteps.Count
    };
}
```

---

## 📊 메시지 흐름 상세

### Round 1: Tool 호출 판단

```json
// Request
{
  "messages": [
    {"role": "system", "content": "You are SafeTunes AI Assistant..."},
    {"role": "user", "content": "battery.csv를 MAVD로 분석해줘"}
  ],
  "tools": [5개 Tool 정의]
}

// Response
{
  "message": {
    "role": "assistant",
    "tool_calls": [{"function": {"name": "run_mavd_algorithm", ...}}]
  },
  "finish_reason": "tool_calls"
}
```

### Round 2: Tool 결과 기반 최종 응답

```json
// Request
{
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "tool_calls": [...]},
    {"role": "tool", "content": "✓ Success\n96 cells analyzed..."}
  ],
  "tools": [5개 Tool 정의]
}

// Response
{
  "message": {
    "role": "assistant",
    "content": "분석이 완료되었습니다! 📊 결과 요약: ...",
    "tool_calls": null
  },
  "finish_reason": "stop"
}
```

---

## 🔍 중요한 설계 결정

### 1. Tool Choice "auto"

```csharp
if (tools != null && tools.Count > 0)
{
    request.ToolChoice = "auto";
}
```

**의미**: LLM이 자율적으로 판단
- Tool이 필요하면 → `tool_calls` 반환
- Tool 없이 답변 가능하면 → `content` 반환

**다른 옵션**:
- `"none"`: Tool 절대 사용 안 함
- `{"type": "function", "function": {"name": "run_mavd_algorithm"}}`: 특정 Tool 강제

### 2. MaxIterations 제한

```csharp
private const int MaxIterations = 10;

while (iteration < MaxIterations)
{
    iteration++;
    // ...
}

if (iteration >= MaxIterations)
{
    throw new InvalidOperationException("Agent exceeded maximum iterations");
}
```

**이유**: 무한 루프 방지
- LLM이 계속 Tool을 호출할 수 있음
- 통계적으로 대부분 2-3회 iteration으로 완료
- 복잡한 배치 작업도 5-6회면 충분

### 3. 대화 히스토리 관리

```csharp
var messages = new List<ChatMessage>();
messages.Add(ChatMessage.System(SystemPrompt));

if (conversationHistory != null)
    messages.AddRange(conversationHistory);

messages.Add(ChatMessage.User(userMessage));

// ... Tool 실행 후
messages.Add(assistantMessage);                    // LLM 응답
messages.Add(ChatMessage.Tool(toolCallId, result)); // Tool 결과
```

**효과**:
- LLM이 이전 대화 기억
- "그걸 다시 해줘", "같은 파일로 RDV도 돌려줘" 등 가능

---

## 🎯 실제 사용 예시

### Example 1: 단순 분석

```
👤 User: "C:/data/car_001.csv를 MAVD로 분석해줘"

🔄 Iteration 1:
  → LLM: tool_calls = [run_mavd_algorithm(csv_path="C:/data/car_001.csv")]
  → Agent: CLI 실행 → "✓ Success, 96 cells, 3 faults"
  
🔄 Iteration 2:
  → LLM: content = "분석 완료! 96개 셀, 3개 결함 발견"
  → Agent: 최종 응답 반환
  
✅ 총 2회 iteration으로 완료
```

### Example 2: 파라미터 조정

```
👤 User: "같은 파일인데 voltage threshold 3.5V로 낮춰서 다시 분석해줘"

🔄 Iteration 1:
  → LLM: tool_calls = [run_mavd_algorithm(
           csv_path="C:/data/car_001.csv", 
           voltage_threshold=3.5)]
  → Agent: CLI 실행 → "✓ Success, 96 cells, 8 faults"
  
🔄 Iteration 2:
  → LLM: content = "더 엄격한 기준으로 재분석. 8개 결함 발견 (이전 3개 → 8개)"
  → Agent: 최종 응답 반환
  
✅ 대화 히스토리 덕분에 "같은 파일" 이해
```

### Example 3: 복합 쿼리

```
👤 User: "워크스페이스 목록 보여주고, 가장 최근 것의 정보 알려줘"

🔄 Iteration 1:
  → LLM: tool_calls = [list_workspaces()]
  → Agent: "cli_mavd_car_001_20260331_143500\ncli_rdv_car_002_..."
  
🔄 Iteration 2:
  → LLM: tool_calls = [get_workspace_info("cli_mavd_car_001_20260331_143500")]
  → Agent: "Workspace: ..., 96 cells, 1 modules"
  
🔄 Iteration 3:
  → LLM: content = "워크스페이스 5개 있습니다. 가장 최근 것은: ..."
  → Agent: 최종 응답 반환
  
✅ 2개 Tool을 순차적으로 호출
```

---

## 🛡️ 에러 핸들링

### 1. LLM 연결 실패

```csharp
try
{
    var response = await _ollama.CreateChatCompletionAsync(...);
}
catch (HttpRequestException ex)
{
    _logger.Error(ex, "Ollama API connection failed");
    return "❌ AI 서버 연결 실패. 네트워크를 확인해주세요.";
}
catch (TimeoutException ex)
{
    _logger.Error(ex, "Ollama request timed out");
    return "⏱️ 요청 시간 초과. 다시 시도해주세요.";
}
```

### 2. Tool 실행 실패

```csharp
private async Task<string> ExecuteToolAsync(...)
{
    try
    {
        // Tool 실행
    }
    catch (Exception ex)
    {
        _logger.Error(ex, "Tool execution failed: {Tool}", toolName);
        return $"Error executing tool: {ex.Message}";
    }
}
```

**중요**: 에러도 LLM에게 전달
- LLM이 에러 메시지 보고 재시도 또는 사용자에게 설명

### 3. CLI 실행 실패

```csharp
var result = await _cliExecutor.ExecuteAsync(arguments, cancellationToken);

if (result.ExitCode != 0)
{
    return $"✗ Algorithm failed (Exit Code: {result.ExitCode})\n{result.StandardError}";
}
```

---

## 📈 성능 최적화

### 1. Tool Definition 캐싱

```csharp
public SafeTunesAgent(...)
{
    // ⬇️ 생성자에서 한 번만 생성
    _availableTools = ToolRegistry.GetAllTools();
}
```

**이유**: Tool 정의는 변하지 않으므로 매번 생성할 필요 없음

### 2. HTTP Client 재사용

```csharp
private readonly HttpClient _httpClient;

public OllamaService(...)
{
    _httpClient = new HttpClient { Timeout = TimeSpan.FromMinutes(5) };
    // ⬆️ 인스턴스 전체에서 재사용
}
```

**이유**: Connection pooling, DNS 캐싱

### 3. JSON Serialization 옵션 재사용

```csharp
private static readonly JsonSerializerOptions JsonOptions = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull
};
```

---

## 🔮 향후 확장 가능성

### 1. Streaming 응답

```csharp
public async IAsyncEnumerable<string> StreamChatCompletionAsync(...)
{
    request.Stream = true;
    
    using var response = await _httpClient.PostAsync(...);
    using var stream = await response.Content.ReadAsStreamAsync();
    
    await foreach (var line in ReadLinesAsync(stream))
    {
        if (line.StartsWith("data: "))
        {
            var chunk = JsonSerializer.Deserialize<ChatCompletionChunk>(line[6..]);
            yield return chunk.Choices[0].Delta.Content;
        }
    }
}
```

### 2. Parallel Tool Calls

```csharp
if (assistantMessage.ToolCalls != null && assistantMessage.ToolCalls.Count > 0)
{
    // ⬇️ 병렬 실행
    var tasks = assistantMessage.ToolCalls.Select(tc => 
        ExecuteToolAsync(tc.Function.Name, tc.Function.Arguments, ct));
    
    var results = await Task.WhenAll(tasks);
    
    for (int i = 0; i < assistantMessage.ToolCalls.Count; i++)
    {
        messages.Add(ChatMessage.Tool(
            assistantMessage.ToolCalls[i].Id, 
            results[i]));
    }
}
```

### 3. Function Calling 검증

```csharp
private bool ValidateToolCall(ToolCall toolCall)
{
    var tool = _availableTools.FirstOrDefault(t => t.Function.Name == toolCall.Function.Name);
    if (tool == null) return false;
    
    var args = JsonSerializer.Deserialize<Dictionary<string, JsonElement>>(
        toolCall.Function.Arguments);
    
    // Required 파라미터 체크
    foreach (var required in tool.Function.Parameters.Required)
    {
        if (!args.ContainsKey(required))
        {
            _logger.Warning("Missing required parameter: {Param}", required);
            return false;
        }
    }
    
    return true;
}
```

---

## 📚 참고 자료

### OpenAI Function Calling Specification
- https://platform.openai.com/docs/guides/function-calling
- JSON Schema: https://json-schema.org/

### Ollama API Documentation
- https://github.com/ollama/ollama/blob/main/docs/api.md
- OpenAI Compatibility: https://ollama.com/blog/openai-compatibility

### Related Papers
1. "ReACT: Synergizing Reasoning and Acting in Language Models" (2023)
   - https://arxiv.org/abs/2210.03629

2. "Toolformer: Language Models Can Teach Themselves to Use Tools" (2023)
   - https://arxiv.org/abs/2302.04761

---

## 💡 핵심 요약

### Function Calling의 본질

```
❌ LLM이 Tool을 직접 실행하는 게 아님
✅ LLM은 "이 Tool을 호출하세요"라는 정보만 반환
✅ 실제 실행은 우리 코드(SafeTunesAgent)가 담당
✅ 실행 결과를 다시 LLM에게 전달
✅ LLM이 결과를 해석해서 사용자에게 친절한 응답
```

### 장점

1. 🔒 **보안**: LLM이 시스템 명령 직접 실행 불가
2. ✅ **검증**: Tool 실행 전후 우리가 제어
3. 🛡️ **에러 핸들링**: CLI 실패해도 복구 가능
4. 🔄 **확장성**: 새 Tool 추가 시 LLM 재학습 불필요
5. 📊 **추적**: 모든 Tool 실행 로깅 가능

### 실제 Flow

```
User → Agent → LLM (with Tools)
              ↓
LLM → ToolCall info
              ↓
Agent → Execute Tool (CLI)
              ↓
Agent → LLM (with result)
              ↓
LLM → Natural language response
              ↓
Agent → User
```

---

**작성자**: SafeTunes Development Team  
**최종 수정**: 2026년 3월 31일  
**버전**: 1.0

이 문서는 SafeTunes AI Agent의 Function Calling 메커니즘을 설명합니다.  
질문이나 피드백은 프로젝트 이슈 트래커에 등록해주세요.
