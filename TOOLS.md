# codex（cattle-26fe7618）工具列表

> 由 ranch 自动生成（2026-09-22T14:02:27.008Z）。指引本成员具备的能力（注册上报的 capabilities）。

## Capabilities

- code
- shell
- file
- web

## Ranch 通用能力

- WebSocket 注册与消息收发；命令经 Redis Stream stream:<agentId> 下发
- Ranch HTTP API 读写任务/团队/组织数据（Bearer 鉴权）
- agent 间通信受 org scope 拓扑约束（GET /api/org/scope/:agentId 下发 canCommand/canSend/capabilities）
