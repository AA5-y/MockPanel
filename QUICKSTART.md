# 快速开始指南 - 多智能体评审系统

## 1. 启动系统

### 前提条件
- OpenAgents 运行环境已安装
- 网络配置正确（localhost:8700）

### 启动步骤

```bash
# 方式1：启动网络（自动加载所有agent）
openagents-studio network start network.yaml

# 方式2：通过Python启动
python -m openagents.network.launcher network.yaml
```

系统启动后：
- Studio Web UI: http://localhost:8700/studio
- Agent通讯: localhost:8600 (gRPC)

---

## 2. 进行学生评审

### 步骤 A: 进入评审频道
1. 打开 http://localhost:8700/studio
2. 连接到 `review_panel` 频道
3. 确保看到 6 个 Agent 在线：
  - ✅ designprof (设计评委)
  - ✅ mechprof (工程评委)
  - ✅ csprof (CS/AI评委)
  - ✅ xdisciplineprof (交叉学科评委)
  - ✅ trollprof (插科打诨评委)
  - ✅ metaassistant (综合分析助手)

### 步骤 B: 学生提交汇报（文本或语音）
```
示例输入：
"我的项目是一个智能家居控制系统，基于Python和MQTT协议。
支持语音控制、自动化场景、能耗优化。
已完成原型，待优化。"
```

### 步骤 C: 等待多评委并行反馈
- 各评委会在 2-5 秒内各自给出评价和问题
- 评论会在频道中顺序显示
- **注意**: Agent 之间不会相互回应，只对人工输入作反应

### 步骤 D: 查看 Meta Assistant 的建议
等 5 位评委都给出问题后，metaassistant 会汇总：
- 关键问题梳理
- 改进方向排序
- 回答建议

### 步骤 E: 学生逐点回答
在频道中依次回答各评委的问题

### 步骤 F: 评委追问和深化
可选：评委基于学生答复进一步追问

### 步骤 G: Meta Assistant 最终总结
metaassistant 给出最终的改进建议和优化方向报告

---

## 3. 自定义Agent行为

### 修改评委的评价风格

编辑相应的 YAML 文件的 `instruction` 字段：

**例如：修改 designprof 的审视重点**

编辑 `agents/designprof.yaml`:
```yaml
instruction: |
  You are DESIGN_PROF...
  
  YOUR FOCUS:
  - 改为关注色彩搭配和品牌一致性
  - 改为关注国际化和可访问性标准
  ...
```

### 调整响应延迟

影响各评委的回应速度和讨论节奏：

```yaml
reaction_delay: "random(X, Y)"  # 单位：秒

# 示例配置
# 快速响应：random(1, 2)
# 自然节奏：random(2, 4)  [当前默认]
# 深思熟虑：random(3, 6)
```

### 调整消息长度限制

```yaml
instruction: |
  # 改变这一行
  1. Keep responses under 150 words  # 默认值
  # 可改为：
  1. Keep responses under 200 words  # 更详细
  1. Keep responses under 100 words  # 更简洁
```

---

## 4. 监控和调试

### 查看Agent日志

```bash
# 查看最近的日志
tail -f logs/agents/*.log

# 查看特定agent的日志
tail -f logs/agents/design_prof.log
```

### 检查Agent连接状态

在 Studio UI 中：
1. 右侧 "Agents" 面板
2. 绿点 = 在线
3. 红点 = 离线（检查logs）

### 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|--------|
| Agent 不回应 | 未连接到admin组或频道 | 检查 network.yaml 中的 group 配置 |
| 消息堆积 | Agent超时或卡顿 | 查看日志，重启该agent |
| 频道找不到 | 频道未注册 | 检查 network.yaml 的 default_channels |

---

## 5. 高级配置

### 调整网络参数

编辑 `network.yaml`:

```yaml
network:
  agent_timeout: 180          # 增加agent处理时间
  message_timeout: 30         # 增加消息等待时间
  message_queue_size: 2000    # 增加消息队列
  max_connections: 30         # 支持更多并发
```

### 添加新的评委Agent

1. 创建新文件 `agents/new_prof.yaml`
2. 复制现有评委的模板
3. 修改 `agent_id` 和 `instruction`
4. 在 `network.yaml` 的 `agents` 部分注册

---

## 6. 推荐的工作流程

### 单个学生评审（15-30分钟）
1. 学生提交汇报 (1分钟)
2. 多评委提问 (3-5分钟)
3. Meta助手给建议 (1-2分钟)
4. 学生回答 (5-10分钟)
5. 评委追问 (2-5分钟)
6. 最终总结 (2-3分钟)

### 批量评审（多个学生）
- 顺序进行：Student 1 → Student 2 → ...
- 评委和助手在频道中保持上线
- 每个评审之间清空频道消息（可选）

---

## 7. 预期的系统输出

### Meta Assistant 最终报告示例

```
【评审总结】- Student Project X

📋 关键反馈归纳：
1. 设计层面：UI/UX 需要用户测试验证
2. 工程层面：成本分析和制造可行性还需深化
3. 技术层面：算法性能和扩展性有提升空间
4. 创新性：跨学科应用潜力大，但社会影响评估缺失
5. 鲁棒性：边界情况和异常处理覆盖不足

⭐ 优先级改进方向：
【高优】
- 补充对抗性场景测试（troll_prof 强调）
- 完成用户可用性测试（design_prof 强调）

【中优】
- 成本-效益分析（mech_prof 强调）
- 算法基准测试对标（cs_prof 强调）

【低优】
- 社会影响白皮书（xdiscipline_prof 强调）

🎯 下一步建议：
1. 制定详细的改进计划和时间表
2. 优先完成高优级项目
3. 2周后进行第二轮评审
```

---

## 8. 联系与支持

- **系统文档**: 见 `REVIEW_SYSTEM.md`
- **Agent配置**: 见 `agents/` 目录
- **网络配置**: 见 `network.yaml`
- **日志位置**: `logs/agents/`

