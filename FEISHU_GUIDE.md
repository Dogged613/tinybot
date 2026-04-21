# nanobot 飞书使用说明
  核心结论：gateway 不能关闭，它是机器人的大脑，关掉就下线了。                                                 
                                                                                                               
  如果不想一直开着终端，用这个命令让它在后台跑：                                                               
  cd ~/Agent/tinybot && source .venv/bin/activate                                                              
  nohup nanobot gateway > ~/.nanobot/gateway.log 2>&1 &                                                        
                                                            
  机器人目前能做的事概括：对话问答、操作你电脑上的文件、执行终端命令、联网搜索、定时任务、跨对话记忆。可以直接 
  在飞书里试试发「你能做什么」让它自我介绍。  
## 一、关于后台服务

**nanobot gateway 必须保持运行**，机器人才能收发消息。

关闭终端或停止脚本后，机器人会立即离线，飞书发消息不会有任何回复。

### 推荐：让服务在后台持续运行

每次使用前在终端执行：
```bash
cd ~/Agent/tinybot
source .venv/bin/activate
nanobot gateway
```

如果你想关掉终端窗口但保持服务运行，可以用 `nohup`：
```bash
nohup nanobot gateway > ~/.nanobot/gateway.log 2>&1 &
```

查看日志：
```bash
tail -f ~/.nanobot/gateway.log
```

停止服务：
```bash
pkill -f "nanobot gateway"
```

---

## 二、机器人能做什么

### 1. 日常对话与问答
直接发消息即可，支持连续多轮对话，机器人会记住上下文。

### 2. 文件操作
- 读取、创建、编辑文件
- 搜索文件内容（grep/glob）
- 示例：「帮我在桌面创建一个 todo.txt，内容是今天的待办事项」

### 3. 执行终端命令
机器人可以在你的电脑上运行 shell 命令：
- 示例：「查看我的磁盘空间使用情况」
- 示例：「帮我压缩 Downloads 文件夹下所有 jpg 图片」

### 4. 网络搜索与网页阅读
- 搜索最新资讯
- 读取指定网页内容并总结
- 示例：「搜索今天的 A 股市场行情并总结」
- 示例：「读取这个网页并帮我提取关键信息：https://xxx.com」

### 5. 定时任务（Cron）
让机器人在指定时间自动执行任务：
- 示例：「每天早上 8 点提醒我查看邮件」
- 示例：「每周一早上发给我本周天气预报」

### 6. 记忆系统
机器人会记住你的偏好和重要信息，跨对话持久保存：
- 示例：「记住我的名字叫 yishan，我是一名开发者」
- 下次对话时机器人会记得这些信息

### 7. 子任务拆解（SubAgent）
复杂任务自动拆解为多个子步骤并行执行：
- 示例：「帮我分析这个项目的代码质量，找出潜在问题」

### 8. 心跳定时任务（Heartbeat）
编辑 `~/.nanobot/workspace/HEARTBEAT.md`，设置每 30 分钟自动执行的周期任务：
```markdown
## Periodic Tasks
- [ ] 检查天气并发送摘要
- [ ] 扫描重要邮件
```
也可以直接对机器人说：「帮我添加一个每天检查天气的周期任务」

---

## 三、飞书内置命令

在飞书对话框直接发送以下命令：

| 命令 | 说明 |
|------|------|
| `/new` | 开启新对话（清除当前上下文） |
| `/stop` | 停止当前正在执行的任务 |
| `/restart` | 重启机器人 |
| `/status` | 查看机器人当前状态 |
| `/dream` | 立即触发记忆整理（Dream 系统） |
| `/dream-log` | 查看最近一次记忆变更内容 |
| `/dream-restore` | 列出可恢复的记忆版本 |
| `/help` | 查看所有可用命令 |

---

## 四、飞书配置回顾（排障备忘）

配置文件位置：`~/.nanobot/config.json`

飞书关键配置：
```json
"feishu": {
  "enabled": true,
  "appId": "cli_a96e6b1a11389cc8",
  "appSecret": "Jws94oHI2o8QdRhLVS4CSbnplrYgZ8Zn",
  "allowFrom": ["*"],
  "streaming": true,
  "domain": "feishu"
}
```

**必须开通的飞书权限：**
- `im:message`（发送消息）
- `im:message.p2p_msg:readonly`（接收私聊消息）
- `cardkit:card:write`（流式回复）

**事件订阅：**
- `im.message.receive_v1`，订阅方式选**长连接**

每次修改飞书应用权限后，必须重新**发布新版本**才能生效。

---

## 五、常见问题

**Q：发消息没有回复？**
1. 确认 `nanobot gateway` 正在运行
2. 确认飞书应用权限已包含 `im:message.p2p_msg:readonly`
3. 确认已发布最新版本
4. 从飞书工作台找到应用，而不是搜索联系人

**Q：机器人回复很慢？**
正常现象，机器人在执行工具调用时需要一些时间。飞书会显示「👍 处理中」的表情回应，表示已收到消息。

**Q：如何让机器人只响应特定人？**
将 `allowFrom` 改为指定的飞书 open_id：
```json
"allowFrom": ["ou_你的openid"]
```
open_id 可以在 gateway 日志里发消息时看到。

**Q：想换一个 AI 模型？**
修改 `~/.nanobot/config.json` 中的 `model` 字段，重启 gateway 生效。

---

*文档生成时间：2026-04-19*
