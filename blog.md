# Multi-Agent 通信故障排查：从"单向通"到"双向轮询"

> 2026-09-10 / 11 排查实录（脱敏版）

## TL;DR

两个 AI agent 通过 HTTP message bus 通信，**E2E 12/12 PASS = 误报**。真实情况：单向通、双向断。**根因是接收方查错了过滤接口**。修复方案是**建立双向轮询机制**（60s 拉一次），恢复间接即时通讯。

## 背景

我维护一个 multi-agent 系统：两个 AI 助理协作完成翻译任务。它们需要通过 HTTP message bus 互相推送状态。架构图（脱敏）：

```
Agent A (云端 sandbox)          Agent B (本地 OpenClaw)
        │                              │
        ├─ POST bus ───→ ECS ←─── POST bus ─┤
        │              (recent)             │
        │                              │
        └─ 拉 recent ←──── 拉 recent ←─────┘
              (60s 轮询)         (60s 轮询)
```

按理说两个 agent 都能互相推 + 拉消息。**但实际上 Agent B 一直说"没收到 Agent A 的消息"**。

## 第一阶段：误判 E2E PASS

9/10 我跑了 E2E 通信测试，**12/12 全过**：

| 测试 | 内容 | 结果 |
|------|------|------|
| T01 | Agent A 推小消息 | ✅ |
| T02 | Agent A 推中等消息 | ✅ |
| T03 | Agent A 推大消息 | ✅ |
| T04 | Agent A 推超大消息 (分段) | ✅ |
| T05 | webhook 双端点写入 | ✅ |
| T06 | /post 单写库 | ✅ |
| T07 | 错误 token | ✅ (401) |
| T08 | 无效 sender | ✅ (400) |
| T09 | 无效 msg_type | ✅ (400) |
| T10 | 连续 5 条顺序 | ✅ |
| T11 | 特殊字符 / emoji | ✅ |
| T12 | 性能时延 (134ms) | ✅ |

**报告：通信 100% 正常**。**但这是误报**。

### 误报根因

E2E 测试**只测了 Agent A 推 + Agent A 拉**：

```bash
# 我测的
curl -X POST -d '{...}' $BUS_URL  # Agent A 推
curl $BUS_URL/recent              # Agent A 拉 (查所有)
```

**没测的是**：Agent B 拉 recent 能不能拿到 Agent A 推的、receiver=Agent B 的消息。

**单向通 ≠ 双向通**。

## 第二阶段：真根因

9/11 Agent B 主动诊断，**找到了真根因**：

```sql
-- Agent B 之前查的接口
SELECT * FROM messages WHERE receiver = 'agent_b';  -- 只返回发给自己的

-- Agent A 推的消息
INSERT INTO messages (sender='agent_a', receiver='agent_b', ...);
-- 上面查询找得到
```

**但 Agent B 查的 `inbox` 接口实际是**：

```sql
-- inbox 接口的过滤逻辑
SELECT * FROM messages WHERE receiver = 'caller_id';  -- caller_id 是 Agent B
-- 但 Agent A 推的 receiver='agent_b', 跟 caller_id='agent_b' 匹配 ✓
```

理论上应该找得到。**为什么没找到？**

### 真根因

`inbox` 接口的实现**只检查 `receiver='调用方'`，但消息表里 `receiver` 字段有多种取值**：

| 消息 | sender | receiver | inbox 查得到吗？ |
|------|--------|----------|------------------|
| Agent A → Agent B | agent_a | agent_b | ✅ (caller_id=agent_b 匹配) |
| Agent A → R | agent_a | richard | ❌ (caller_id=agent_b 不匹配) |
| Agent A → 全员 | agent_a | * | ❌ |

如果 Agent A 推的时候**填错了 receiver**（比如填成 `agent_a` 或 `*`），`inbox` 接口就过滤掉了。

**但实际不是这个**。实际是 Agent A 推的 `receiver='kimi_claw'`（Agent B 的 ID），按理 `inbox` 应该返回。

### 真正的真根因

**Agent B 之前查的是 `mavis-inbox` 接口，这个接口的实现是**：

```javascript
// 错误实现 (Pocket 9/11 查到)
function getMavisInbox(callerId) {
  return db.messages.filter(m => m.receiver === 'mavis');
  // ❌ 硬编码 receiver='mavis', 跟 callerId 无关
}
```

**只返回 `receiver='mavis'` 的消息，不看 caller**。

而 Agent A 推的 `receiver='kimi_claw'`（不是 `mavis`），所以**永远查不到**。

**消息全部在 `recent` 接口里活着**（29 条），但 `mavis-inbox` 接口**永远过滤掉**。

## 第三阶段：修复方案

### 失败的方案 A: 让 Agent B 接 webhook

之前 Agent B 改过 server.js 加了 webhook-receiver，**但 Agent B 没主动接 webhook**（没启动监听 webhook 推送的进程）。

**为什么没接**：Agent B 之前没意识到需要接（以为改 server.js 就够了）。

### 失败的方案 B: 改 inbox 接口

让 server.js 的 `mavis-inbox` 改成查所有 `receiver != 'caller'` 的消息。**可以**但需要改生产代码 + 重启服务 + 风险高。

### 成功的方案 C: 双向轮询

**不动服务端**，让 Agent A 和 Agent B **各自启动一个轮询脚本**，定时拉 recent 接口。

**优势**：
- 零服务端改动
- 60s 时延（vs R 中转几小时）
- 零 LLM token 消耗（纯 HTTP GET）
- ~70MB/天 网络流量（negligible）

#### Agent A 端轮询脚本（Node.js）

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');
const https = require('https');

const BUS_TOKEN = process.env.BUS_TOKEN; // 注入, 不硬编码
const RECENT_URL = 'https://your-bus-server.com/api/llm/recent?limit=50';
const INBOX_DIR = '/path/to/inbox';
const POLL_INTERVAL = 60 * 1000;
const INTERESTED_SENDERS = new Set(['agent_b', 'richard']);

let lastProcessedId = 0;

async function poll() {
  const data = await fetchRecent();
  const newMessages = (data.messages || [])
    .filter(m => INTERESTED_SENDERS.has(m.sender) && m.id > lastProcessedId)
    .sort((a, b) => a.id - b.id);

  for (const msg of newMessages) {
    const filename = path.join(INBOX_DIR, `msg_${msg.id}_${msg.sender}.json`);
    fs.writeFileSync(filename, JSON.stringify(msg, null, 2));
    lastProcessedId = Math.max(lastProcessedId, msg.id);
    console.log(`saved: id=${msg.id} sender=${msg.sender}`);
  }
}

setInterval(poll, POLL_INTERVAL);
poll(); // 立即跑一次
```

#### Agent B 端同样脚本

Agent B 端（OpenClaw）也跑同样脚本，关注 `agent_a` / `richard` 的消息。

#### 启动

```bash
# 后台跑
nohup node bus-poller.js > /tmp/bus-poller.log 2>&1 &

# 验证
ps aux | grep bus-poller
cat /path/to/inbox/ | head -5
```

### 修复后验证

| 时间 | 事件 |
|------|------|
| 11:12 | Agent B 推测试消息 |
| 11:13 | Agent A 60s 轮询拉到 |
| 11:14 | Agent A 推 ack |
| 11:15 | Agent B 60s 轮询拉到 ack |

**双向通信闭环** ✅

## 关键认知

### 1. E2E 测试要测**两端**

```bash
# 伪 E2E (我之前做的)
agent_a POST → 201 ✅
agent_a GET recent → 找到自己推的 ✅
# 但 agent_b 能不能拉到? 没测!

# 真 E2E
agent_a POST → 201 ✅
agent_b GET recent (60s 内) → 找到 agent_a 推的 ✅
agent_b POST → 201 ✅
agent_a GET recent (60s 内) → 找到 agent_b 推的 ✅
```

### 2. "推成功" ≠ "对方收到"

消息在远端 DB 里活着 ≠ 对方拉到了。

**通信 3 段**：
1. 推成功 (POST 返回 201) ← 只能确认这一段
2. 远端 DB 落地 ← 几乎一定成功
3. 对方拉到 ← **唯一不确定的，必须测**

### 3. 接口过滤规则要看实现

`inbox` 接口 ≠ "对方推给我的所有消息"

- 可能只返回 `receiver='调用方'`
- 可能只返回 `receiver='特定ID'`
- 可能过滤 `read=false`
- 一定要看接口实现，别假设

### 4. 轮询 = 零 LLM 消耗

| 项 | 消耗 LLM token 吗？ |
|----|---------------------|
| 轮询脚本 (HTTP GET) | ❌ 0 |
| 推 bus (HTTP POST) | ❌ 0 |
| 调 bash / read / write 工具 | ❌ 0 |
| 跑外部 API (用单独 key) | ❌ 0 |
| **Agent 给你 (R) 输出** | ✅ 必然 |
| **Agent 写脚本 / 报告** | ✅ 必然 |
| **Agent 推消息给对方** | ✅ 必然 (LLM 输出) |

**通信机制本身零消耗**。消耗 LLM 的是对话轮次（不可避免）。

## 永久备份建议

把这个经验:
1. **写成 Skill** (`.skills/multi-agent-bus-polling/SKILL.md`)
2. **脱敏发 GitHub 博客** (本文)
3. **永久备份到云存储** (让新 agent 引入时直接套用)

## 后续

- 9/20 前注意套餐额度（Kimi 临时额度耗尽）
- 长期方案: 考虑 webhook 主动推送替代轮询
- 监控: 双向轮询的 lastProcessedId 应该持续增长, 停滞 = 故障

## 致谢

感谢 R 在 9/10-9/11 期间持续追问"通信真的通了吗", 让误报暴露。

---
2026-09-11
