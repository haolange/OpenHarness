# MCP 客户端实现

<cite>
**本文档引用的文件**
- [client.py](file://src/openharness/mcp/client.py)
- [types.py](file://src/openharness/mcp/types.py)
- [config.py](file://src/openharness/mcp/config.py)
- [mcp_tool.py](file://src/openharness/tools/mcp_tool.py)
- [read_mcp_resource_tool.py](file://src/openharness/tools/read_mcp_resource_tool.py)
- [list_mcp_resources_tool.py](file://src/openharness/tools/list_mcp_resources_tool.py)
- [base.py](file://src/openharness/tools/base.py)
- [test_integration.py](file://tests/test_mcp/test_integration.py)
- [test_stdio_flow.py](file://tests/test_mcp/test_stdio_flow.py)
- [fake_mcp_server.py](file://tests/fixtures/fake_mcp_server.py)
</cite>

## 目录
1. [简介](#简介)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构概览](#架构概览)
5. [详细组件分析](#详细组件分析)
6. [依赖分析](#依赖分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)
10. [附录](#附录)

## 简介

本文档详细介绍了 OpenHarness 项目中的 MCP（Model Context Protocol）客户端实现。MCP 是一个用于在 AI 应用程序和外部工具/资源之间建立标准化通信协议的框架。本实现提供了完整的 MCP 客户端管理器，支持多种传输方式（当前主要支持 STDIO），实现了连接管理、会话生命周期控制、工具调用和资源访问等功能。

该实现基于 Python 异步编程模型，使用 `mcp` 库作为底层通信层，通过 `AsyncExitStack` 确保资源的正确清理。系统设计遵循松耦合原则，通过适配器模式将 MCP 工具和资源暴露为标准的 OpenHarness 工具接口。

## 项目结构

OpenHarness 的 MCP 实现位于 `src/openharness/mcp/` 目录下，包含以下关键文件：

```mermaid
graph TB
subgraph "MCP 模块结构"
A[client.py<br/>MCP 客户端管理器]
B[types.py<br/>数据模型定义]
C[config.py<br/>配置加载器]
D[__init__.py<br/>模块导出]
end
subgraph "工具适配器"
E[mcp_tool.py<br/>MCP 工具适配器]
F[read_mcp_resource_tool.py<br/>资源读取工具]
G[list_mcp_resources_tool.py<br/>资源列表工具]
end
subgraph "基础工具框架"
H[base.py<br/>工具基类定义]
end
subgraph "测试文件"
I[test_integration.py<br/>集成测试]
J[test_stdio_flow.py<br/>STDIO 流测试]
K[fake_mcp_server.py<br/>测试服务器]
end
A --> B
A --> C
E --> A
F --> A
G --> A
E --> H
F --> H
G --> H
I --> A
J --> A
J --> K
```

**图表来源**
- [client.py:1-175](file://src/openharness/mcp/client.py#L1-L175)
- [types.py:1-77](file://src/openharness/mcp/types.py#L1-L77)
- [config.py:1-17](file://src/openharness/mcp/config.py#L1-L17)

**章节来源**
- [client.py:1-175](file://src/openharness/mcp/client.py#L1-L175)
- [types.py:1-77](file://src/openharness/mcp/types.py#L1-L77)
- [config.py:1-17](file://src/openharness/mcp/config.py#L1-L17)

## 核心组件

### McpClientManager 类

`McpClientManager` 是 MCP 客户端实现的核心类，负责管理所有 MCP 服务器连接和操作。该类提供了完整的连接生命周期管理功能。

#### 主要职责
- 连接管理：建立和维护与 MCP 服务器的连接
- 会话生命周期：管理 ClientSession 的创建、初始化和清理
- 工具调用：执行 MCP 工具并处理返回结果
- 资源访问：读取 MCP 资源并格式化输出
- 状态跟踪：监控所有连接的状态变化

#### 关键属性
- `_server_configs`: 存储服务器配置字典
- `_statuses`: 追踪每个服务器的连接状态
- `_sessions`: 维护活跃的 MCP 会话
- `_stacks`: 管理资源清理的 AsyncExitStack

**章节来源**
- [client.py:20-175](file://src/openharness/mcp/client.py#L20-L175)

### 数据模型

系统使用 Pydantic 模型来定义 MCP 配置和状态信息：

#### 服务器配置模型
- `McpStdioServerConfig`: STDIO 传输配置
- `McpHttpServerConfig`: HTTP 传输配置  
- `McpWebSocketServerConfig`: WebSocket 传输配置
- `McpServerConfig`: 服务器配置联合类型

#### 元数据模型
- `McpToolInfo`: 工具元数据信息
- `McpResourceInfo`: 资源元数据信息
- `McpConnectionStatus`: 连接状态信息

**章节来源**
- [types.py:11-77](file://src/openharness/mcp/types.py#L11-L77)

## 架构概览

MCP 客户端实现采用分层架构设计，确保了良好的可扩展性和可维护性：

```mermaid
graph TB
subgraph "应用层"
A[OpenHarness 应用]
B[工具注册表]
end
subgraph "适配器层"
C[McpToolAdapter]
D[ReadMcpResourceTool]
E[ListMcpResourcesTool]
end
subgraph "MCP 管理层"
F[McpClientManager]
G[连接状态管理]
H[会话生命周期管理]
end
subgraph "传输层"
I[STDIO 传输]
J[HTTP 传输]
K[WebSocket 传输]
end
subgraph "外部 MCP 服务器"
L[MCP 工具服务器]
M[MCP 资源服务器]
end
A --> B
B --> C
B --> D
B --> E
C --> F
D --> F
E --> F
F --> G
F --> H
H --> I
H --> J
H --> K
I --> L
I --> M
J --> L
J --> M
K --> L
K --> M
```

**图表来源**
- [client.py:20-175](file://src/openharness/mcp/client.py#L20-L175)
- [mcp_tool.py:14-56](file://src/openharness/tools/mcp_tool.py#L14-L56)
- [read_mcp_resource_tool.py:18-36](file://src/openharness/tools/read_mcp_resource_tool.py#L18-L36)
- [list_mcp_resources_tool.py:15-37](file://src/openharness/tools/list_mcp_resources_tool.py#L15-L37)

## 详细组件分析

### 连接管理实现

#### STDIO 传输连接流程

```mermaid
sequenceDiagram
participant App as 应用程序
participant Manager as McpClientManager
participant Stack as AsyncExitStack
participant Stdio as STDIO 客户端
participant Session as ClientSession
participant Server as MCP 服务器
App->>Manager : connect_all()
Manager->>Manager : 遍历服务器配置
Manager->>Manager : 检查配置类型
Manager->>Manager : _connect_stdio(name, config)
Manager->>Stack : 创建 AsyncExitStack
Stack->>Stdio : stdio_client(StdioServerParameters)
Stdio->>Server : 启动进程
Server-->>Stdio : 进程就绪
Stdio-->>Stack : 返回流对象
Stack->>Session : ClientSession(read_stream, write_stream)
Session->>Session : initialize()
Session->>Server : 发送初始化消息
Server-->>Session : 初始化确认
Session->>Session : list_tools()
Session->>Session : list_resources()
Session-->>Manager : 工具和资源列表
Manager->>Manager : 更新状态和会话
Manager-->>App : 连接完成
```

**图表来源**
- [client.py:121-175](file://src/openharness/mcp/client.py#L121-L175)

#### 连接状态管理

系统使用 `McpConnectionStatus` 模型来跟踪每个服务器的连接状态：

```mermaid
stateDiagram-v2
[*] --> pending
pending --> connected : 连接成功
pending --> failed : 连接失败
connected --> failed : 连接断开
failed --> pending : 尝试重连
connected --> disabled : 禁用状态
disabled --> pending : 启用重新连接
```

**图表来源**
- [types.py:66-77](file://src/openharness/mcp/types.py#L66-L77)

**章节来源**
- [client.py:36-57](file://src/openharness/mcp/client.py#L36-L57)
- [client.py:121-175](file://src/openharness/mcp/client.py#L121-L175)

### 工具调用实现

#### 参数序列化和结果处理

```mermaid
flowchart TD
Start([开始工具调用]) --> GetSession["获取服务器会话"]
GetSession --> ValidateArgs["验证输入参数"]
ValidateArgs --> CallTool["调用 session.call_tool()"]
CallTool --> ProcessContent["处理响应内容"]
ProcessContent --> ExtractText{"提取文本内容"}
ExtractText --> |是| AppendText["追加文本到结果"]
ExtractText --> |否| SerializeJSON["序列化为 JSON"]
SerializeJSON --> AppendJSON["追加 JSON 到结果"]
AppendText --> CheckStructured{"有结构化内容?"}
CheckStructured --> |是| UseStructured["使用结构化内容"]
CheckStructured --> |否| CheckParts{"有内容片段?"}
UseStructured --> JoinResults["连接结果"]
CheckParts --> |是| JoinResults
CheckParts --> |否| NoOutput["返回 '(no output)'"]
JoinResults --> ReturnResult["返回最终结果"]
NoOutput --> ReturnResult
ReturnResult --> End([结束])
```

**图表来源**
- [client.py:92-106](file://src/openharness/mcp/client.py#L92-L106)

#### 工具适配器模式

MCP 工具通过 `McpToolAdapter` 适配器转换为标准 OpenHarness 工具：

```mermaid
classDiagram
class BaseTool {
+string name
+string description
+BaseModel input_model
+execute(arguments, context) ToolResult
+is_read_only(arguments) bool
+to_api_schema() dict
}
class McpToolAdapter {
-McpClientManager _manager
-McpToolInfo _tool_info
+execute(arguments, context) ToolResult
}
class McpToolInfo {
+string server_name
+string name
+string description
+dict input_schema
}
BaseTool <|-- McpToolAdapter
McpToolAdapter --> McpToolInfo : 使用
McpToolAdapter --> McpClientManager : 依赖
```

**图表来源**
- [mcp_tool.py:14-34](file://src/openharness/tools/mcp_tool.py#L14-L34)
- [base.py:30-52](file://src/openharness/tools/base.py#L30-L52)

**章节来源**
- [mcp_tool.py:14-56](file://src/openharness/tools/mcp_tool.py#L14-L56)
- [base.py:30-76](file://src/openharness/tools/base.py#L30-L76)

### 资源访问实现

#### 资源读取流程

```mermaid
sequenceDiagram
participant Tool as ReadMcpResourceTool
participant Manager as McpClientManager
participant Session as ClientSession
participant Server as MCP 服务器
Tool->>Manager : read_resource(server_name, uri)
Manager->>Manager : 获取会话实例
Manager->>Session : session.read_resource(uri)
Session->>Server : 发送读取请求
Server-->>Session : 返回资源内容
Session-->>Manager : ReadResourceResult
Manager->>Manager : 处理内容类型
Manager->>Manager : 文本内容直接使用
Manager->>Manager : 二进制内容序列化
Manager-->>Tool : 字符串化结果
Tool-->>Tool : 返回 ToolResult
```

**图表来源**
- [client.py:108-119](file://src/openharness/mcp/client.py#L108-L119)

#### 资源列表工具

系统提供了专门的工具来列出所有可用的 MCP 资源：

**章节来源**
- [read_mcp_resource_tool.py:18-36](file://src/openharness/tools/read_mcp_resource_tool.py#L18-L36)
- [list_mcp_resources_tool.py:15-37](file://src/openharness/tools/list_mcp_resources_tool.py#L15-L37)

### 错误处理机制

系统实现了多层次的错误处理机制：

#### 连接错误处理
- 使用 `try-except` 块捕获连接异常
- 通过 `AsyncExitStack` 确保资源正确清理
- 设置详细的错误状态信息

#### 工具调用错误处理
- 捕获并处理工具调用异常
- 提供默认回退值（如 "(no output)"）
- 记录详细的错误信息

#### 资源访问错误处理
- 处理不同内容类型的读取
- 提供类型安全的内容访问
- 统一的字符串化输出

**章节来源**
- [client.py:166-175](file://src/openharness/mcp/client.py#L166-L175)

## 依赖分析

### 外部依赖

系统依赖以下关键外部库：

```mermaid
graph TB
subgraph "核心依赖"
A[mcp 库<br/>MCP 协议实现]
B[pydantic<br/>数据验证和序列化]
C[asyncio<br/>异步编程支持]
end
subgraph "内部模块"
D[openharness.mcp.client]
E[openharness.mcp.types]
F[openharness.mcp.config]
G[openharness.tools.base]
end
subgraph "测试依赖"
H[pytest<br/>测试框架]
I[unittest<br/>单元测试]
end
A --> D
B --> E
C --> D
D --> E
D --> F
G --> D
H --> I
```

**图表来源**
- [client.py:8-17](file://src/openharness/mcp/client.py#L8-L17)
- [types.py:5-8](file://src/openharness/mcp/types.py#L5-L8)

### 内部模块依赖

```mermaid
graph LR
A[client.py] --> B[types.py]
A --> C[config.py]
D[mcp_tool.py] --> A
D --> E[base.py]
F[read_mcp_resource_tool.py] --> A
F --> E
G[list_mcp_resources_tool.py] --> A
G --> E
H[test_integration.py] --> A
I[test_stdio_flow.py] --> A
I --> J[fake_mcp_server.py]
```

**图表来源**
- [client.py:12-17](file://src/openharness/mcp/client.py#L12-L17)
- [mcp_tool.py:9-11](file://src/openharness/tools/mcp_tool.py#L9-L11)

**章节来源**
- [client.py:12-17](file://src/openharness/mcp/client.py#L12-L17)
- [mcp_tool.py:9-11](file://src/openharness/tools/mcp_tool.py#L9-L11)

## 性能考虑

### 连接池管理

系统通过 `AsyncExitStack` 实现高效的资源管理：
- 自动清理已断开的连接
- 避免内存泄漏
- 支持优雅关闭

### 异步操作优化

- 使用 `async/await` 模式避免阻塞
- 并行处理多个服务器连接
- 流式处理大文件内容

### 缓存策略

- 工具和资源元数据缓存
- 连接状态快速查询
- 减少重复的网络往返

### 最佳实践建议

1. **连接复用**: 在应用生命周期内复用连接而非频繁创建/销毁
2. **超时设置**: 为长时间运行的操作设置合理的超时时间
3. **错误重试**: 实现指数退避的重试机制
4. **资源监控**: 监控连接状态和性能指标

## 故障排除指南

### 常见问题诊断

#### 连接失败
- 检查服务器命令路径是否正确
- 验证环境变量配置
- 确认服务器进程启动日志

#### 工具调用错误
- 验证输入参数格式
- 检查工具名称拼写
- 确认服务器支持该工具

#### 资源访问问题
- 验证资源 URI 格式
- 检查权限设置
- 确认资源存在性

### 调试技巧

使用测试服务器进行本地调试：
- 参考 `fake_mcp_server.py` 进行功能验证
- 利用集成测试检查完整流程
- 通过单元测试验证边界条件

**章节来源**
- [test_integration.py:17-80](file://tests/test_mcp/test_integration.py#L17-L80)
- [test_stdio_flow.py:16-55](file://tests/test_mcp/test_stdio_flow.py#L16-L55)

## 结论

OpenHarness 的 MCP 客户端实现提供了一个完整、健壮且易于使用的 MCP 客户端解决方案。该实现具有以下特点：

1. **模块化设计**: 清晰的分层架构便于维护和扩展
2. **异步支持**: 完全基于 asyncio 的异步实现
3. **类型安全**: 使用 Pydantic 模型确保数据完整性
4. **错误处理**: 完善的错误处理和恢复机制
5. **测试覆盖**: 全面的单元测试和集成测试

该实现为开发者提供了将 MCP 工具和资源无缝集成到 OpenHarness 生态系统的能力，同时保持了良好的性能和可靠性。

## 附录

### 完整使用示例

#### 基本连接配置

```python
# 创建 MCP 服务器配置
config = {
    "my_server": McpStdioServerConfig(
        command="python",
        args=["/path/to/server.py"],
        env={"API_KEY": "secret"}
    )
}

# 初始化客户端管理器
manager = McpClientManager(config)

# 连接所有服务器
await manager.connect_all()

# 检查连接状态
statuses = manager.list_statuses()
```

#### 工具调用示例

```python
# 获取工具注册表
registry = create_default_tool_registry(manager)

# 查找特定工具
tool = registry.get("mcp__my_server__hello")

# 执行工具
if tool:
    result = await tool.execute(
        tool.input_model.model_validate({"name": "world"}),
        ToolExecutionContext(cwd=Path("."))
    )
    print(result.output)
```

#### 资源访问示例

```python
# 列出所有资源
list_tool = registry.get("list_mcp_resources")
if list_tool:
    result = await list_tool.execute(
        list_tool.input_model(),
        ToolExecutionContext(cwd=Path("."))
    )
    print(result.output)

# 读取特定资源
resource_tool = registry.get("read_mcp_resource")
if resource_tool:
    result = await resource_tool.execute(
        resource_tool.input_model.model_validate({
            "server": "my_server",
            "uri": "mcp://example/resource"
        }),
        ToolExecutionContext(cwd=Path("."))
    )
    print(result.output)
```

#### 重连机制使用

```python
# 触发重连
await manager.reconnect_all()

# 或者单独更新配置
manager.update_server_config("my_server", new_config)
await manager.reconnect_all()
```

**章节来源**
- [test_stdio_flow.py:17-54](file://tests/test_mcp/test_stdio_flow.py#L17-L54)
- [test_integration.py:52-80](file://tests/test_mcp/test_integration.py#L52-L80)