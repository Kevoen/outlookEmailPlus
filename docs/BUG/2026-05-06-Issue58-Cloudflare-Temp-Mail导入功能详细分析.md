# Cloudflare Temp Mail 添加导入功能（Issue #58）

**创建日期**: 2026-05-06
**关联 Issue**: https://github.com/ZeroPointSix/outlookEmailPlus/issues/58
**关联模块**: `outlook_web/services/temp_mail_service.py`、`outlook_web/controllers/temp_emails.py`、`outlook_web/routes/temp_emails.py`、`static/js/features/temp_emails.js`
**状态**: 🟡 分析完成，用户确认轻量方案（前端按钮 + 复用现有后端），待进入实现
**优先级建议**: P2（功能增强，不影响现有链路）

---

## 1. 问题概述

用户在 GitHub Issue #58 中请求为 **Cloudflare Temp Mail** 添加导入功能。当前临时邮箱页面仅支持「生成」新邮箱，不支持手动输入已有邮箱地址进行导入。

版本：**v2.3.0**

标签：`enhancement`

---

## 2. 需求描述

### 2.1 用户诉求

- 在 Cloudflare Temp Mail 管理页面增加「导入」功能
- 允许用户手动输入或粘贴已有的 CF Temp Mail 邮箱地址
- 导入后该邮箱应能正常读取邮件、提取验证码、删除邮件

### 2.2 当前能力差距

| 功能 | 账号管理页（批量导入） | 临时邮箱独立页面 | 差距 |
|------|----------------------|-----------------|------|
| 生成新邮箱 | ❌ 不支持 | ✅ 支持 | — |
| 导入临时邮箱 | ✅ 支持（混合在批量导入中） | ❌ 不支持 | **缺独立入口** |
| 删除临时邮箱 | ❌ 不支持 | ✅ 支持 | — |
| 读取邮件 | ❌ 不支持 | ✅ 支持 | — |

> 账号管理页的批量导入虽然支持临时邮箱（通过 `_handle_temp_mail_import`），但用户可能不知道这个入口，且混合在大量普通账号导入中不够直观。

---

## 3. 深度分析

### 3.1 全链路梳理

```
[用户] 在临时邮箱页面点击「导入」按钮，输入 foo@zerodotsix.top
    ↓
[前端] static/js/features/temp_emails.js — 调用 POST /api/temp-emails/import
    ↓
[路由] outlook_web/routes/temp_emails.py — /api/temp-emails/import
    ↓
[控制器] outlook_web/controllers/temp_emails.py — api_import_temp_email()
    ↓
[服务] outlook_web/services/temp_mail_service.py — import_user_mailbox()
    ↓
[Provider] outlook_web/services/temp_mail_provider_cf.py — create_mailbox()
    ↓ POST /admin/new_address
[CF Worker] 创建/返回邮箱地址 + JWT
    ↓
[数据库] temp_emails 表 — 写入记录（含 meta_json.provider_jwt）
```

### 3.2 后端 Service 层：`import_user_mailbox()`

**位置**：`outlook_web/services/temp_mail_service.py:336`

```python
def import_user_mailbox(self, email_addr: str, *, allow_local_fallback: bool = True) -> dict[str, Any]:
    normalized_email = str(email_addr or "").strip()
    if not normalized_email:
        raise TempMailError("INVALID_PARAM", "邮箱地址不能为空", status=400)

    existing = temp_emails_repo.get_temp_email_by_address(normalized_email)
    if existing:
        return _mailbox_from_record(existing)

    prefix = None
    domain = None
    if "@" in normalized_email:
        prefix, domain = normalized_email.rsplit("@", 1)

    provider = None
    provider_name = None
    try:
        provider = self._get_provider(purpose="runtime")
        provider_name = str(getattr(provider, "provider_name", "") or "").strip() or None
    except TempMailError:
        if not allow_local_fallback:
            raise

    if provider is not None:
        probe_mailbox = {
            "kind": temp_emails_repo.TEMP_MAIL_KIND,
            "email": normalized_email,
            "provider_name": provider_name,
            ...
        }
        try:
            probe_result = provider.list_messages(probe_mailbox)
        except Exception:
            probe_result = None
        else:
            if isinstance(probe_result, list) and len(probe_result) > 0:
                return self._create_or_load_mailbox_record(...)

        try:
            create_result = self._create_mailbox(provider, prefix=prefix, domain=domain)
        except Exception:
            create_result = None
        if isinstance(create_result, dict) and create_result.get("success"):
            created_email = str(create_result.get("email") or "").strip() or normalized_email
            return self._create_or_load_mailbox_record(..., meta=create_result.get("meta"), ...)

    if not allow_local_fallback:
        raise TempMailError("TEMP_EMAIL_CREATE_FAILED", "临时邮箱导入失败", status=502)

    return self._create_or_load_mailbox_record(...)
```

**对 CF Temp Mail 的实际行为**：

1. **本地查重**：检查 `temp_emails` 表，已存在则直接返回
2. **上游探测**：`provider.list_messages(probe_mailbox)` → CF provider 需要 JWT，`probe_mailbox` 无 JWT → 抛 `UNAUTHORIZED` → 探测失败
3. **上游创建**：调用 `_create_mailbox(provider, prefix, domain)` → CF `POST /admin/new_address`
   - 若 CF Worker 返回已有地址 + JWT → 成功导入，JWT 随 `meta` 落库
   - 若 CF Worker 返回 409 / 错误 → 导入失败
4. **allow_local_fallback**：
   - `accounts.py` 批量导入传 `False` → 失败直接报错
   - 默认 `True` → 失败则本地兜底写入（无 JWT，后续取件会 401）

**JWT 落库链路**：
```
create_mailbox() 返回 {email, meta: {provider_jwt: "eyJ...", provider_mailbox_id: 12345}}
    ↓
_create_or_load_mailbox_record(meta=create_result.get("meta"))
    ↓
temp_emails_repo.insert_temp_email(meta_json=json.dumps(meta))
    ↓
数据库 temp_emails.meta_json 字段持久化 JWT
    ↓
后续 list_messages(mailbox) 时
    ↓
CF provider._get_jwt(mailbox) 从 meta_json 解析 provider_jwt
    ↓
请求 GET /api/mails 时携带 Authorization: Bearer <JWT>
```

**结论**：`import_user_mailbox` 已经完整实现了「创建/导入 → JWT 持久化 → 后续取件使用 JWT」的闭环。当前瓶颈不在 Service 层，而在**该能力未通过独立 HTTP API 暴露给临时邮箱页面**。

### 3.3 后端 Controller 层：临时邮箱控制器

**位置**：`outlook_web/controllers/temp_emails.py`

当前已有 API：
- `api_get_temp_emails()` — GET /api/temp-emails
- `api_generate_temp_email()` — POST /api/temp-emails/generate
- `api_delete_temp_email()` — DELETE /api/temp-emails/<email>
- `api_get_temp_email_messages()` — GET /api/temp-emails/<email>/messages
- ...

**缺失**：`api_import_temp_email()` — POST /api/temp-emails/import

### 3.4 后端路由层：临时邮箱路由

**位置**：`outlook_web/routes/temp_emails.py`

当前路由注册：
```python
handlers = [
    ("/api/temp-emails", api_get_temp_emails, ["GET"]),
    ("/api/temp-emails/options", api_get_temp_email_options, ["GET"]),
    ("/api/temp-emails/generate", api_generate_temp_email, ["POST"]),
    # ... 无 import 路由
]
```

### 3.5 前端层：`temp_emails.js`

**位置**：`static/js/features/temp_emails.js`

当前临时邮箱页面功能：
- `loadTempEmails()` — 加载列表
- `generateTempEmail()` — 生成新邮箱（调用 POST /api/temp-emails/generate）
- `selectTempEmail()` — 选中邮箱
- `loadTempEmailMessages()` — 加载邮件
- `deleteTempEmail()` — 删除邮箱

**缺失**：`importTempEmail()` — 导入邮箱（无函数、无按钮、无输入框）

---

## 4. 影响范围

### 4.1 直接影响

1. **临时邮箱独立页面**：用户无法直观导入已有 CF Temp Mail 邮箱
2. **用户体验**：必须通过「账号管理」批量导入入口间接完成，学习成本高

### 4.2 间接影响

1. **无**：该功能为纯新增，不影响现有临时邮箱生成/读取/删除链路
2. **无**：不改动 `accounts.py` 批量导入逻辑，现有批量导入能力保持不变

### 4.3 不影响

1. **普通邮箱（Outlook/IMAP）**：完全无关
2. **CF Worker 外部 API / 邮箱池**：不改动 pool/claim 链路
3. **现有临时邮箱生成**：生成按钮和逻辑保持原样

---

## 5. 实现方案建议

### Phase 1：后端新增导入 API（最小改动）

**目标**：暴露 `import_user_mailbox` 能力给临时邮箱页面

**修改点 1**：`outlook_web/controllers/temp_emails.py`

新增函数：
```python
@login_required
def api_import_temp_email() -> Any:
    """导入已有临时邮箱（CF Temp Mail）"""
    data = request.json or {}
    email = str(data.get("email") or "").strip()
    
    if not email:
        return build_error_response("INVALID_PARAM", "邮箱地址不能为空", status=400)
    
    if not re.match(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", email):
        return build_error_response("INVALID_PARAM", "邮箱格式不正确", status=400)
    
    try:
        mailbox = temp_mail_service.import_user_mailbox(email, allow_local_fallback=False)
        log_audit("import", "temp_email", email, "导入临时邮箱")
        return jsonify({
            "success": True,
            "email": mailbox["email"],
            "message": "临时邮箱导入成功",
            "message_en": "Temp mailbox imported successfully",
        })
    except TempMailError as exc:
        return build_error_response(
            exc.code,
            exc.message,
            status=exc.status,
            message_en="Failed to import temp mailbox",
        )
```

**修改点 2**：`outlook_web/routes/temp_emails.py`

在 `handlers` 列表中追加：
```python
(
    "/api/temp-emails/import",
    temp_emails_controller.api_import_temp_email,
    ["POST"],
),
```

### Phase 2：前端增加导入按钮与输入框

**目标**：在临时邮箱页面提供直观的导入入口

**修改点 1**：`templates/index.html`（或对应 partial）

在临时邮箱控制栏（与「生成」按钮同一区域）增加：
- 输入框：`id="tempEmailImportInput"`，placeholder="输入要导入的邮箱地址"
- 按钮：`onclick="importTempEmail()"`，文案「导入」

**修改点 2**：`static/js/features/temp_emails.js`

新增函数：
```javascript
async function importTempEmail() {
    const input = document.getElementById('tempEmailImportInput');
    const email = (input && input.value || '').trim();
    
    if (!email) {
        showToast(translateAppTextLocal('请输入邮箱地址'), 'warning');
        return;
    }
    
    try {
        showToast(translateAppTextLocal('正在导入…'), 'info');
        const response = await fetch('/api/temp-emails/import', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email })
        });
        
        const data = await response.json();
        
        if (data.success) {
            showToast(data.message || translateAppTextLocal('导入成功'), 'success');
            if (input) input.value = '';
            delete accountsCache['temp'];
            loadTempEmails(true);
        } else {
            handleApiError(data, translateAppTextLocal('导入失败'));
        }
    } catch (error) {
        showToast(translateAppTextLocal('导入失败'), 'error');
    }
}
```

### Phase 3：国际化词条补充

**修改点**：`static/js/i18n.js`

新增词条（中/英）：
- `导入临时邮箱` / `Import Temp Mailbox`
- `请输入邮箱地址` / `Please enter an email address`
- `导入成功` / `Import successful`
- `导入失败` / `Import failed`

---

## 6. 验证思路

### 6.1 单元测试

1. **Controller 层**：`tests/test_temp_emails.py`（或新增测试文件）
   - 正常导入：POST /api/temp-emails/import 返回 200，数据库有记录，meta_json 含 provider_jwt
   - 重复导入：再次导入同一地址，返回 200 或 409（视语义而定）
   - 格式错误：POST 非法邮箱，返回 400
   - 上游失败：mock CF Worker 返回 409，返回 502 且本地无记录（allow_local_fallback=False）

2. **Service 层**：`tests/test_temp_mail_service.py`
   - 验证 `import_user_mailbox` 对 CF provider 的调用链路已存在（已有测试覆盖）

### 6.2 人工验收

1. 在临时邮箱页面输入一个 CF Temp Mail 地址，点击导入
2. 验证列表中出现该邮箱
3. 向该邮箱发送测试邮件，验证能正常读取
4. 验证提取验证码功能正常
5. 验证删除邮箱功能正常

---

## 7. 当前真实状态

1. **Issue #58 状态**：GitHub 上仍为 Open，无 PR
2. **代码状态**：未做任何修改，等待用户确认后进入实现
3. **分析状态**：已完成全链路代码走读，确认后端 `import_user_mailbox` 已具备完整 JWT 闭环能力
4. **建议实现顺序**：Phase 1（后端 API）→ Phase 2（前端按钮）→ Phase 3（i18n）

---

## 8. 关联文档

- `WORKSPACE.md` 第 257 条：Issue #58 实现分析记录
- `docs/BUG/2026-05-06-Issue58-CF-Temp-Mail导入功能-JWT瓶颈分析.md`：精简版实现分析
- `outlook_web/services/temp_mail_service.py:336`：`import_user_mailbox()` 核心逻辑
- `outlook_web/controllers/accounts.py:1008`：批量导入中临时邮箱处理参考

---

## 9. 风险与注意事项

| 风险点 | 说明 | 缓解措施 |
|-------|------|---------|
| CF Worker 对重复 name 返回 409 | 导入会失败，用户看到报错 | 前端提示"该邮箱可能已在 CF Worker 存在但无法获取凭证，请确认后重试" |
| 导入后 JWT 失效 | CF Worker 侧 JWT 理论上可能过期 | 当前架构无 JWT 刷新机制，若发生需手动删除后重新导入 |
| 与批量导入语义不一致 | `accounts.py` 批量导入已用 `allow_local_fallback=False`，独立导入建议保持一致 | Phase 1 方案已明确使用 `allow_local_fallback=False` |
