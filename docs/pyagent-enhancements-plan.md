# nanobot 核心功能增强最终方案

为 nanobot 补充 pyagent 的核心功能和工具，遵循最小侵入原则，便于合并 upstream。

## 🔍 调查结果总结

### 1. 工具对比分析

**三个项目的工具对比：**

| 工具 | opencode | pyagent | nanobot | 结论 |
|------|----------|---------|---------|------|
| **read/view** | ✅ view | ✅ read | ✅ read_file | 都有 |
| **write** | ✅ write | ✅ write | ✅ write_file | 都有 |
| **edit** | ✅ edit | ✅ edit | ✅ edit_file | 都有 |
| **ls** | ✅ ls | ✅ ls | ✅ list_dir | 都有 |
| **bash/exec** | ✅ bash | ✅ bash | ✅ exec | 都有 |
| **grep** | ✅ grep | ✅ grep | ❌ | **nanobot 缺少** |
| **glob** | ✅ glob | ✅ glob | ❌ | **nanobot 缺少** |
| **patch** | ✅ patch | ✅ patch | ❌ | **nanobot 缺少** |
| **git** | ❌ | ✅ git/status/diff/log | ❌ | **pyagent 独有** |
| **webfetch** | ✅ fetch | ✅ webfetch | ✅ web_fetch | 都有 |
| **diagnostics** | ✅ | ❌ | ❌ | opencode 独有 |
| **sourcegraph** | ✅ | ❌ | ❌ | opencode 独有 |

**需要补充的工具（按优先级）：**
1. **Git 工具集** - pyagent 独有，编程助手必备
2. **PatchTool** - opencode 和 pyagent 都有，nanobot 缺少
3. **GrepTool** - opencode 和 pyagent 都有，nanobot 缺少
4. **GlobTool** - opencode 和 pyagent 都有，nanobot 缺少

---

### 2. 会话压缩方案调查

**opencode 的实现：**
- 使用 `autoCompact` 配置（默认 true）
- 当 token 使用达到 95% 上下文窗口时自动触发
- 调用 LLM 生成摘要
- 创建新会话并保存摘要
- **实现方式**：在 `agent.go` 的 `Summarize()` 方法中
- **没有独立的压缩工具包**，是集成在 agent 中的功能

**nanobot 的实现：**
- 已有 `MemoryConsolidator` 类
- 基于 token 阈值（context_window // 2）自动触发
- 使用 LLM 生成摘要并保存到 MEMORY.md 和 HISTORY.md
- 支持渐进式整合（最多 5 轮）

**pyagent 的实现：**
- 完整的压缩策略系统（4 种策略）
- `CompressionConfig` 配置类
- 支持手动/自动/混合触发模式

**结论：**
- **没有成熟的独立会话压缩工具包**
- 三个项目都是自己实现的
- nanobot 已有基础实现，可以增强但不需要完全替换
- 建议：添加轻量级策略（RECENT、SLIDING_WINDOW）作为可选项

---

### 3. 新增文件位置规划

基于 nanobot 的目录结构，新增文件应放在：

```
nanobot/
├── agent/
│   └── tools/
│       ├── git.py          # 新增：Git 工具集
│       ├── patch.py        # 新增：Patch 工具
│       ├── grep.py         # 新增：Grep 工具
│       ├── glob.py         # 新增：Glob 工具
│       └── compression.py  # 新增：压缩策略（可选）
```

**原则：**
- 所有新工具放在 `nanobot/agent/tools/` 目录
- 遵循现有的文件命名规范
- 不修改现有文件结构

---

## 🎯 最终实施方案

### 阶段一：Git 工具集（必需，3-4 天）

**新增文件：**
- `nanobot/agent/tools/git.py`

**包含工具：**
1. `GitTool` - 通用 Git 命令执行
2. `GitStatusTool` - Git 状态查询
3. `GitDiffTool` - Git 差异比较
4. `GitLogTool` - Git 提交历史

**实施细节：**
- 从 `pyagent/src/pyagent/tools/git.py` 移植
- 适配 nanobot 的 `Tool` 基类（不是 `BaseTool`）
- 工具接口：`name`, `description`, `parameters`, `execute()`
- 在 `AgentLoop._register_default_tools()` 中注册

**修改的现有文件：**
- `nanobot/agent/loop.py`：在 `_register_default_tools()` 中添加注册代码（约 5 行）

---

### 阶段二：Patch/Grep/Glob 工具（推荐，4-5 天）

**新增文件：**
- `nanobot/agent/tools/patch.py`
- `nanobot/agent/tools/grep.py`
- `nanobot/agent/tools/glob.py`

**实施细节：**
- 从 pyagent 对应文件移植
- 适配 nanobot 工具系统
- 在 `AgentLoop._register_default_tools()` 中注册

**修改的现有文件：**
- `nanobot/agent/loop.py`：添加注册代码（约 3 行）

---

### 阶段三：压缩策略增强（可选，3-4 天）

**新增文件：**
- `nanobot/agent/compression.py`（可选）

**实施方式：**
- 在 `nanobot/agent/memory.py` 中添加策略枚举和配置类
- 实现 RECENT 和 SLIDING_WINDOW 策略
- 保持现有 SUMMARY 策略作为默认
- 在 `nanobot/config/schema.py` 中添加配置项

**修改的现有文件：**
- `nanobot/agent/memory.py`：添加策略类（约 100 行）
- `nanobot/agent/loop.py`：传递配置（约 5 行）
- `nanobot/config/schema.py`：添加配置项（约 20 行）

---

## 📁 文件变更清单

### 新增文件（最小侵入）

```
nanobot/agent/tools/
├── git.py          # 新增 ~400 行
├── patch.py        # 新增 ~200 行
├── grep.py         # 新增 ~150 行
└── glob.py         # 新增 ~100 行
```

### 修改现有文件（最小化修改）

**必需修改：**
```python
# nanobot/agent/loop.py（仅在 _register_default_tools 方法中添加）
def _register_default_tools(self) -> None:
    # ... 现有代码 ...
    
    # 新增：Git 工具集
    from nanobot.agent.tools.git import GitTool, GitStatusTool, GitDiffTool, GitLogTool
    self.tools.register(GitTool(workspace=self.workspace))
    self.tools.register(GitStatusTool(workspace=self.workspace))
    self.tools.register(GitDiffTool(workspace=self.workspace))
    self.tools.register(GitLogTool(workspace=self.workspace))
    
    # 新增：其他工具
    from nanobot.agent.tools.patch import PatchTool
    from nanobot.agent.tools.grep import GrepTool
    from nanobot.agent.tools.glob import GlobTool
    self.tools.register(PatchTool(workspace=self.workspace))
    self.tools.register(GrepTool(workspace=self.workspace))
    self.tools.register(GlobTool(workspace=self.workspace))
```

**可选修改（压缩策略）：**
```python
# nanobot/agent/memory.py（添加策略枚举）
from enum import Enum

class CompressionStrategy(Enum):
    SUMMARY = "summary"           # 现有的 LLM 摘要
    RECENT = "recent"             # 新增：保留最近消息
    SLIDING_WINDOW = "sliding_window"  # 新增：滑动窗口

# 在 MemoryConsolidator 中添加策略选择逻辑
```

---

## ✅ 最小侵入原则

### 1. 不修改现有工具
- 所有现有工具保持不变
- 不修改 `filesystem.py`, `shell.py`, `web.py` 等

### 2. 最小化现有文件修改
- 只在 `loop.py` 的 `_register_default_tools()` 中添加注册代码
- 不修改其他方法或逻辑
- 修改行数 < 20 行

### 3. 新功能独立
- 所有新工具都是独立文件
- 可以单独测试
- 不影响现有功能

### 4. 向后兼容
- 所有新工具都是可选的
- 不改变现有 API
- 配置保持兼容

---

## 🔄 合并 upstream 策略

### Git 分支策略
```bash
# 当前分支
feature/pyagent-enhancements

# 合并 upstream 时
git fetch upstream
git rebase upstream/main

# 如果有冲突，只可能在：
# - nanobot/agent/loop.py（_register_default_tools 方法）
# - nanobot/agent/memory.py（如果添加了压缩策略）
```

### 冲突解决
- 新增文件不会有冲突
- `loop.py` 的修改在独立的方法中，冲突概率低
- 即使有冲突，也只是工具注册代码，容易解决

---

## 📊 工作量估算

| 任务 | 工作量 | 文件数 | 代码行数 |
|------|--------|--------|----------|
| Git 工具集 | 3-4 天 | 1 新增 + 1 修改 | ~400 新增 + ~10 修改 |
| Patch/Grep/Glob | 4-5 天 | 3 新增 + 1 修改 | ~450 新增 + ~5 修改 |
| 压缩策略增强 | 3-4 天 | 0-1 新增 + 2-3 修改 | ~200 新增 + ~30 修改 |
| **总计** | **10-13 天** | **4-5 新增 + 2-4 修改** | **~1050 新增 + ~45 修改** |

---

## 🎯 推荐实施顺序

### 方案 A（完整实施）
1. Git 工具集（必需）
2. Patch/Grep/Glob 工具（推荐）
3. 压缩策略增强（可选）

**优点：**
- 功能最完整
- 与 pyagent 和 opencode 对齐

**缺点：**
- 工作量最大（10-13 天）

---

### 方案 B（核心优先）
1. Git 工具集（必需）
2. PatchTool（必需）
3. GrepTool（推荐）

**优点：**
- 覆盖最核心的功能
- 工作量适中（6-8 天）

**缺点：**
- 缺少 GlobTool 和压缩增强

---

### 方案 C（最小可行）
1. Git 工具集（必需）
2. PatchTool（必需）

**优点：**
- 工作量最小（5-6 天）
- 补充最关键的功能

**缺点：**
- 缺少 Grep/Glob 和压缩增强

---

## 🔧 技术实施细节

### 工具适配模板

**nanobot 的 Tool 基类：**
```python
class Tool(ABC):
    name: str
    description: str
    parameters: dict
    
    @abstractmethod
    def execute(self, **kwargs) -> Any:
        pass
```

**pyagent 的 BaseTool：**
```python
class BaseTool(ABC):
    name: ClassVar[str]
    description: ClassVar[str]
    parameters: ClassVar[Optional[type[ToolParams]]] = None
    
    @abstractmethod
    def execute(self, **kwargs) -> ToolResult:
        pass
```

**适配方式：**
- 将 `ClassVar` 改为实例属性
- 将 `ToolParams` 转换为 dict
- 保持 `execute()` 方法签名

---

## 📝 配置示例

### Git 工具配置（无需额外配置）
```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5"
    }
  }
}
```

### 压缩策略配置（可选）
```json
{
  "agents": {
    "compression": {
      "strategy": "summary",
      "threshold": 0.5,
      "keep_recent": 10
    }
  }
}
```

---

## 🧪 测试策略

### 单元测试
- 每个新工具都需要单元测试
- 放在 `tests/test_tools/` 目录
- 测试文件命名：`test_git.py`, `test_patch.py` 等

### 集成测试
- 测试工具注册
- 测试工具调用
- 测试错误处理

### 手动测试
- 在实际项目中测试 Git 操作
- 测试 Patch 应用
- 测试 Grep/Glob 搜索

---

## 🚀 实施步骤

### 步骤 1：创建分支（已完成）
```bash
git checkout -b feature/pyagent-enhancements
```

### 步骤 2：实施 Git 工具集
1. 创建 `nanobot/agent/tools/git.py`
2. 从 pyagent 移植代码
3. 适配 nanobot 工具基类
4. 在 `loop.py` 中注册
5. 添加测试
6. 提交：`git commit -m "feat: add Git tools (status/diff/log)"`

### 步骤 3：实施其他工具
1. 创建 `patch.py`, `grep.py`, `glob.py`
2. 移植和适配
3. 注册工具
4. 添加测试
5. 提交：`git commit -m "feat: add patch/grep/glob tools"`

### 步骤 4：压缩策略增强（可选）
1. 修改 `memory.py`
2. 添加策略枚举和实现
3. 添加配置支持
4. 添加测试
5. 提交：`git commit -m "feat: add compression strategies"`

### 步骤 5：文档更新
1. 更新 README.md
2. 添加工具使用示例
3. 提交：`git commit -m "docs: update README with new tools"`

---

## 📌 注意事项

### 1. 代码风格
- 遵循 nanobot 的代码风格
- 使用 ruff 格式化
- 添加类型提示

### 2. 错误处理
- 所有工具都要有错误处理
- 返回清晰的错误消息
- 不要让工具崩溃

### 3. 安全性
- Git 操作要验证路径
- Patch 应用要检查权限
- 防止路径遍历攻击

### 4. 性能
- Grep/Glob 要限制搜索范围
- 避免扫描大型目录
- 添加超时机制

---

## 🎯 成功标准

### 功能完整性
- [ ] Git 工具可以执行基本操作
- [ ] Patch 工具可以应用补丁
- [ ] Grep 工具可以搜索文件
- [ ] Glob 工具可以匹配文件

### 代码质量
- [ ] 所有新代码有单元测试
- [ ] 测试覆盖率 > 80%
- [ ] 通过 ruff 检查
- [ ] 通过类型检查

### 兼容性
- [ ] 不影响现有功能
- [ ] 向后兼容
- [ ] 可以合并 upstream

### 文档
- [ ] README 更新
- [ ] 工具使用示例
- [ ] 配置说明

---

## 🔍 待决策项

请确认以下问题：

1. **实施范围**：选择方案 A/B/C？
   - 方案 A：完整实施（10-13 天）
   - 方案 B：核心优先（6-8 天）
   - 方案 C：最小可行（5-6 天）

2. **压缩策略**：是否实施？
   - 是：添加轻量级策略
   - 否：保持现有实现

3. **优先级调整**：是否有其他优先级考虑？

---

## 📦 交付物

### 代码
- [ ] 新增工具文件（4-5 个）
- [ ] 修改现有文件（2-4 个）
- [ ] 单元测试文件

### 文档
- [ ] README 更新
- [ ] 工具使用指南
- [ ] 配置说明

### 测试
- [ ] 单元测试
- [ ] 集成测试
- [ ] 手动测试报告

---

**当前状态：**
- ✅ 分支已创建：`feature/pyagent-enhancements`
- ✅ 调查已完成
- ✅ 方案已保存到 docs
- ⏳ 等待方案确认和实施
