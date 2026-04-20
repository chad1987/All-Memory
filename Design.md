
# 一、设计目标

## 1. 核心目标

* OpenClaw 与 Hermes 的 `MEMORY.md`、`USER.md` **同源一致**
* 任一侧修改 → 另一侧可见（准实时）
* 避免冲突覆盖、重复写入、语义污染
* 支持结构化检索与长期演进

---

## 2. 设计原则

1. **单一事实源（SSOT）**

   * 文件不是源头，数据库才是源头

2. **文件仅为“投影（Projection）”**

   * MEMORY.md / USER.md = 渲染结果

3. **结构化优先**

   * Markdown只是展示层

4. **双向同步但单向裁决**

   * 写入统一走 Memory Hub

---

# 二、总体架构

```id="arch1"
        OpenClaw                  Hermes
     (workspace)              (memory system)
        │                          │
        │ 读写文件                 │
        ▼                          ▼
   MEMORY.md / USER.md（镜像层 / Projection）
                │
                │ 文件监听（Watcher）
                ▼
        ┌────────────────────┐
        │   Memory Sync Agent │  ← 核心组件
        └────────┬───────────┘
                 │
                 ▼
        ┌────────────────────┐
        │    Memory Hub       │
        │ (结构化存储中心)    │
        └────────────────────┘
```

---

# 三、核心组件设计

## 1. Memory Hub（唯一数据源）

### 数据结构（统一模型）

```json id="schema1"
{
  "id": "uuid",
  "type": "user_profile | memory | task | preference",
  "content": "用户正在减重计划",
  "source": "openclaw | hermes",
  "tags": ["健康", "减肥"],
  "importance": 0.8,
  "updated_at": "2026-04-20T10:00:00",
  "version": 3
}
```

---

## 2. 文件层（Projection）

### USER.md（用户画像）

结构标准化：

```markdown id="user_md"
# 用户画像

## 基本信息
- 身高：172cm
- 体重：74kg

## 长期目标
- 3个月减至70kg

## 偏好
- 偏向低碳饮食

## 约束
- 高血压、高血脂
```

---

### MEMORY.md（动态记忆）

```markdown id="memory_md"
# 长期记忆

## 重要事项
- 用户正在执行减重计划（2026-03）

## 历史行为
- 曾尝试断食但失败

## 当前任务
- 制定饮食+运动计划
```

---

# 四、同步机制设计（核心）

## 1. 同步模式

采用：

> **“事件驱动 + 文件监听 + API写入”三段式**

---

## 2. 写入路径（统一入口）

### 原则

所有写入必须经过 Memory Hub：

```id="flow1"
Agent → Memory API → DB → 渲染 → 写回文件
```

---

## 3. 文件监听（Watcher）

监听：

```id="watch_paths"
OpenClaw/workspace/*/MEMORY.md
OpenClaw/workspace/*/USER.md
Hermes/memory/*.md
```

工具建议：

* Python watchdog
* 或 Node.js chokidar

---

## 4. 同步流程

### 场景1：OpenClaw 写入 MEMORY.md

```id="flow2"
1. 文件变更触发 watcher
2. 解析 Markdown → 结构化数据
3. 调用 Memory Hub API（update）
4. Hub 更新版本号
5. 通知 Hermes（或其 watcher触发）
6. Hermes 文件重渲染
```

---

### 场景2：Hermes 写入 USER.md

流程同上（反向同步）

---

# 五、Markdown 解析与生成规则

## 1. 解析规则（MD → JSON）

| Markdown 区块 | 映射字段    |
| ----------- | ------- |
| 一级标题        | type    |
| 二级标题        | tags    |
| 列表项         | content |

---

示例：

```markdown id="parse_example"
## 偏好
- 低碳饮食
```

→

```json id="parse_json"
{
  "type": "preference",
  "content": "低碳饮食"
}
```

---

## 2. 渲染规则（JSON → MD）

按优先级输出：

```id="render_order"
用户画像 > 长期目标 > 偏好 > 历史行为
```

---

# 六、冲突解决机制（关键）

## 1. 冲突类型

| 类型   | 示例               |
| ---- | ---------------- |
| 覆盖冲突 | 两边修改同一条          |
| 语义冲突 | “低碳饮食” vs “高碳饮食” |
| 重复写入 | 相同内容多次记录         |

---

## 2. 解决策略

### （1）版本号机制

```json id="versioning"
{
  "version": 3
}
```

规则：

* 新版本覆盖旧版本
* 若版本冲突 → merge

---

### （2）语义去重（embedding）

* 相似度 > 0.9 → 判定重复

---

### （3）优先级策略

| 来源        | 优先级 |
| --------- | --- |
| USER.md   | 高   |
| MEMORY.md | 中   |
| 自动生成      | 低   |

---

# 七、关键实现细节

## 1. Memory Sync Agent（核心服务）

职责：

* 文件监听
* Markdown解析
* API调用
* 冲突处理
* 文件重写

---

## 2. 防循环写入机制（必须）

问题：

```id="loop_problem"
写入 → watcher触发 → 再写入 → 死循环
```

解决：

增加标记：

```json id="flag"
"source": "sync_agent"
```

或：

```id="lock"
写文件前加 lock 标志
```

---

## 3. 写入节流（Debounce）

* 500ms 内多次修改 → 合并一次

---

# 八、部署方案（本地）

## 组件

* Memory Hub（FastAPI）
* Chroma / SQLite
* Sync Agent（Python）
* Watcher

---

## 目录建议

```id="dir"
~/ai-memory/
 ├── hub/
 ├── sync-agent/
 ├── db/
 └── config.yaml
```

---

# 九、最小可运行版本（建议）

如果要快速落地：

### Step 1

只同步 USER.md（先解决核心）

### Step 2

再同步 MEMORY.md

### Step 3

再引入 embedding / 向量检索

---

# 十、风险与边界

## 1. 不建议做的事

* ❌ 直接 rsync 两个文件
* ❌ Git 同步（冲突不可控）
* ❌ 仅靠文本 diff

---

## 2. 潜在问题

* Agent 写入风格差异
* 记忆膨胀
* 低质量信息污染

---

# 十一、本质总结

这是一个典型的：

> **“多 Agent 共享状态一致性问题”**

对应经典解法：

* 数据中心化（Memory Hub）
* 视图外化（Markdown）
* 同步事件化（Watcher + API）

