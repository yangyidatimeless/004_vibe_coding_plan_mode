### 发言 11 - 允灿
**时间**: 2026-03-22 21:00

**我的见解**：

感谢少平舅舅的详细交互设计方案！我来回应你关于 API 接口定义的 3 个问题：

## 🔧 API 接口定义回应

### 问题 1：Plan 生成接口的响应格式

**我的建议：采用"结构化数据 + Markdown 双格式"方案**

**理由**：
- 结构化数据便于前端解析和展示（步骤列表、状态管理等）
- Markdown 格式便于用户阅读和复制（可以导出为文档）
- 两者可以互相转换，不冲突

**响应格式示例**：
```json
{
  "plan_id": "plan_20260322_001",
  "title": "用户认证功能开发",
  "description": "创建用户模型、认证服务和 API 端点",
  "steps": [
    {
      "step_id": 1,
      "title": "创建用户模型",
      "description": "在 models/ 目录下创建 User.py 文件",
      "type": "create_file",
      "path": "models/user.py",
      "details": {
        "content_preview": "class User(Base):...",
        "estimated_time": "2s"
      },
      "status": "pending",
      "risk_level": "low"
    },
    {
      "step_id": 2,
      "title": "创建认证服务",
      "description": "在 services/ 目录下创建 auth.py 文件",
      "type": "create_file",
      "path": "services/auth.py",
      "details": {
        "content_preview": "def login(user, password):...",
        "estimated_time": "3s"
      },
      "status": "pending",
      "risk_level": "low"
    }
  ],
  "markdown_view": "### 执行计划\n\n1. **创建用户模型** - 在 models/ 目录下创建 User.py 文件\n2. **创建认证服务** - 在 services/ 目录下创建 auth.py 文件\n...",
  "created_at": "2026-03-22T21:00:00+08:00",
  "estimated_total_time": "15s"
}
```

**API 端点设计**：
```
POST /api/v1/plans/generate
Request: { "user_request": "帮我创建一个用户认证功能" }
Response: 上述结构化数据

GET /api/v1/plans/{plan_id}/markdown
Response: { "markdown": "### 执行计划\n..." }
```

**前端使用建议**：
- 用 `steps` 数组渲染步骤列表（支持展开/收起）
- 用 `markdown_view` 提供给用户复制/导出
- 用 `risk_level` 决定是否需要额外确认（高风险步骤）

---

### 问题 2：步骤执行状态的更新频率

**我的建议：采用"WebSocket 推送 + 轮询降级"方案**

**Phase 1（MVP）- 轮询方案**：
- **轮询间隔**：`1000ms`（1 秒）
- **理由**：
  - 实现简单，可靠性高
  - 1 秒延迟对于文件操作级别的任务是可接受的
  - 不会给服务器造成太大压力（假设并发用户不多）

**轮询 API 设计**：
```
GET /api/v1/plans/{plan_id}/status
Response: {
  "plan_id": "plan_001",
  "overall_status": "executing",  // pending, executing, completed, failed, cancelled
  "current_step_id": 3,
  "steps": [
    { "step_id": 1, "status": "completed", "result": "success", "duration_ms": 2300 },
    { "step_id": 2, "status": "completed", "result": "success", "duration_ms": 1800 },
    { "step_id": 3, "status": "executing", "progress": 60, "current_action": "正在创建路由处理器" },
    { "step_id": 4, "status": "pending" },
    { "step_id": 5, "status": "pending" }
  ],
  "last_updated": "2026-03-22T21:05:32+08:00"
}
```

**Phase 2 - WebSocket 升级方案**：
- 建立 WebSocket 连接后，服务端主动推送状态变更
- 延迟可降低到 `100-200ms`
- 适合需要精细进度反馈的场景（如大文件处理、长时间运行任务）

**前端实现建议**：
```javascript
// Phase 1: 轮询
const pollInterval = setInterval(async () => {
  const status = await fetchStatus(planId);
  updateUI(status);
  if (status.overall_status === 'completed' || status.overall_status === 'failed') {
    clearInterval(pollInterval);
  }
}, 1000);

// Phase 2: WebSocket
const ws = new WebSocket(`ws://api/plans/${planId}/ws`);
ws.onmessage = (event) => {
  const status = JSON.parse(event.data);
  updateUI(status);
};
```

---

### 问题 3：文件 diff 接口

**我的建议：提供独立的 diff 接口，支持多种格式**

**API 设计**：
```
GET /api/v1/plans/{plan_id}/steps/{step_id}/diff
Query params: 
  - format: "unified" | "json" | "html" (默认 unified)
  - context_lines: 3 (上下文行数，默认 3 行)

Response (format=unified):
{
  "step_id": 3,
  "path": "routes/auth.py",
  "diff_type": "modify",  // create, modify, delete
  "diff": "@@ -1,10 +1,15 @@\n from fastapi import APIRouter\n+from .auth import login\n ..."
}

Response (format=json):
{
  "step_id": 3,
  "path": "routes/auth.py",
  "diff_type": "modify",
  "hunks": [
    {
      "old_start": 1,
      "old_lines": 10,
      "new_start": 1,
      "new_lines": 15,
      "changes": [
        { "type": "unchanged", "line": "from fastapi import APIRouter" },
        { "type": "add", "line": "from .auth import login" },
        { "type": "unchanged", "line": "..." }
      ]
    }
  ]
}
```

**使用场景**：
- **前端展开视图**：用 `format=json` 解析后高亮展示（绿色新增、红色删除）
- **用户预览**：用 `format=unified` 展示传统 diff 格式
- **HTML 预览**：用 `format=html` 直接渲染（服务端生成带样式的 HTML）

**实现方案**：
- 用 Python 的 `difflib` 库生成 diff
- 对于创建新文件：diff 为完整文件内容（标记为全部新增）
- 对于修改文件：对比原文件和新内容生成 diff
- 对于删除文件：diff 为完整原内容（标记为全部删除）

---

## 📋 API 接口完整列表（Phase 1）

基于以上讨论，我整理 Phase 1 的完整 API 接口：

| 方法 | 端点 | 说明 | 响应时间 |
|------|------|------|----------|
| POST | `/api/v1/plans/generate` | 生成 Plan | < 3s |
| GET | `/api/v1/plans/{plan_id}` | 获取 Plan 详情 | < 100ms |
| GET | `/api/v1/plans/{plan_id}/markdown` | 获取 Markdown 格式 | < 100ms |
| POST | `/api/v1/plans/{plan_id}/approve` | 批准执行 Plan | < 100ms |
| POST | `/api/v1/plans/{plan_id}/cancel` | 取消 Plan | < 100ms |
| GET | `/api/v1/plans/{plan_id}/status` | 获取执行状态（轮询） | < 100ms |
| GET | `/api/v1/plans/{plan_id}/steps/{step_id}/diff` | 获取文件 diff | < 200ms |
| POST | `/api/v1/plans/{plan_id}/retry` | 重试失败的步骤 | < 100ms |

**OpenAPI Spec**：
- 我会在 2026-03-25 前提供完整的 OpenAPI 3.0 规范文档
- 少平舅舅可以直接导入 Swagger UI 或 Postman 查看接口详情
- 支持自动生成前端 TypeScript 类型定义（用 openapi-typescript）

---

## 回应少平舅舅的交互设计

**关于渐进式披露设计**：
- L1 简化视图：用 `steps[].title` + `steps[].description`
- L2 详细视图：用 `steps[].details` + diff 接口
- L3 技术视图：用 `steps[].details` 中的完整技术信息

**关于优先级**：
- 同意易达的 P0-P3 划分
- 我会优先实现 P0 接口的核心功能
- P1-P2 功能根据时间情况逐步完善

---

## 下一步行动

我会在 **2026-03-25** 前完成：
1. ✅ 服务端技术选型文档（已完成）
2. ⏳ API 接口定义（OpenAPI Spec）- 包含上述所有接口
3. ⏳ 核心逻辑原型代码（Plan 生成 + 步骤执行）
4. ⏳ 基础单元测试框架搭建

**交付物**：
- OpenAPI 规范文档（YAML 格式）
- 可运行的 FastAPI 原型（支持 Plan 生成和状态查询）
- 核心逻辑的单元测试（覆盖率 > 80%）

---

## 讨论结束确认

我对目前议题已无其他补充，已回应少平舅舅的所有问题。我对其他人的发言也无任何质疑。满足讨论结束条件 1。

**【允灿】：我已无其他补充，结束讨论。**

---
