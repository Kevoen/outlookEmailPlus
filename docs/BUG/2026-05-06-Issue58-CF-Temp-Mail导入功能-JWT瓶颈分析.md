# Issue #58 — Cloudflare Temp Mail 导入功能实现分析

**创建日期**: 2026-05-06
**关联 Issue**: https://github.com/ZeroPointSix/outlookEmailPlus/issues/58
**分析人员**: AI 代码分析助手
**状态**: 分析完成，用户已确认无需过度讨论 CF Worker JWT 获取细节，关注系统层 JWT 使用保证

---

## 概述

Issue #58 请求为 Cloudflare Temp Mail 添加导入功能。用户明确只需要**前端导入按钮 + 复用现有后端**，不关心 CF Worker 侧如何重新获取 JWT 的具体机制，只要求**系统层面确保导入后取件使用正确的 JWT**。

本文档在精简 CF Worker 行为讨论的前提下，聚焦「当前后端能力是否已足够」以及「前端按钮需要调用什么接口」。

---

## 当前后端能力评估

### 1. 批量导入链路（已有）

- **文件**: `outlook_web/controllers/accounts.py:1008`
- **函数**: `_handle_temp_mail_import()`
- **行为**: 在「账号管理」页面批量导入时，若检测到临时邮箱格式，调用 `temp_mail_service.import_user_mailbox(email, allow_local_fallback=False)`
- **JWT 保证**: `import_user_mailbox` 成功后，会将 `meta.provider_jwt` 写入 `temp_emails.meta_json`，后续 `list_messages` 自动从中读取

### 2. Service 层导入逻辑

- **文件**: `outlook_web/services/temp_mail_service.py:336`
- **函数**: `TempMailService.import_user_mailbox()`

**对 CF Temp Mail 的实际行为**：
1. 本地查重（temp_emails 表）→ 已存在直接返回
2. 尝试 `provider.list_messages(probe_mailbox)` 探测 → CF 需要 JWT，探测失败
3. 尝试 `provider.create_mailbox()` → 调用 CF `POST /admin/new_address`
4. 若成功返回 `{email, meta: {provider_jwt, ...}}`，落库
5. 若失败且 `allow_local_fallback=False`，报错

**JWT 落库链路已完整**：`create_mailbox` 成功 → `_create_or_load_mailbox_record(meta=result.meta)` → `meta_json` 持久化 → 后续 `list_messages` 从 `meta_json` 解析 JWT。

### 3. 现有临时邮箱 API（缺少导入端点）

当前 `temp_emails.py` 路由：
- GET `/api/temp-emails` — 列表
- POST `/api/temp-emails/generate` — 生成
- DELETE `/api/temp-emails/<email>` — 删除
- GET `/api/temp-emails/<email>/messages` — 读邮件
- ...

**缺少**: `POST /api/temp-emails/import` — 导入

---

## 结论

### 后端是否已足够？

**基本足够，只差一个 HTTP 端点暴露**。

`temp_mail_service.import_user_mailbox()` 已经：
- 支持传入完整邮箱地址进行导入/创建
- 成功后将 JWT 写入 `meta_json`
- 后续 `list_messages` 自动读取 `meta_json` 中的 JWT 用于取件

但需要新增：
1. `outlook_web/controllers/temp_emails.py` — `api_import_temp_email()` 函数
2. `outlook_web/routes/temp_emails.py` — `POST /api/temp-emails/import` 路由注册

### 前端需要做什么？

在 `static/js/features/temp_emails.js` 和对应 HTML 模板中：
1. 增加「导入」按钮（与「生成」按钮并列）
2. 增加输入框供用户输入要导入的完整邮箱地址（如 `foo@zerodotsix.top`）
3. 调用 `POST /api/temp-emails/import` 并处理成功/失败反馈

### 改动范围预估

| 文件 | 改动 | 工作量 |
|------|------|--------|
| `outlook_web/controllers/temp_emails.py` | 新增 `api_import_temp_email()` | 小 |
| `outlook_web/routes/temp_emails.py` | 新增 `POST /api/temp-emails/import` | 极小 |
| `static/js/features/temp_emails.js` | 新增导入按钮逻辑 | 小 |
| `templates/index.html` 或 partial | 新增导入输入框 | 极小 |

---

## 关联文件清单

| 文件路径 | 用途 |
|----------|------|
| `outlook_web/services/temp_mail_service.py:336` | 导入核心逻辑 `import_user_mailbox()` |
| `outlook_web/services/temp_mail_provider_cf.py:371` | CF Provider `create_mailbox()` |
| `outlook_web/controllers/accounts.py:1008` | 批量导入调用方（已有参考） |
| `outlook_web/controllers/temp_emails.py` | 待增加导入 handler |
| `outlook_web/routes/temp_emails.py` | 待增加导入路由 |
| `static/js/features/temp_emails.js` | 待增加导入按钮 |

---

## 备注

- 当前 `accounts.py` 的批量导入已使用 `allow_local_fallback=False`，临时邮箱独立导入建议保持一致，避免本地兜底导致「导入成功但无 JWT」的假成功。
- 若 CF Worker `/admin/new_address` 对重复 name 返回 409，导入会失败并返回明确错误，这是预期行为。
- 本分析未修改任何业务代码。
