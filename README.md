# Multi-Agent Bus 通信经验

> 排查实录：从"单向通"误报，到"双向轮询"修复

## TL;DR

两个 AI agent 通过 HTTP message bus 通信，**E2E 12/12 PASS = 误报**。真实情况：单向通、双向断。

**根因**：接收方查了过滤接口（只查 `receiver='自己'`），错过了 `receiver='对方'` 的消息。

**修复**：建立**双向轮询机制**（60s 拉一次 recent 接口），恢复间接即时通讯。

📖 完整故事：[`blog_multi_agent_bus.md`](./blog_multi_agent_bus.md)

## 包含

- 📝 **博客文章**: 完整故障排查记录（脱敏版）
- 🛠️ **Skill**: 新 AI agent 可自动加载, 故障时触发排查
- 📜 **协议**: 双方协作的实施规范
- 🔧 **脚本**: 轮询脚本模板（Node.js）

## 快速开始

如果你也遇到"两个 AI agent 通信断"的问题:

```bash
# 1. 加载 skill
# (在你的 agent 平台加载 .skills/multi-agent-bus-polling/SKILL.md)

# 2. 跑诊断 (4 步)
# - 推消息方: HTTP 201 吗?
# - 查 recent 接口: 找得到吗?
# - 查 inbox 接口: 找得到吗? (对比)
# - 接收方: 60s 拉 recent, 落地目录有文件吗?

# 3. 跑修复
nohup node bus-poller.js > /tmp/bus-poller.log 2>&1 &

# 4. 验证双向
# Agent A 推 → 60s 内 Agent B 拉到 → Agent B 推 → 60s 内 Agent A 拉到
```

## 关键认知

- **单向通 ≠ 双向通**: E2E 必须测两端
- **推成功 ≠ 对方收到**: 唯一不确定的是"对方拉没拉"
- **轮询 = 零 LLM token**: 纯 HTTP GET
- **`inbox` 接口 ≠ 全消息**: 看实现

详见 [博客文章](./blog_multi_agent_bus.md)。

## 贡献

欢迎贡献:
- 其他 multi-agent 系统的通信方案
- 轮询脚本改进
- 故障排查经验

## License

MIT
