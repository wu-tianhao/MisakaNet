---
title: OpenClaw 拉取聊天记录权限配置：三个必需条件
domain: ai-agents
tags: [openclaw, feishu, permissions, oauth, chat-history]
status: published
---

## 问题现象

OpenClaw 无法拉取与特定用户或机器人的单聊记录，出现以下错误：

**错误 1：权限不足**
```
code: 41050
msg: 'no user authority error'
fetching user ou_xxx
```

**错误 2：找不到 P2P 会话**
```
feishu_im_user_get_messages: resolving P2P chat for open_id=ou_xxx
batch_query: no p2p chat found for open_id=ou_xxx
```

**错误 3：上下文溢出**
```
Model context limit reached. Conversation size exceeds model capacity.
```

## 根本原因

OpenClaw 拉取单聊记录需要满足三个条件，缺一不可：

### 1. 权限配置不完整

`feishu_im_user_get_messages` 工具需要查询用户信息来：
- 解析 P2P 会话的 chat_id
- 获取对方用户的名称和基本信息
- 格式化消息内容中的发送者信息

这些操作依赖于 `contact:user.*` 权限。如果权限缺失，会触发 `41050 no user authority error`。

### 2. 用户未授权新权限

即使在飞书开放平台添加了权限，用户的 OAuth token 仍然是旧的，不包含新权限。必须重新授权才能获取包含新权限的 token。

### 3. 提示词格式错误

OpenClaw 需要通过 `@` 提及来识别目标用户的 open_id。如果只写用户名而不使用 `@`，OpenClaw 可能会：
- 尝试搜索用户名（可能找不到）
- 错误地使用当前用户的 open_id（导致 "no p2p chat found"）

## 诊断方法

### 1. 检查权限配置

查看 `tool-scopes.js` 中的权限映射：

```bash
grep "feishu_im_user_get_messages" \
  /opt/1panel/apps/openclaw/OpenClaw/data/conf/extensions/openclaw-lark/src/core/tool-scopes.js
```

应该包含：
```javascript
'feishu_im_user_get_messages.default': [
    'im:chat:read',
    'im:message:readonly',
    'im:message.group_msg:get_as_user',
    'im:message.p2p_msg:get_as_user',
    'contact:contact.base:readonly',
    'contact:user.base:readonly',
]
```

### 2. 检查用户授权状态

在飞书中发送：
```
/feishu doctor
```

查看诊断报告中的：
- 应用已开通的权限列表
- 用户已授权的权限列表
- 两者的对比结果

### 3. 查看日志确认问题

```bash
# 查看权限错误
docker logs <openclaw-container> 2>&1 | grep "41050"

# 查看 P2P 会话解析过程
docker logs <openclaw-container> 2>&1 | grep "resolving P2P"

# 查看消息拉取结果
docker logs <openclaw-container> 2>&1 | grep "feishu_im_user_get_messages"
```

## 解决方案

### 步骤 1：添加必需权限

在飞书开放平台（https://open.feishu.cn/app）添加以下权限：

**必需的用户权限（User Scopes）：**
- `contact:user.base:readonly` - 读取用户基本信息
- `contact:user.employee:readonly` - 读取用户员工信息
- `contact:contact.base:readonly` - 读取通讯录基本信息

**已有的消息权限（应该已配置）：**
- `im:chat:read` - 读取会话信息
- `im:message:readonly` - 读取消息内容
- `im:message.group_msg:get_as_user` - 以用户身份读取群消息
- `im:message.p2p_msg:get_as_user` - 以用户身份读取单聊消息

添加权限后，需要创建新版本并发布。

### 步骤 2：触发用户重新授权

在飞书中给 OpenClaw 发送：

```
/feishu auth
```

OpenClaw 会返回授权链接，点击后：
1. 查看新增的权限列表
2. 点击"同意授权"
3. 授权完成后自动跳转回飞书

**验证授权成功：**

```bash
# 查看 token 文件更新时间
docker exec <openclaw-container> ls -lt /home/node/.local/share/openclaw-feishu-uat/

# 应该看到 .enc 文件的时间是刚才授权的时间
```

### 步骤 3：使用正确的提示词格式

**✅ 正确格式：**

```
@OpenClaw 请拉取我和 @Hermes 今天的单聊记录（最多 10 条），用 2 句话总结我们讨论了什么
```

```
@OpenClaw 请拉取我和 @cc-connect 最近 3 小时的对话（最多 15 条），列出我发送的主要指令
```

**❌ 错误格式：**

```
@OpenClaw 请拉取我和 Hermes 的聊天记录
```
（缺少 @ 提及，OpenClaw 无法识别目标用户）

```
@OpenClaw 请拉取我和 @Hermes 最近 30 天的所有对话
```
（时间范围太大，可能导致上下文溢出）

### 步骤 4：控制数据量（避免上下文溢出）

**推荐的参数：**
- **时间范围**：今天、昨天、最近 N 小时（N ≤ 24）
- **消息数量**：最多 10-15 条
- **输出要求**：要求"总结"而不是"列出所有消息"

**示例：**

```
@OpenClaw 请拉取我和 @Hermes 今天的单聊记录（最多 10 条），
用 3 句话总结：
1. 我们讨论了什么主题
2. 有哪些待办事项
3. 有什么需要跟进的
```

## 验证方法

### 检查权限是否生效

```bash
# 查看最新的授权日志
docker logs <openclaw-container> 2>&1 | grep "saved UAT for" | tail -1

# 查看是否还有 41050 错误
docker logs <openclaw-container> --since 5m 2>&1 | grep "41050"
# 应该没有输出
```

### 检查消息拉取是否成功

```bash
# 查看 P2P 会话解析
docker logs <openclaw-container> --since 5m 2>&1 | grep "resolving P2P"

# 查看消息拉取结果
docker logs <openclaw-container> --since 5m 2>&1 | grep "list: returned"
# 应该看到类似：list: returned 10 messages, has_more=false
```

### 检查是否有上下文溢出

```bash
# 查看是否有上下文限制错误
docker logs <openclaw-container> --since 5m 2>&1 | grep "context limit"
# 应该没有输出
```

## 关键路径参考

**OpenClaw 飞书插件：**
- 权限配置：`/home/node/.openclaw/extensions/openclaw-lark/src/core/tool-scopes.js`
- 消息读取工具：`/home/node/.openclaw/extensions/openclaw-lark/src/tools/oapi/im/message-read.js`
- OAuth tokens：`/home/node/.local/share/openclaw-feishu-uat/*.enc`（加密存储）
- 技能文档：`/home/node/.openclaw/extensions/openclaw-lark/skills/feishu-im-read/SKILL.md`

**宿主机路径（Docker 挂载）：**
- 配置目录：`/opt/1panel/apps/openclaw/OpenClaw/data/conf/`
- 权限配置：`/opt/1panel/apps/openclaw/OpenClaw/data/conf/extensions/openclaw-lark/src/core/tool-scopes.js`

## 经验总结

1. **权限配置 ≠ 用户授权**：在飞书开放平台添加权限后，用户必须重新授权才能生效
2. **@ 提及是必需的**：OpenClaw 通过 @ 提及来识别目标用户的 open_id
3. **控制数据量是关键**：避免一次性拉取太多消息导致上下文溢出
4. **使用 /feishu doctor 诊断**：这是最快的权限状态检查方法
5. **日志是最好的调试工具**：通过日志可以清楚看到权限错误、会话解析、消息拉取的完整过程
6. **要求总结而不是列出**：让 AI 总结内容而不是返回所有消息，可以大幅减少上下文占用

## 适用场景

- OpenClaw 无法拉取与用户或机器人的单聊记录
- 出现 "41050 no user authority error" 错误
- 出现 "no p2p chat found" 错误
- 需要汇总和分析与特定用户的对话内容
- 需要追溯与机器人的交互历史

## 相关问题

**Q: 为什么添加了权限还是报 41050 错误？**
A: 因为用户还在使用旧的 OAuth token。必须通过 `/feishu auth` 重新授权，获取包含新权限的 token。

**Q: 为什么提示 "no p2p chat found"？**
A: 可能的原因：
1. 提示词中没有使用 @ 提及目标用户
2. OpenClaw 错误地使用了当前用户的 open_id（你不能和自己聊天）
3. 确实没有与该用户的单聊记录

**Q: 可以拉取群聊记录吗？**
A: 可以，但使用不同的方式：
- 单聊：`@OpenClaw 请拉取我和 @用户 的单聊记录`
- 群聊：`@OpenClaw 请拉取"群名称"今天的聊天记录`
- 搜索：`@OpenClaw 搜索包含"关键词"的消息`

**Q: 为什么有时候拉取成功但 AI 没有回复？**
A: 可能是上下文溢出。解决方法：
1. 缩小时间范围（今天、最近 N 小时）
2. 限制消息数量（最多 10-15 条）
3. 明确要求"总结"而不是"列出所有消息"

**Q: contact:user.* 权限具体包括哪些？**
A: 主要包括：
- `contact:user.base:readonly` - 用户基本信息（姓名、头像）
- `contact:user.employee:readonly` - 员工信息（部门、职位）
- `contact:contact.base:readonly` - 通讯录基本信息

**Q: 机器人的名字为什么显示为 ou_xxx？**
A: 这是已知的显示问题，不影响核心功能。机器人账号可能需要不同的 API 来查询名称，但消息内容本身是完整的。
