# Multi-Agent 设计专业学生汇报评审系统

一个基于多智能体的设计专业学生项目/论文评审系统，集合 5 个 AI 评委从不同专业角度进行并行评审。

## 🎯 系统概述

该系统模拟真实的学术评审场景：
- **5位问题制造者** - 从不同角度提出尖锐问题
- **1位问题收敛者** - 汇总反馈并提供改进建议
- **并行工作流** - 评委独立评审，互不干扰
- **结构化输出** - 最终生成优化方向报告

## 🤖 核心 Agent 角色

| Agent ID | 角色 | 专业视角 | 职能 |
|---------|------|---------|------|
| `designprof` | 设计学科评委 | UX/UI、美学、交互设计 | 从用户体验角度提问 |
| `mechprof` | 机械/工程评委 | 工程可行性、制造工艺 | 挑战实现的可行性 |
| `csprof` | 计算机/AI评委 | 算法、技术实现 | 质疑技术深度 |
| `xdisciplineprof` | 交叉学科评委 | 系统整体性、创新性 | 评估跨领域协同 |
| `trollprof` | 插科打诨评委 | 对抗性视角、边界情景 | 挑战假设和逻辑漏洞 |
| `metaassistant` | **综合分析助手** | **全局视图、问题收敛** | **汇总反馈提出建议** |

## 📋 工作流程

```
学生提交汇报材料
        ↓
    【第一阶段：多评委并行提问】
    5个评委独立工作，各自在 2-5 秒内给出问题
        ↓
    【metaassistant 综合分析】
    提供结构化的回答建议
        ↓
    【第二阶段：学生回答】
    逐点回答各评委问题
        ↓
    【第三阶段：评委追问与深化】
    可选的进一步质疑和澄清
        ↓
    【最终总结：meta_assistant 输出报告】
    改进建议 + 优化方向 + 优先级排序
```

## 📚 文档导引

### 快速上手
👉 **[QUICKSTART.md](QUICKSTART.md)** - 5分钟快速入门
- 系统启动方法
- 进行评审的步骤
- 常见问题排查

### 详细系统设计
👉 **[REVIEW_SYSTEM.md](REVIEW_SYSTEM.md)** - 完整系统文档
- 6个Agent角色详解
- 工作流程可视化
- 性能参数调优
- 使用场景示例

### 技术协议
👉 **[AGENT_PROTOCOL.md](AGENT_PROTOCOL.md)** - Agent通信协议
- 消息通信模型
- Agent状态机
- 并发协调机制
- 故障处理

## 🚀 快速开始

### 1️⃣ 启动系统

```bash
# 启动网络和所有agent
openagents network start MockPanel
openagents agent start designprof.yaml
openagents agent start csprof.yaml
openagents agent start mechprof.yaml
openagents agent start trollprof.yaml
openagents agent start xdisciplineprof.yaml
openagents agent start metaassistant.yaml
```

系统将在 `http://localhost:8700/studio` 运行

### 2️⃣ 进入评审频道

1. 打开 Studio UI
2. 加入 `review_panel` 频道
3. 确认所有 6 个 Agent 在线

### 3️⃣ 学生提交汇报

在频道中输入项目描述（30-100字），例如：
```
我的项目是基于ResNet的物体识别系统，
能识别1000种物体，准确率98.5%。
已完成原型，待优化性能。
```

### 4️⃣ 等待评委反馈

约 5-10 秒内：
- ✅ 5位评委各自给出评价和问题
- ✅ meta_assistant 提供综合建议
- ✅ 学生逐点回答

### 5️⃣ 获得最终报告

评审结束时，metaassistant 输出：
- 关键问题梳理
- 改进方向优先级
- 具体优化建议

## 📁 项目结构

```
MockPanel/
├── agents/                    # Agent配置文件
│   ├── designprof.yaml      # 设计评委
│   ├── mechprof.yaml        # 工程评委
│   ├── csprof.yaml          # CS/AI评委
│   ├── xdisciplineprof.yaml # 交叉学科评委
│   ├── trollprof.yaml       # 插科打诨评委
│   ├── metaassistant.yaml   # 综合分析助手
│   ├── assistant.yaml        # 基础助手
│   └── critic.yaml           # 评论者
├── network.yaml              # 网络和agent注册配置
├── config/                   # 全局配置
├── logs/                     # 系统日志
├── README.md                 # 本文件
├── QUICKSTART.md             # 快速开始指南
├── REVIEW_SYSTEM.md          # 系统详细文档
└── AGENT_PROTOCOL.md         # 通信协议文档
```

## ⚙️ 配置参数

### 响应延迟（模拟自然讨论节奏）
```yaml
reaction_delay: "random(2, 4)"  # 单位：秒
```

### 网络参数（network.yaml）
| 参数 | 默认值 | 用途 |
|------|--------|------|
| agent_timeout | 180 | Agent最大处理时间 |
| message_timeout | 30 | 消息等待超时 |
| message_queue_size | 1000 | 消息队列容量 |
| max_connections | 20 | 最大连接数 |

## 🔧 自定义和扩展

### 修改评委的评价风格
编辑 `agents/[agent_name].yaml` 中的 `instruction` 字段

### 添加新的专业评委
1. 创建 `agents/new_prof.yaml`
2. 参考现有评委模板配置
3. 在 `network.yaml` 中注册新agent

### 调整系统参数
- 响应速度：修改 `reaction_delay`
- 消息长度：修改 instruction 中的字数限制
- 网络容量：修改 `network.yaml` 的对应参数

## 📊 系统特性

✅ **并行工作** - 5位评委同时独立工作，无阻塞  
✅ **自然节奏** - 随机延迟模拟真实讨论流程  
✅ **智能综合** - Meta_assistant 识别关键问题和矛盾点  
✅ **结构化输出** - 最终报告包含优先级排序  
✅ **易于定制** - YAML配置，修改指令词立即生效  
✅ **完整文档** - 从快速入门到协议细节的全方位指南  

## 🎓 典型应用场景

### 学生论文评审
研究生学位论文答辩前的多角度评审

### 产品方案评审
新产品概念设计的多学科评价

### 创业项目投融资
Pitch演讲前的模拟评委反馈

### 研究论文审稿
学术论文多维度审查

## 🐛 故障排查

### Agent 不回应
- 检查 `logs/agents/` 中的日志
- 确认agent是否连接到admin组
- 查看网络配置的agent注册

### 消息延迟过长
- 检查 `agent_timeout` 设置
- 查看系统负载
- 考虑增加 `message_queue_size`

### 频道消息混乱
- 清空旧消息（可选）
- 确认 `max_thread_depth` 配置合理
- 重启Studio UI

详见 **[QUICKSTART.md](QUICKSTART.md) - 4. 监控和调试**

## 📖 更多资源

- **OpenAgents 官方文档** - https://openagents.ai
- **gRPC 通信** - Port 8600
- **HTTP/Studio UI** - Port 8700

## 📝 更新日志

### v1.0 (2026-01-07)
- ✅ 完整的5-agent评审系统
- ✅ 并行评委工作流
- ✅ 综合分析和改进建议
- ✅ 完整文档和快速开始指南
- ✅ 详细的通信协议

## 👥 系统要求

- OpenAgents 运行环境
- Python 3.8+
- 网络连接 (localhost:8700, 8600)
- 足够的内存运行6个Agent
