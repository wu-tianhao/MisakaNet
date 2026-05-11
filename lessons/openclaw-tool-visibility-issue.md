---
title: OpenClaw 工具可见性问题：AI 拒绝使用已实现的功能
domain: ai-agents
tags: [openclaw, feishu, debugging, tool-visibility, ai-agent]
status: published
---

## 问题现象

OpenClaw 的 AI 拒绝使用某个工具的特定功能，声称「API 不支持」，但实际上该功能在代码层面已完整实现。

**具体案例：**
- 工具：`feishu_drive_file`
- 功能：`create_folder`（创建文件夹）
- AI 回复：「飞书云空间 API 目前不支持直接创建文件夹」
- 实际情况：代码中已实现，OAuth 已授权，权限已配置

## 根本原因

OpenClaw 的架构分为两层：

1. **工具层（Tools）**：在代码中注册，定义参数和执行逻辑
2. **技能层（Skills）**：提供给 AI 的文档，描述工具的使用方法

当 AI 处理请求时，会尝试读取 `/home/node/.openclaw/plugin-skills/<tool-name>/SKILL.md`。如果文件不存在：
- AI 无法获取完整的工具能力说明
- 依赖训练数据中的知识（可能过时）
- 过于保守，拒绝使用未在文档中明确说明的功能

**日志证据：**
```
[tools] read failed: ENOENT: no such file or directory, 
access '/home/node/.openclaw/plugin-skills/feishu-drive-file/SKILL.md'
```

## 诊断方法

### 1. 确认功能是否实现

检查工具源码：
```bash
# 查找工具实现
find /opt/1panel/apps/openclaw/OpenClaw/data/conf/extensions/openclaw-lark \
  -name "*.js" -path "*/tools/*" | grep <tool-name>

# 检查是否有目标 action
grep -n "action.*<action-name>" <tool-file.js>
```

### 2. 检查 SKILL.md 是否存在

```bash
ls -la /opt/1panel/apps/openclaw/OpenClaw/data/conf/plugin-skills/<tool-name>/
```

### 3. 查看日志确认问题

```bash
docker logs <container> 2>&1 | grep "SKILL.md"
```

## 解决方案

### 步骤 1：创建 SKILL.md 文档

在 `/opt/1panel/apps/openclaw/OpenClaw/data/conf/plugin-skills/<tool-name>/SKILL.md` 创建文档。

**文档结构：**
```markdown
# <tool_name> - 工具名称

## 概述
简要说明工具的用途

## 支持的操作

### ✅ <action_name> - 操作名称

**功能**：详细描述

**参数**：
- `param1`: 说明
- `param2`: 说明

**示例**：
\`\`\`json
{
  "action": "<action_name>",
  "param1": "value1"
}
\`\`\`

**返回**：
- `field1`: 说明
- `field2`: 说明

## 重要说明
- 明确说明功能已实现并可用
- 列出所需权限
- 提供使用场景示例
```

### 步骤 2：重启 OpenClaw

```bash
docker restart <openclaw-container>
```

### 步骤 3：使用明确的提示词

在飞书中发送：
```
请阅读 <tool-name> 技能文档，然后使用 <action_name> 功能...
```

## 关键路径参考

**OpenClaw 飞书插件：**
- 工具代码：`/home/node/.openclaw/extensions/openclaw-lark/src/tools/oapi/`
- 技能文档：`/home/node/.openclaw/plugin-skills/`
- OAuth tokens：`/home/node/.local/share/openclaw-feishu-uat/*.enc`（加密）
- 权限配置：`/home/node/.openclaw/extensions/openclaw-lark/src/core/tool-scopes.js`

## 验证方法

### 检查 OAuth 授权状态

```bash
# 查找 token 文件
docker exec <container> find /home/node/.local/share/openclaw-feishu-uat -name "*.enc"

# 查看授权日志
docker logs <container> 2>&1 | grep "saved UAT for"
```

### 检查权限配置

```bash
# 查看工具所需权限
grep "<tool_name>.<action>" \
  /opt/1panel/apps/openclaw/OpenClaw/data/conf/extensions/openclaw-lark/src/core/tool-scopes.js
```

## 经验总结

1. **代码实现 ≠ AI 可见**：工具在代码层面注册不等于 AI 知道如何使用
2. **SKILL.md 是关键**：这是 AI 理解工具能力的主要来源
3. **日志是线索**：`ENOENT: SKILL.md` 是典型的可见性问题信号
4. **明确的文档胜过隐式的代码**：即使工具描述在代码中存在，独立的 SKILL.md 能显著提高 AI 的使用意愿
5. **重启生效**：修改 SKILL.md 后需要重启容器

## 适用场景

- OpenClaw 拒绝使用某个工具的特定功能
- AI 说「不支持」但你确认代码中已实现
- 工具调用日志显示 SKILL.md 读取失败
- 需要为自定义工具创建文档

## 相关问题

**Q: 为什么其他飞书技能有 SKILL.md 但 feishu_drive_file 没有？**
A: 因为 feishu_drive_file 是底层工具（Tool），不是面向用户的技能（Skill）。其他如 feishu-create-doc 是封装好的技能，有完整文档。

**Q: 可以直接修改工具代码的 description 吗？**
A: 可以，但 SKILL.md 的优先级更高，且更容易被 AI 读取和理解。

**Q: OAuth token 为什么不在 oauth-tokens 目录？**
A: OpenClaw 的飞书插件使用加密存储，token 保存在 `.local/share/openclaw-feishu-uat/` 并以 `.enc` 扩展名加密。
