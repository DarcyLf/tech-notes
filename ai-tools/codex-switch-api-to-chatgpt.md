# Codex 从自定义 API / Model Provider 切换到 ChatGPT 账号登录

> 本文记录一次 Windows 环境下，将 Codex 从自定义 API / Model Provider 配置切换到 ChatGPT 官方账号登录的过程。
>
> 本文重点讨论的是 **Codex 的认证方式和 Model Provider 配置**，不是把 API 余额“转移”到 ChatGPT Plus。

## 1. 背景

此前，我的 Codex 使用自定义 Model Provider。

`~/.codex/config.toml` 中的配置大致类似：

```toml
model_provider = "custom_provider"
model = "some-model"

[model_providers.custom_provider]
name = "custom_provider"
base_url = "https://example.com"
wire_api = "responses"
requires_openai_auth = true
```

后来我希望停止使用这个自定义 Provider，改为：

```text
ChatGPT 账号
    ↓
Codex
    ↓
OpenAI 官方 Provider
```

也就是直接使用 ChatGPT 账号登录 Codex。

---

## 2. 先区分三个容易混淆的概念

在处理这个问题之前，需要先区分：

```text
ChatGPT Account
        ↓
身份 / ChatGPT Plan

Model Provider
        ↓
模型请求发送到哪里

API Key
        ↓
API 身份认证与 API Billing
```

它们不是同一个东西。

例如：

```toml
model_provider = "custom_provider"
```

这里的：

```text
custom_provider
```

并不是 ChatGPT 用户名。

它只是 Codex 使用的 Model Provider ID。

因此，在客户端界面中看到一个自定义 Provider 名称时，不应该直接把它理解成“当前登录账号”。

---

## 3. ChatGPT Plus 和 API Billing 不是一回事

这是切换过程中最容易误解的地方之一。

OpenAI 的 ChatGPT 和 API Platform 使用独立的 Billing 系统。

因此：

```text
ChatGPT Plus
```

并不等于：

```text
OpenAI API 余额
```

也不是：

```text
购买 Plus
    ↓
自动获得等额 API Token
```

但是，Codex 支持使用 ChatGPT 账号登录。

在这种模式下，符合条件的 Codex 使用量可以计入 ChatGPT Plan 所包含的 Codex Usage。

而如果使用自己的 API Key：

```text
Codex
    ↓
OpenAI API Key
    ↓
API Pricing
```

则按照 API 的计费体系计算。

因此可以简单理解为：

```text
方式 A

Sign in with ChatGPT
        ↓
ChatGPT Plan
        ↓
Codex Usage


方式 B

API Key
        ↓
OpenAI API Platform
        ↓
API Billing
```

两者不要混为一谈。

---

## 4. Codex 的配置文件在哪里？

在 Windows 中，用户级 Codex 配置通常位于：

```text
C:\Users\<username>\.codex\config.toml
```

也就是：

```text
~/.codex/config.toml
```

Codex 会从这里读取用户级默认配置。

例如：

```toml
model = "some-model"
model_provider = "custom_provider"
```

其中：

```toml
model_provider
```

决定使用哪个 Model Provider。

---

## 5. 自定义 Model Provider 是什么？

Codex 支持在：

```toml
[model_providers.<id>]
```

下面定义自定义 Provider。

例如：

```toml
model_provider = "my_provider"

[model_providers.my_provider]
name = "my_provider"
base_url = "https://example.com"
wire_api = "responses"
requires_openai_auth = true
```

这里：

```text
my_provider
```

只是一个自定义 Provider ID。

而：

```text
base_url
```

决定请求发送到哪个 Endpoint。

因此：

```text
ChatGPT 登录账号
```

和：

```text
model_provider
```

是两个不同层次的概念。

---

## 6. 我的目标

原来的结构大致是：

```text
Codex
   ↓
custom_provider
   ↓
Custom API Endpoint
```

希望改成：

```text
Codex
   ↓
OpenAI
   ↓
ChatGPT Account
```

因此，需要做两件事：

```text
移除旧的自定义 Provider 配置
+
使用 ChatGPT 账号完成 Codex 登录
```

---

## 7. 修改 `config.toml`

首先打开：

```text
C:\Users\<username>\.codex\config.toml
```

原来的配置可能类似：

```toml
model_provider = "custom_provider"
model = "some-model"

[model_providers.custom_provider]
name = "custom_provider"
base_url = "https://example.com"
wire_api = "responses"
requires_openai_auth = true
```

如果已经确定不再使用这个 Provider，可以移除：

```toml
model_provider = "custom_provider"
```

以及对应的：

```toml
[model_providers.custom_provider]
...
```

因为 Codex 内置默认 Provider 是：

```text
openai
```

所以不一定需要手动写：

```toml
model_provider = "openai"
```

也可以让 Codex 使用默认值。

例如最终配置可以只保留仍然需要的设置：

```toml
model = "..."
model_reasoning_effort = "high"
```

以及其他与 Provider 无关的个人配置。

---

## 8. 使用 ChatGPT 账号登录

Codex 支持直接使用 ChatGPT 账号登录。

不同 Codex 客户端的入口可能略有不同，例如：

```text
ChatGPT Desktop / Codex
Codex CLI
Codex IDE Extension
```

如果使用 Codex CLI，可以通过登录流程选择：

```text
Sign in with ChatGPT
```

完成浏览器授权。

完成后：

```text
ChatGPT Account
        ↓
Codex Authentication
        ↓
OpenAI Provider
```

不再需要把自定义 API Endpoint 作为默认 Provider。

---

## 9. 如何确认切换成功？

最简单的方法是创建一个新的 Codex thread。

如果：

```text
新 thread 可以正常发送消息
+
不再出现旧 Provider
+
请求可以正常返回
```

说明当前配置和认证基本已经切换成功。

也可以检查：

```text
~/.codex/config.toml
```

确认不再存在：

```toml
model_provider = "old_provider"
```

以及不再需要的：

```toml
[model_providers.old_provider]
```

配置。

---

## 10. 一个容易踩的坑：新对话正常，但旧对话打不开

完成上述切换以后，我遇到了另一个问题：

```text
新对话
    ↓
正常


旧对话
    ↓
Model provider `old_provider` not found
```

最开始我以为：

```text
是不是 config.toml 没改干净？
```

但进一步排查发现：

> 旧 Codex thread 会保留创建时使用的 `model_provider` 信息。

因此：

```text
删除旧 Provider 配置
```

并不会自动执行：

```text
所有历史 thread
old_provider -> openai
```

这也是为什么：

```text
当前配置已经正常
```

但：

```text
旧 thread 仍然报错
```

---

## 11. 配置迁移和历史迁移是两个问题

最终我把整个问题拆成了两层。

### 第一层：当前 Codex 配置

解决：

```text
Custom API / Provider
        ↓
ChatGPT Account + OpenAI
```

主要涉及：

```text
config.toml
+
Codex Authentication
```

### 第二层：历史 thread

解决：

```text
Old Thread
        ↓
Old model_provider
        ↓
OpenAI
```

主要涉及 Codex 的本地历史数据，例如：

```text
~/.codex/state_5.sqlite
```

以及：

```text
~/.codex/sessions/.../rollout-*.jsonl
```

这两个问题不要混在一起处理。

---

## 12. 如果旧历史仍然报错

如果切换完成后，新 thread 已经可以正常使用，但旧 thread 出现：

```text
Model provider `<old-provider>` not found
```

说明：

```text
当前配置
```

大概率已经没有问题。

真正需要处理的是：

```text
历史 thread 保存的 Provider Metadata
```

我把这一部分单独整理成了另一篇笔记：

[迁移 Codex 本地历史对话：从旧 Model Provider 切换到 OpenAI](./codex-history-migration.md)

其中记录了：

```text
完整备份 .codex
        ↓
分析 state_5.sqlite
        ↓
定位 rollout JSONL
        ↓
验证 thread metadata
        ↓
迁移 model_provider
        ↓
恢复旧 thread
```

---

## 13. 最终结构

完成两部分迁移以后，整体状态从：

```text
Codex
  │
  ├── Current Config
  │       ↓
  │   Custom Provider
  │
  └── Historical Threads
          ↓
      Old Provider
```

变成：

```text
Codex
  │
  ├── Current Config
  │       ↓
  │     OpenAI
  │
  └── Historical Threads
          ↓
        OpenAI

          +
          
    ChatGPT Account
```

这样：

```text
新 thread
+
迁移后的旧 thread
```

都可以继续在当前 Codex 环境中使用。

---

## 14. 总结

这次切换过程中，我觉得最值得记录的是下面几个概念：

### 1. `model_provider` 不是账号

```toml
model_provider = "xxx"
```

表示 Codex 使用哪个 Model Provider，而不是当前登录的 ChatGPT 用户名。

### 2. ChatGPT 和 API Billing 是两套体系

```text
ChatGPT subscription
≠
API billing
```

使用 ChatGPT 登录 Codex，与使用自己的 API Key，是不同的使用和计费路径。

### 3. 当前配置和历史 thread 是两层状态

```text
config.toml
```

控制当前 Codex 配置。

而历史 thread 可能仍然保存：

```text
old model_provider
```

所以：

```text
当前配置迁移成功
```

不代表：

```text
历史 thread 自动迁移成功
```

### 4. 先让新 thread 正常，再处理旧历史

比较安全的排查顺序是：

```text
确认 ChatGPT 登录
        ↓
清理旧 Provider 配置
        ↓
创建新 thread 测试
        ↓
确认当前环境正常
        ↓
再处理历史 thread
```

这样可以把：

```text
认证问题
配置问题
历史数据问题
```

拆开排查，而不是同时修改多个地方。

---

## References

- [Using Codex with your ChatGPT plan](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
- [Managing billing for ChatGPT and the API platform](https://help.openai.com/en/articles/9039756-managing-billing-settings-on-chatgpt-web-and-platform)
- [Codex configuration basics](https://developers.openai.com/zh-Hans/docs/config-file/config-basic)

---

## Disclaimer

本文记录的是一次具体环境下的配置迁移实践。

Codex 的登录流程、模型、配置项和客户端界面可能随版本更新而变化，因此实际操作时应优先参考 OpenAI 最新官方文档。

对于自定义 Model Provider，请在删除配置前确认该 Provider 已经不再被其他工作流使用。

如果还需要迁移本地历史 thread，请务必先完整备份：

```text
~/.codex
```

再修改任何 SQLite 或 rollout 文件。
