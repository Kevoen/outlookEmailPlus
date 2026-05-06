# TODO: Issue #58 — Cloudflare Temp Mail 导入功能实施计划

> 创建日期：2026-05-06
> 关联 Issue：`https://github.com/ZeroPointSix/outlookEmailPlus/issues/58`
> 关联 BUG 文档：`docs/BUG/2026-05-06-Issue58-Cloudflare-Temp-Mail导入功能详细分析.md`
> 当前目标：为临时邮箱独立页面增加导入入口，复用现有后端 `import_user_mailbox` 能力。

---

## 会话约束（本任务执行时保持）

1. 默认先文档后实现；
2. 每次推进同步 `WORKSPACE.md`；
3. 结果反馈通过 MCP（不只写文档）；
4. 每个 Phase 完成后必须运行相关测试确认无回归；
5. 禁止未经用户确认擅自修改代码。

---

## 任务概览

| 阶段 | 任务数 | 目标 | 状态 |
|---|---:|---|---|
| Phase 0: 基线测试 | 2 | 确认当前临时邮箱相关测试通过，建立回归基线 | ⬜ 待实施 |
| Phase 1: 后端导入 API | 3 | 新增 `POST /api/temp-emails/import` 端点 | ⬜ 待实施 |
| Phase 2: 前端导入按钮 | 3 | 增加导入输入框、按钮、调用逻辑 | ⬜ 待实施 |
| Phase 3: 回归与验收 | 3 | 定向测试、全量回归、人工验收 | ⬜ 待实施 |

---

## Phase 0: 基线测试

**目标**：在修改任何代码前，先确认当前临时邮箱相关测试全部通过，建立可对比的回归基线。

### Task 0.1：运行临时邮箱 Service 层测试
- [ ] 命令：`python -m unittest tests.test_temp_mail_service -v`
- [ ] 预期：全部通过（验证 `import_user_mailbox` 现有逻辑无回归）

### Task 0.2：运行临时邮箱 Controller/API 测试
- [ ] 命令：`python -m unittest tests.test_temp_emails -v`
- [ ] 预期：全部通过（验证现有 API 无回归）
- [ ] 备选：若 `test_temp_emails` 不存在，运行 `python -m unittest discover -s tests -p "*temp*" -v`

---

## Phase 1: 后端导入 API

**目标**：暴露 `import_user_mailbox` 能力为独立 HTTP 端点，供前端调用。

**涉及文件**：
- `outlook_web/controllers/temp_emails.py`
- `outlook_web/routes/temp_emails.py`

### Task 1.1：Controller 新增 `api_import_temp_email()`
- [ ] 在 `outlook_web/controllers/temp_emails.py` 中新增函数：
  ```python
  @login_required
  def api_import_temp_email() -> Any:
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
          return build_error_response(exc.code, exc.message, status=exc.status, message_en="Failed to import temp mailbox")
  ```
- [ ] 检查是否需要新增 `import re`

### Task 1.2：路由注册 `POST /api/temp-emails/import`
- [ ] 在 `outlook_web/routes/temp_emails.py` 的 `handlers` 列表中追加：
  ```python
  (
      "/api/temp-emails/import",
      temp_emails_controller.api_import_temp_email,
      ["POST"],
  ),
  ```

### Task 1.3：运行测试
- [ ] `python -m compileall -q outlook_web/controllers/temp_emails.py outlook_web/routes/temp_emails.py`
- [ ] `python -m unittest tests.test_temp_mail_service tests.test_temp_emails -v`
- [ ] 如有新增测试文件需求，在 Phase 3 补齐

---

## Phase 2: 前端导入按钮

**目标**：在临时邮箱页面提供直观的导入入口，与现有「生成」按钮并列。

**涉及文件**：
- `templates/index.html`
- `static/js/features/temp_emails.js`
- `static/js/i18n.js`

### Task 2.1：模板增加导入输入框与按钮
- [ ] 文件：`templates/index.html`
- [ ] 位置：临时邮箱左侧面板输入区域（`tempEmailDomainSelect` 下方，`tempEmailOptionsHint` 上方或下方）
- [ ] 新增 HTML：
  ```html
  <div style="display:flex;gap:8px;margin-top:4px;">
      <input type="text" class="form-input" id="tempEmailImportInput" placeholder="输入要导入的邮箱地址" style="flex:1;">
      <button class="btn btn-sm btn-primary" onclick="importTempEmail()">导入</button>
  </div>
  ```
- [ ] 确保样式与现有 `form-input` / `btn btn-sm` 保持一致

### Task 2.2：`temp_emails.js` 新增 `importTempEmail()`
- [ ] 文件：`static/js/features/temp_emails.js`
- [ ] 在 `generateTempEmail()` 附近新增函数：
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

### Task 2.3：`i18n.js` 补充词条
- [ ] 文件：`static/js/i18n.js`
- [ ] 在 `translations` 对象中补充（中/英）：
  - `导入临时邮箱` / `Import Temp Mailbox`
  - `请输入邮箱地址` / `Please enter an email address`
  - `正在导入…` / `Importing…`
  - `导入成功` / `Import successful`
  - `导入失败` / `Import failed`

---

## Phase 3: 回归与验收

**目标**：确认新功能不破坏现有链路，且人工验收通过。

### Task 3.1：定向回归测试
- [ ] `python -m compileall -q outlook_web/controllers/temp_emails.py outlook_web/routes/temp_emails.py static/js/features/temp_emails.js static/js/i18n.js`
- [ ] `python -m unittest tests.test_temp_mail_service tests.test_temp_emails -v`
- [ ] 若存在 `test_temp_mail_plugin_api`、`test_temp_mail_plugin_e2e` 等，一并运行

### Task 3.2：全量回归测试
- [ ] `python -m unittest discover -s tests -v`
- [ ] 确认无新增失败（与 Phase 0 基线对比）

### Task 3.3：人工验收
- [ ] 启动本地服务：`python start.py`
- [ ] 打开临时邮箱页面
- [ ] 在导入输入框中输入一个 CF Temp Mail 地址（如 `test@zerodotsix.top`）
- [ ] 点击「导入」按钮
- [ ] 验证：列表中出现该邮箱
- [ ] 验证：能正常读取邮件（如有邮件）
- [ ] 验证：能正常提取验证码
- [ ] 验证：能正常删除该邮箱
- [ ] 验证：重复导入同一邮箱时给出合理提示（已存在或成功）

---

## 风险与注意事项

| 风险点 | 说明 | 缓解措施 |
|-------|------|---------|
| CF Worker 对重复 name 返回 409 | 导入会失败，用户看到报错 | 后端返回明确错误码，前端展示友好提示 |
| 导入后 JWT 失效 | CF Worker 侧 JWT 理论上可能过期 | 当前架构无 JWT 刷新机制，若发生需手动删除后重新导入 |
| 与批量导入语义不一致 | `accounts.py` 批量导入已用 `allow_local_fallback=False` | Task 1.1 已明确使用 `allow_local_fallback=False` 保持一致 |
| 前端 i18n 词条遗漏 | 不同语言环境下提示文案缺失 | Task 2.3 同时补充中英词条，默认回退英文 |

---

## 关联文件清单

| 文件路径 | 用途 | 改动类型 |
|----------|------|---------|
| `outlook_web/controllers/temp_emails.py` | 新增导入 handler | 新增函数 |
| `outlook_web/routes/temp_emails.py` | 注册导入路由 | 新增路由 |
| `templates/index.html` | 新增导入输入框与按钮 | 新增 HTML |
| `static/js/features/temp_emails.js` | 新增导入前端逻辑 | 新增函数 |
| `static/js/i18n.js` | 补充导入相关词条 | 新增词条 |
