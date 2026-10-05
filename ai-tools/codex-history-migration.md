# 迁移 Codex 本地历史对话：从旧 Model Provider 切换到 OpenAI

> **适用场景**：在 Windows 上使用过 Codex / ChatGPT Desktop，并曾配置自定义或旧的 `model_provider`。切换到 OpenAI / ChatGPT 官方登录后，部分历史对话仍然可见，但打开旧 thread 时出现 `Model provider '<provider>' not found`，导致无法继续对话。
>
> **重要说明**：本文记录的是一次本地数据迁移实践，并非 OpenAI 官方提供的迁移方法。Codex / ChatGPT Desktop 的内部存储结构可能随版本变化。操作前务必完整备份 `.codex` 目录。

## 1. 问题背景

此前我的 Codex 环境使用过多个 model provider。后来切换到 ChatGPT 官方账号登录，并移除了旧 provider 配置。

新对话可以正常使用，但部分历史 thread 打开后出现类似错误：

```text
Model provider `legacy_provider` not found
```

一开始很容易认为这是 `config.toml` 的问题。

但实际测试发现，仅修改当前配置并不能让这些旧 thread 恢复，因为旧 thread 自身仍然保存着创建时使用的 provider 元数据。

因此，问题实际上分成了两部分：

```text
当前配置
    ↓
config.toml

历史会话
    ↓
state_5.sqlite + rollout JSONL
```

要恢复旧 thread，需要处理的是后者。

---

## 2. Codex 本地历史数据在哪里？

本次环境为 Windows。

Codex 的本地数据位于：

```text
C:\Users\<username>\.codex\
```

排查后，与历史 thread 迁移直接相关的主要有：

```text
.codex\
├── state_5.sqlite
└── sessions\
    └── YYYY\MM\DD\rollout-*.jsonl
```

其中：

- `state_5.sqlite`：保存 thread 索引和元数据；
- `rollout-*.jsonl`：保存具体 session 数据。

SQLite 中的 `threads` 表包含类似字段：

```text
id
title
model
model_provider
rollout_path
source
cwd
...
```

而对应 rollout JSONL 的第一行通常是：

```json
{
  "type": "session_meta",
  "payload": {
    "model_provider": "legacy_provider"
  }
}
```

也就是说，历史 thread 使用的 provider 信息并不只存在于当前配置文件中。

---

## 3. 先检查数据库中的 provider

在修改任何数据之前，可以先使用 SQLite **只读模式**检查当前 provider 分布。

CMD：

```cmd
python -c "import sqlite3; p=r'C:\Users\<username>\.codex\state_5.sqlite'; c=sqlite3.connect('file:'+p+'?mode=ro',uri=True); print(c.execute('SELECT model_provider,COUNT(*) FROM threads GROUP BY model_provider ORDER BY COUNT(*) DESC').fetchall()); c.close()"
```

可能会得到类似：

```text
[
    ('legacy_provider_a', ...),
    ('legacy_provider_b', ...),
    ('custom', ...),
    ('openai', ...)
]
```

这里不要看到旧 provider 就立即全部替换。

首先应该确认：

- 哪些 provider 确实属于需要迁移的旧环境；
- 哪些 provider 仍然可能被其他配置使用；
- 哪些 thread 已经属于 `openai`。

例如，本次只处理：

```text
legacy_provider_a -> openai
legacy_provider_b -> openai
```

其他来源不明确的 provider 暂时保留。

---

## 4. 修改之前：完整备份

### 4.1 完全退出 ChatGPT Desktop / Codex

在修改 SQLite 或 session 文件前，应先完全退出客户端。

不要只关闭窗口，还要确认程序没有继续在后台运行。

这样可以避免迁移过程中应用同时写入：

```text
state_5.sqlite
```

或者：

```text
sessions\...\rollout-*.jsonl
```

---

### 4.2 备份整个 `.codex`

在 CMD 中执行：

```cmd
xcopy "%USERPROFILE%\.codex" "%USERPROFILE%\.codex-backup-before-openai-migration\" /E /I /H /Y
```

备份完成后，可以检查：

```cmd
dir "%USERPROFILE%\.codex-backup-before-openai-migration"
```

备份目录位于：

```text
C:\Users\<username>\.codex-backup-before-openai-migration\
```

确认至少包含：

```text
state_5.sqlite
sessions\
...
```

再进行后续操作。

> 不建议仅依赖 Windows 的“以前的版本”或系统还原点。直接复制 `.codex` 可以获得一个明确、独立的迁移前快照。

---

## 5. 为什么不能只修改 `config.toml`？

旧 provider 可能原本在 `config.toml` 中定义，例如：

```toml
model_provider = "legacy_provider"
```

删除旧 provider 配置以后，新 session 可以使用当前 OpenAI provider。

但是旧 thread 本身仍然可能保存：

```text
model_provider = legacy_provider
```

因此打开旧 thread 时仍然会出现：

```text
Model provider `legacy_provider` not found
```

换句话说：

> 修改 `config.toml` 只改变当前配置，并不会自动迁移已经创建的历史 thread。

---

## 6. 为什么不能只修改 SQLite？

进一步检查发现，provider 信息至少同时存在于：

```text
state_5.sqlite
```

以及：

```text
rollout-*.jsonl
```

例如 SQLite 中：

```text
threads.model_provider = legacy_provider
```

而对应 JSONL 中：

```json
{
  "type": "session_meta",
  "payload": {
    "model_provider": "legacy_provider"
  }
}
```

因此，如果决定迁移历史 thread，更稳妥的做法是同时处理：

```text
SQLite thread metadata
            +
rollout session metadata
```

而不是只修改其中一处。

---

## 7. 先检查某一个 thread

在批量迁移之前，可以先查看数据库中的 thread：

```cmd
python -c "import sqlite3; p=r'C:\Users\<username>\.codex\state_5.sqlite'; c=sqlite3.connect('file:'+p+'?mode=ro',uri=True); rows=c.execute(\"SELECT id,title,model,model_provider,rollout_path FROM threads WHERE model_provider='legacy_provider_a' ORDER BY updated_at DESC\").fetchall(); [print(r) for r in rows[:20]]; c.close()"
```

可以得到类似：

```text
thread ID
title
model
model_provider
rollout_path
```

然后检查对应 rollout JSONL。

通常第一行包含：

```json
{
  "type": "session_meta",
  "payload": {
    "id": "...",
    "model_provider": "legacy_provider_a"
  }
}
```

这一步可以帮助确认：

```text
SQLite thread
        ↓
rollout_path
        ↓
对应 JSONL session
```

之间的关系。

---

## 8. 批量迁移脚本

确认需要迁移的 provider 后，可以创建：

```text
C:\Users\<username>\migrate_codex_threads.py
```

内容如下：

```python
import sqlite3
import json
import os

DB = r"C:\Users\<username>\.codex\state_5.sqlite"

# 修改为自己实际需要迁移的旧 provider。
LEGACY_PROVIDERS = {
    "legacy_provider_a",
    "legacy_provider_b",
}

NEW_PROVIDER = "openai"

conn = sqlite3.connect(DB)
cur = conn.cursor()

placeholders = ",".join("?" for _ in LEGACY_PROVIDERS)

rows = cur.execute(
    f"""
    SELECT id, title, model_provider, rollout_path
    FROM threads
    WHERE model_provider IN ({placeholders})
    """,
    tuple(LEGACY_PROVIDERS),
).fetchall()

print(f"Found {len(rows)} threads to migrate.")
print()

db_migrated = 0
json_migrated = 0
json_already_openai = 0
missing_files = []
json_errors = []

for thread_id, title, old_provider, rollout_path in rows:

    try:
        # 1. 检查 rollout 文件。
        if not rollout_path or not os.path.isfile(rollout_path):
            missing_files.append(
                (thread_id, title, rollout_path)
            )
            continue

        with open(rollout_path, "r", encoding="utf-8") as f:
            lines = f.readlines()

        if not lines:
            raise ValueError("Empty rollout file")

        # 2. 第一行应该是 session_meta。
        meta = json.loads(lines[0])

        if meta.get("type") != "session_meta":
            raise ValueError(
                f"First line is not session_meta: {meta.get('type')}"
            )

        payload = meta.get("payload", {})

        # 3. 验证 SQLite thread ID 与 JSON session ID。
        session_id = payload.get("session_id") or payload.get("id")

        if session_id != thread_id:
            raise ValueError(
                f"Thread ID mismatch: "
                f"DB={thread_id}, JSON={session_id}"
            )

        # 4. 检查 JSON 中的 provider。
        json_provider = payload.get("model_provider")

        if json_provider in LEGACY_PROVIDERS:

            payload["model_provider"] = NEW_PROVIDER

            lines[0] = (
                json.dumps(
                    meta,
                    ensure_ascii=False,
                    separators=(",", ":"),
                )
                + "\n"
            )

            with open(
                rollout_path,
                "w",
                encoding="utf-8",
            ) as f:
                f.writelines(lines)

            json_migrated += 1

        elif json_provider == NEW_PROVIDER:

            json_already_openai += 1

        else:

            raise ValueError(
                f"Unexpected JSON provider: "
                f"{json_provider!r}; "
                f"DB provider: {old_provider!r}"
            )

        # 5. 只有 JSONL 验证/修改成功以后，
        #    才修改对应 SQLite thread。
        cur.execute(
            f"""
            UPDATE threads
            SET model_provider = ?
            WHERE id = ?
              AND model_provider IN ({placeholders})
            """,
            (
                NEW_PROVIDER,
                thread_id,
                *LEGACY_PROVIDERS,
            ),
        )

        if cur.rowcount == 1:
            db_migrated += 1

    except Exception as e:

        json_errors.append(
            (
                thread_id,
                title,
                rollout_path,
                str(e),
            )
        )

        # 出现异常时跳过该 thread。
        continue


conn.commit()
conn.close()


print()
print("========== MIGRATION REPORT ==========")

print(f"Threads found:         {len(rows)}")
print(f"SQLite migrated:       {db_migrated}")
print(f"JSONL migrated:        {json_migrated}")
print(f"JSONL already openai:  {json_already_openai}")
print(f"Missing rollout files: {len(missing_files)}")
print(f"JSON errors:           {len(json_errors)}")


if missing_files:

    print("\n--- Missing rollout files ---")

    for item in missing_files:
        print(item)


if json_errors:

    print("\n--- JSON errors ---")

    for item in json_errors:
        print(item)


print("\nDONE")
```

需要修改两处。

首先把：

```text
<username>
```

替换为自己的 Windows 用户名。

然后把：

```python
LEGACY_PROVIDERS = {
    "legacy_provider_a",
    "legacy_provider_b",
}
```

替换成前面通过 SQLite 查询得到、并且已经确认需要迁移的 provider。

例如：

```python
LEGACY_PROVIDERS = {
    "old_provider_1",
    "old_provider_2",
}
```

---

## 9. 运行迁移

确保 ChatGPT Desktop / Codex 已经完全退出。

然后：

```cmd
python "C:\Users\<username>\migrate_codex_threads.py"
```

脚本会逐条检查历史 thread。

完成后会输出类似：

```text
========== MIGRATION REPORT ==========
Threads found:         ...
SQLite migrated:       ...
JSONL migrated:        ...
JSONL already openai:  ...
Missing rollout files: ...
JSON errors:           ...

DONE
```

---

## 10. 为什么脚本故意设计得比较保守？

这里没有直接执行：

```text
legacy_provider -> openai
```

的全局字符串替换。

脚本会逐个 thread 检查：

1. rollout 文件是否存在；
2. JSONL 是否为空；
3. 第一行是否能够解析；
4. 第一行是否为 `session_meta`；
5. SQLite thread ID 与 JSON session ID 是否一致；
6. JSON 中的 provider 是否属于预期 provider。

只有通过检查后，才修改：

```text
JSONL
+
SQLite
```

如果其中任何一步异常，就跳过该 thread。

脚本也不会主动修改：

```text
encrypted_content
消息正文
title
thread ID
cwd
model
reasoning_effort
其他 provider
已经属于 openai 的 thread
```

---

## 11. 实际遇到的一个坑：Thread ID mismatch

迁移过程中，我遇到过：

```text
Thread ID mismatch:
DB=<thread-id-A>
JSON=<session-id-B>
```

也就是说：

```text
SQLite thread ID
        ≠
rollout session ID
```

这说明并不是所有数据库记录与 `rollout_path` 指向的 session 都能简单假设为严格一一对应。

因此，本次没有取消校验去强行迁移这些记录，而是：

```text
正常记录
   ↓
迁移

异常记录
   ↓
跳过并保留原状
```

这样即使少数 thread 暂时没有迁移，也不会为了追求 100% 成功率而破坏本地历史数据。

---

## 12. 迁移后验证

迁移完成后，可以再次使用只读模式检查：

```cmd
python -c "import sqlite3; p=r'C:\Users\<username>\.codex\state_5.sqlite'; c=sqlite3.connect('file:'+p+'?mode=ro',uri=True); print('PROVIDERS:'); [print(r) for r in c.execute('SELECT model_provider,COUNT(*) FROM threads GROUP BY model_provider ORDER BY COUNT(*) DESC')]; print('\nTOTAL:',c.execute('SELECT COUNT(*) FROM threads').fetchone()[0]); c.close()"
```

检查重点不是具体数字，而是：

```text
旧 provider 数量下降
openai 数量增加
总 thread 数保持不变
```

本次实际迁移中，大部分旧 thread 成功转换为 `openai`，少量存在校验异常的 thread 被安全跳过。

随后重新启动 ChatGPT Desktop。

找到一个此前会出现：

```text
Model provider `<old-provider>` not found
```

的历史 thread。

检查：

1. 历史消息是否能够正常显示；
2. 是否不再出现 provider missing；
3. 输入框是否可以正常使用；
4. 是否能够发送新消息并获得回复。

本次测试中，迁移后的历史 thread 可以继续正常对话。

---

## 13. 排查过程中几个容易误判的地方

### 13.1 `model_provider` 不是 ChatGPT 账号

配置中：

```toml
model_provider = "..."
```

描述的是模型 provider，并不是 ChatGPT 用户名。

因此：

```text
切换 ChatGPT 登录账号
```

和：

```text
迁移本地 thread provider
```

实际上是两个不同的问题。

---

### 13.2 删除旧 provider 配置不会自动迁移历史

删除 `config.toml` 中的旧 provider 后：

```text
新 thread
    ↓
可以正常使用当前 provider
```

但：

```text
旧 thread
    ↓
仍然保存原 provider
    ↓
Model provider not found
```

因此，仅修改当前配置并不能解决旧 thread 的恢复问题。

---

### 13.3 客户端侧边栏不一定等于 SQLite 中的本地 thread 列表

排查过程中还发现，新版 ChatGPT Desktop 的侧边栏可能同时涉及不同来源的历史对话。

因此：

> 不要仅根据侧边栏中显示的标题，就假设它一定对应 `state_5.sqlite` 中某一条本地 thread。

特别是存在多个同名 thread 时，仅根据：

```text
title = "hi"
```

之类的条件定位并不可靠。

更适合使用：

```text
thread ID
rollout_path
provider
source
updated_at
```

等字段综合判断。

---

### 13.4 不要为了少数异常记录取消所有安全检查

如果绝大部分 thread 已经成功迁移，而少数出现：

```text
Thread ID mismatch
```

或者：

```text
Missing rollout file
```

更合理的做法通常是：

```text
先保留异常记录
```

而不是：

```text
取消验证
→ 全量强制替换
```

本地历史数据比迁移成功率更重要。

---

## 14. 如何回滚？

如果迁移后出现异常，先完全退出 ChatGPT Desktop / Codex。

因为迁移前已经完整备份：

```text
C:\Users\<username>\.codex-backup-before-openai-migration\
```

所以可以恢复到迁移前状态。

在真正覆盖当前 `.codex` 之前，建议把迁移后的版本也另外保留一份，例如：

```text
.codex-after-migration\
```

这样可以同时保留：

```text
迁移前
+
迁移后
```

两个状态，方便进一步排查。

---

## 15. 总结

这次问题表面上看是：

```text
切换登录账号后旧对话打不开
```

但进一步排查后发现，真正的问题是：

> 历史 Codex thread 会将创建时使用的 model provider 持久化到本地状态中。切换当前配置后，旧 thread 的 provider 元数据不会自动更新。

本次最终采用：

```text
完整备份 .codex
        ↓
只读分析 state_5.sqlite
        ↓
确认需要迁移的 provider
        ↓
定位 rollout JSONL
        ↓
逐条验证 session metadata
        ↓
同步修改 JSONL + SQLite
        ↓
异常 thread 跳过
        ↓
重新统计 provider
        ↓
启动客户端抽查旧 thread
```

迁移后，大部分旧 thread 可以在当前 OpenAI provider 下继续正常使用。

对于少数内部状态不一致的记录，则保留原状，后续单独分析。

---

## Disclaimer

本文基于一次具体 Windows + Codex / ChatGPT Desktop 环境下的实际排查结果，仅作为技术记录。

`state_5.sqlite`、rollout JSONL 以及其中字段均属于应用内部存储实现，并不是稳定的公开迁移 API。

未来版本可能修改：

- 数据库 schema；
- session 文件格式；
- provider 字段；
- thread 恢复机制；
- 本地与云端历史的同步方式。

因此，在其他版本使用本文方法前，请先检查当前实际数据结构。

**不要在没有完整备份的情况下直接修改 `.codex`。**
