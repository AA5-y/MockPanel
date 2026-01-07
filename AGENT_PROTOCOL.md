# Agent 交互协议 (Agent Interaction Protocol)

## 概述

本文档定义了多智能体评审系统中各Agent的通信规则、状态转移和交互协议。

---

## 1. 通信模型

### 1.1 通道模型

```
                    ┌─────────────────────┐
                    │  review_panel 频道  │
                    └──────────┬──────────┘
                         │
                    ┌────┴────────────────┐
                    │ 接收所有消息       │
                    │ (Agent + Human)     │
                    └────┬────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    ┌────────┐      ┌────────┐      ┌────────────┐
    │ Agent1 │      │ Agent2 │  ... │ MetaAgent  │
    └────┬───┘      └────┬───┘      └────┬───────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                    send_channel_message()
                         │
                    ┌────▼────────────────┐
                    │ 发布回复至频道      │
                    │ (仅回应人类消息)    │
                    └─────────────────────┘
```

### 1.2 消息类型

| 消息类型 | 发送者 | 接收者 | 处理方式 |
|---------|--------|--------|---------|
| 人工输入 | 人类 | 所有Agent | 处理并发送回复 |
| Agent回复 | Any Agent | 其他Agent | **忽略**，调用 finish() |
| 系统消息 | 系统 | 所有Agent | 视具体类型处理 |

---

## 2. Agent 状态机

```
┌──────────────────────────────────────────────────────────────┐
│                      Agent 生命周期                          │
└──────────────────────────────────────────────────────────────┘

    ┌────────────┐
    │  启动      │
    └─────┬──────┘
          │
          ▼
    ┌─────────────────────┐
    │ 监听频道消息        │
    │ (react_to_all)      │
    └──────┬──────────────┘
           │
    ┌──────▼──────────────────────────────┐
    │ 消息来源判断                        │
    └──────┬────────────┬─────────────────┘
           │            │
      来自人类      来自Agent
           │            │
           ▼            ▼
    ┌────────────┐  ┌────────────┐
    │ 处理消息   │  │ 调用finish()│
    │ 生成回复   │  │ 立即返回   │
    └─────┬──────┘  └────────────┘
          │
      等待延迟
    (reaction_delay)
          │
          ▼
    ┌───────────────────┐
    │ 发送回复          │
    │ send_channel_     │
    │ message()         │
    └─────┬─────────────┘
          │
          ▼
    ┌────────────┐
    │ 继续监听   │
    └────────────┘
```

---

## 3. 角色特定的交互模式

### 3.1 5 位"问题制造者"评委的交互

```
学生消息
    ↓
[Delay: 2-4秒]
    ↓
检查: 是否来自人类? → 否 → finish()
           ↓ 是
评委分析消息内容
    ↓
生成专业视角的问题
    ↓
[send_channel_message()]
    ↓
问题发布到频道
    ↓
继续监听...
```

**关键特性：**
- 独立且并行工作
- 不等待其他评委
- 不回应其他评委的消息
- 响应延迟：2-4秒

### 3.2 Metaassistant "问题收敛者" 的交互

```
观察阶段
    ↓
监听所有消息
    ↓
    ├─ 人类直接提问? → 立即回应建议
    └─ 学生回答评委? → 累积观察
    ↓
[收集足够信息后]
    ↓
综合分析
    ├─ 识别反复问题
    ├─ 分类：设计/工程/技术/交叉/鲁棒性
    ├─ 优先级排序
    └─ 生成建议
    ↓
[send_channel_message()]
    ↓
发布综合反馈
    ↓
准备输出最终建议
```

**关键特性：**
- 更长的响应延迟：3-6秒（思考时间）
- 消息更长：≤200字（包含结构化信息）
- 观察角度：全局视图
- 干预时机：
  - 学生回答后
  - 评委追问进行中
  - 需要澄清矛盾时

---

## 4. 通信规则详解

### 4.1 消息过滤规则

```python
# 伪代码：Agent处理逻辑

def on_message_received(message):
    # 规则1：判断消息来源
    if message.sender_type == "agent":
        # 来自其他Agent，立即结束
        finish()
        return
    
    # 规则2：检查消息是否与自己相关
    if should_respond(message):
        # 规则3：执行响应延迟
        delay = random_delay(config.reaction_delay)
        sleep(delay)
        
        # 规则4：生成响应
        response = generate_response(message)
        
        # 规则5：发送到频道
        send_channel_message(response)
    else:
        # 监听但不回应
        pass
```

### 4.2 响应延迟的意义

| 参数 | 默认值 | 效果 | 场景 |
|------|--------|------|------|
| reaction_delay | random(2,5) | 自然讨论节奏 | 常规评审 |
| 更快 random(1,2) | 快速反馈 | 紧急讨论 |
| 更慢 random(5,10) | 深思熟虑 | 复杂问题 |

### 4.3 发送消息的标准格式

```yaml
# Agent 发送消息时调用
send_channel_message(
  channel="review_panel",
  content="""
  [AGENT_NAME/角色]: 
  <核心观点或问题>
  
  具体分析...
  
  <1-2个探问问题>
  """,
  message_type="agent_response"
)
```

---

## 5. 多Agent并发协调

### 5.1 并发场景：多个评委同时响应

```
时间线：
t=0秒    学生提交消息
         ↓
t=2.3秒  designprof 发送回复① (delay: 2-3秒)
t=3.1秒  csprof 发送回复②     (delay: 3秒)
t=2.8秒  mechprof 发送回复③   (delay: 2.8秒)
t=4.2秒  xdisciplineprof 回复④ (delay: 4秒)
t=3.9秒  trollprof 回复⑤       (delay: 3.9秒)
         ↓
t=5.5秒  metaassistant 发送汇总 (delay: 5.5秒，思考期更长)

结果：
频道中消息顺序 = 发布顺序，不是分类顺序
```

**特点：**
- 评委各自独立工作，无显式同步
- 消息发布顺序由延迟决定
- Metaassistant 延迟更长，确保在所有评委之后

### 5.2 避免Agent之间的回应循环

```
❌ 错误场景 (禁止)：
designprof 消息 → csprof 检测到 → csprof 回应
                                    ↓
                              触发循环...

✅ 正确实现：
designprof 发送 (source: agent)
    ↓
csprof 接收：if source == agent → finish() 立即返回
    ↓
不生成回复，不触发链式反应
```

---

## 6. 评审流程中的Agent协调

### 6.1 第一阶段：问题提出

```
Timeline:
├─ t=0     学生提交汇报
├─ t=2~5   5位评委并行提问 (各自独立)
├─ t=6~7   metaassistant 汇总问题
└─ 输出    "这是5个关键问题区域"
```

### 6.2 第二阶段：学生回答

```
Timeline:
├─ t=8     学生开始回答Q1
├─ t=9     学生继续回答Q2, Q3...
├─ t=12    metaassistant 监听并记录
└─ 输出    "根据你的回答，这些需要改进..."
```

### 6.3 第三阶段：评委追问

```
Timeline:
├─ t=13    designprof 对回答追问
├─ t=15    csprof 对回答追问
├─ t=17    metaassistant 标记关键矛盾点
└─ 输出    "以下是核心改进方向..."
```

### 6.4 第四阶段：最终总结

```
Timeline:
├─ t=20    所有追问结束
├─ t=22    metaassistant 准备最终报告
└─ t=23    发布综合改进建议 (200字以内)
```

---

## 7. 故障处理和超时

### 7.1 Agent 无响应

```
检测条件：
message_timeout = 30秒
agent_timeout = 180秒

如果agent在30秒内未响应：
1. 系统发出"等待中"提示
2. 人类可手动重新@agent
3. 如果180秒仍无响应，标记为离线

处理：
- 检查logs/agents/[agent_name].log
- 查看是否有运行时错误
- 重启该agent
```

### 7.2 消息堆积

```
条件：message_queue_size = 1000

如果消息堆积超过队列大小：
1. 新消息被拒绝
2. 提示"频道拥塞"
3. 解决：清空旧消息或增加队列大小
```

---

## 8. 协议扩展性

### 8.1 添加新的专业评委

模板：

```yaml
type: "openagents.agents.collaborator_agent.CollaboratorAgent"
agent_id: "new_prof"

config:
  model_name: "auto"
  instruction: |
    You are NEW_PROF ([角色描述]) in a multi-agent panel.
    
    YOUR ROLE:
    - [专业视角]
    
    RULES:
    1. Keep responses under 150 words
    2. Use send_channel_message to respond
    3. Only respond to human messages
    4. Ask 1-2 probing questions per response
    5. Focus on [专业领域]

  react_to_all_messages: true
  reaction_delay: "random(2, 4)"

mods:
  - name: "openagents.mods.workspace.messaging"
    enabled: true

connection:
  host: "localhost"
  port: 8700
  transport: "grpc"
```

### 8.2 修改通信机制

如需实现更复杂的协调（如显式投票、多轮同步等），可扩展：
- 使用 `send_direct_message()` 实现点对点通信
- 实现 Custom Mod 处理特定协议
- 在 Metaassistant 中实现状态机

---

## 9. 参考代码片段

### 9.1 标准Agent响应代码

```python
class ProfessorAgent(CollaboratorAgent):
    def on_message(self, message):
        # 过滤：只回应人类
        if message.sender_type != "human":
            self.finish()
            return
        
        # 分析消息
        content = message.content
        
        # 生成回复
        response = self.llm.call(
            system_prompt=self.instruction,
            user_message=content
        )
        
        # 发送
        self.send_channel_message(
            channel="review_panel",
            content=response
        )
```

### 9.2 Metaassistant 综合分析

```python
class MetaAssistant(CollaboratorAgent):
    def __init__(self):
        self.collected_feedback = []
        self.prof_roles = [
            "design", "mech", "cs", "xdiscipline", "troll"
        ]
    
    def on_message(self, message):
        if message.sender_type != "human":
            self.finish()
            return
        
        # 累积反馈
        self.collected_feedback.append(message)
        
        # 当收集足够信息时触发综合
        if self.should_synthesize():
            synthesis = self.synthesize_feedback()
            self.send_channel_message(
                channel="review_panel",
                content=synthesis
            )
```

---

## 10. 最佳实践

### 10.1 系统管理员

- ✅ 定期检查所有agent的连接状态
- ✅ 监控消息延迟，确保系统响应迅速
- ✅ 定期备份频道对话记录
- ✅ 根据反馈调整agent指令

### 10.2 教师/评审员

- ✅ 在学生提交前确认所有agent在线
- ✅ 给予足够时间让metaassistant综合分析
- ✅ 允许学生有充分时间回答
- ✅ 鼓励学生主动追问澄清

### 10.3 学生

- ✅ 准备清楚、结构化的汇报内容
- ✅ 耐心等待所有评委的反馈
- ✅ 记录关键问题，系统整理回答
- ✅ 根据metaassistant的建议优化方案

---

## 附录：完整消息流示例

```
时间  发送者           消息内容                  接收处理
────────────────────────────────────────────────────────
0:00  学生             "我的项目是..."          全部agent处理
0:02  designprof      "关于UI设计的问题..."    其他agent检测来源→finish()
0:03  csprof         "关于算法复杂度..."      其他agent检测来源→finish()
0:04  mechprof       "关于成本估算..."        其他agent检测来源→finish()
0:05  trollprof      "如果...会怎样"         其他agent检测来源→finish()
0:06  xdisciplineprof "社会影响考量..."       其他agent检测来源→finish()
0:07  metaassistant   "综合问题分类如下..."    其他agent检测来源→finish()
0:15  学生             "关于Q1的回答是..."      全部agent监听
0:17  designprof      "但是..."进一步追问      其他agent检测来源→finish()
0:20  metaassistant   "最终改进建议..."       其他agent检测来源→finish()
```

