[根目录](../CLAUDE.md) > **scripts**

# Scripts 核心模块

> 模块职责：实现 NotebookLM Skill 的所有核心功能，包括认证、查询、笔记本管理和浏览器自动化

## 模块职责

该模块是整个技能的核心，包含所有业务逻辑和自动化脚本。采用模块化设计，每个脚本职责明确，通过 `run.py` 统一管理环境。

## 入口与启动

### run.py - 环境包装器
- **功能**: 确保所有脚本在正确的虚拟环境中运行
- **职责**:
  - 自动创建 `.venv`（如果不存在）
  - 安装依赖（首次运行）
  - 激活环境并执行目标脚本
- **使用**: 所有命令必须通过此包装器执行
  ```bash
  python scripts/run.py [script_name] [args...]
  ```

### setup_environment.py - 环境初始化
- **功能**: 创建和配置 Python 虚拟环境
- **触发**: `run.py` 自动检测并调用
- **安装内容**:
  - Python 虚拟环境
  - patchright 浏览器自动化库
  - Python-dotenv 环境管理

## 对外接口

### ask_question.py - 查询接口
- **功能**: 向 NotebookLM 发送查询并获取答案
- **特点**:
  - 实现混合认证方案
  - 智能等待思考消息完成
  - 自动添加后续提问提示
  - 120秒超时（v1.3.0更新）
- **关键方法**:
  - `ask_notebooklm()`: 主要查询逻辑
  - 等待思考消息消失 (`div.thinking-message`)
  - 稳定性检测（3次连续检查）
- **使用示例**:
  ```bash
  python scripts/run.py ask_question.py --question "What hooks are available?"
  python scripts/run.py ask_question.py --question "..." --notebook-id react-docs
  ```

### notebook_manager.py - 笔记本管理
- **功能**: CRUD 操作管理笔记本库
- **类**: `NotebookLibrary`
- **核心方法**:
  - `add_notebook()`: 添加新笔记本（需要完整元数据）
  - `list_notebooks()`: 列出所有笔记本
  - `search()`: 按主题搜索
  - `activate()`: 设置活动笔记本
- **数据存储**: `data/library.json`
- **智能添加**: 通过查询发现笔记本内容后再添加

## 关键依赖与配置

### config.py - 配置中心
```python
# 关键配置
QUERY_INPUT_SELECTORS = [...]    # NotebookLM 输入框选择器
RESPONSE_SELECTORS = [...]       # 答案区域选择器
QUERY_TIMEOUT_SECONDS = 120      # 查询超时（v1.3.0更新）
BROWSER_ARGS = [...]            # Chrome 启动参数
```

### browser_utils.py - 浏览器工具集
- **BrowserFactory**: 创建配置好的浏览器上下文
  - 使用真实 Chrome（非 Chromium）
  - 反检测配置
  - Cookie 注入（解决 Playwright bug #36139）
- **StealthUtils**: 人类化交互
  - 随机 typing 速度 (160-240 WPM)
  - 自然延迟
  - 鼠标移动

### 认证流程 (auth_manager.py)
- **混合认证方案**:
  1. 持久化浏览器配置文件（指纹一致性）
  2. 手动 Cookie 注入（会话 Cookie 持久化）
- **认证状态检查**:
  - 检查 `state.json` 存在
  - 验证状态文件时效（7天内）
- **交互式设置**: 可视化浏览器供手动登录

## 数据模型

### 认证数据结构
```json
// data/auth_info.json
{
  "authenticated": true,
  "account_email": "user@gmail.com",
  "login_time": "2025-12-07T...",
  "browser_state_valid": true
}
```

### 笔记本数据结构
```json
// data/library.json
{
  "notebooks": {
    "react-docs": {
      "id": "react-docs",
      "url": "https://notebooklm.google.com/notebook/...",
      "name": "React Documentation",
      "description": "Official React docs",
      "topics": ["react", "frontend", "hooks"],
      "content_types": ["documentation"],
      "use_cases": ["API reference", "best practices"],
      "added_at": "2025-12-07T..."
    }
  },
  "active_notebook_id": "react-docs",
  "updated_at": "2025-12-07T..."
}
```

## 测试与质量

### 当前测试覆盖
- ❌ 无自动化测试
- ✅ 手动端到端测试（通过实际使用验证）
- ✅ 错误处理覆盖（异常捕获和用户友好提示）

### 代码质量保证
1. **错误处理**: 所有脚本都有完善的 try-catch 块
2. **类型注解**: Python 3.8+ 风格的类型提示
3. **文档字符串**: 每个函数都有详细说明
4. **配置分离**: 所有硬编码值移至 `config.py`

### 建议的测试增强
```python
# 建议添加的测试文件
tests/
├── test_auth_manager.py      # 认证逻辑测试
├── test_notebook_manager.py  # CRUD 操作测试
├── test_browser_utils.py     # 浏览器工具测试
├── test_config.py           # 配置验证测试
└── conftest.py              # 测试配置和 Fixtures
```

## 常见问题 (FAQ)

### Q: 为什么必须使用 run.py？
A: `run.py` 确保：
- 虚拟环境存在（自动创建）
- 依赖已安装（自动安装）
- 正确的 Python 解释器被使用

### Q: Chrome vs Chromium 的区别？
A:
- **Chrome**: 真实浏览器，更好的反检测，跨平台一致性
- **Chromium**: 开源版本，更容易被检测

### Q: 什么是混合认证？
A: 结合两种方法：
1. 持久化浏览器配置文件（保持指纹）
2. 手动 Cookie 注入（解决 Playwright bug）

### Q: 为什么每个查询都是新浏览器？
A: Skill 架构限制 - 每次调用独立
- 优点：状态隔离，不会泄露
- 缺点：性能开销
- 解决方案：后续提问机制

## 相关文件清单

### 核心脚本
- `run.py` - 环境包装器（入口点）
- `ask_question.py` - 查询主逻辑
- `auth_manager.py` - Google 认证
- `notebook_manager.py` - 笔记本库管理
- `browser_session.py` - 浏览器会话管理

### 工具模块
- `browser_utils.py` - 浏览器工厂和隐身工具
- `config.py` - 配置中心
- `cleanup_manager.py` - 数据清理工具
- `setup_environment.py` - 环境初始化

### 数据文件
- `../data/library.json` - 笔记本元数据
- `../data/auth_info.json` - 认证信息
- `../data/browser_state/` - 浏览器状态目录

## 变更记录 (Changelog)

### 2025-12-07 13:12:46
- ✨ 创建 scripts 模块文档
- 📊 分析各脚本的职责和关系
- 🔗 建立内部导航链接
- 📝 记录架构决策和最佳实践

### v1.3.0 (2025-11-21) 关键更新
- **模块化重构**: 拆分 `config.py` 和 `browser_utils.py`
- **超时提升**: 30s → 120s
- **思考消息检测**: 修复不完整答案问题
- **稳定性改进**: 3次连续检查确保答案完整