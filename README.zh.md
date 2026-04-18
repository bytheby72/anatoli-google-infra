# 🔥 适用于 Hermes Agent 的 Google Workspace 基础设施

**[🇺🇸 English](README.md) | [🇷🇺 Русский](README.ru.md)**

> 一次配置，永久忘记。

---

## 😤 痛点：为什么 Google 访问总是断开

如果你曾经为机器人或脚本配置过 Google Drive 或 Gmail 访问权限，你一定知道这种痛苦：

- **OAuth 令牌每小时过期**，refresh token 被 Google 以"可疑活动"为由撤销
- **"Precondition check failed"** — API 明明已启用，令牌也是新的，但 Gmail 就是无法工作
- **Service account** 可以用于 Drive，但**无法访问个人 Gmail**
- 每两周你就要重新打开浏览器，点击"允许"，复制 URL，粘贴到终端...

这不是自动化。这是一份新工作。

---

## ✨ 解决方案：双层架构

这个 skill 一次性彻底解决问题。无需手动重新授权。无需繁琐操作。

| 服务 | 方式 | 为什么永久有效 |
|------|------|--------------|
| **Drive** | Service Account | JSON 密钥。永不过期。永远不会。 |
| **Sheets** | Service Account | 同一个密钥。 |
| **Docs** | Service Account | 同一个密钥。 |
| **Calendar** | Service Account | 同一个密钥。 |
| **Gmail** | OAuth + 自动刷新 | Refresh token 可使用数年。自动更新。 |

### 核心洞察

Google 不提供一种通吃所有服务的方式。Service account 坚如磐石，但对 Gmail 视而不见。OAuth 灵活多变，但喜怒无常。

**我们同时使用两者：**
- **Service account** → Drive、Sheets、Docs、Calendar（除 Gmail 外的所有服务）
- **OAuth** → Gmail（带自动刷新）

Hermes 自动为每个 API 选择正确的方法。你只需写：

```bash
检查网盘
检查邮件
```

---

## 🚀 快速开始

### 1. Service Account（Drive/Sheets/Docs/Calendar）

```bash
# 在 Google Cloud Console 中创建：
# https://console.cloud.google.com/iam-admin/serviceaccounts
# → 创建密钥 → JSON → 保存为：
~/.hermes/google_service_account.json

# 启用 API：
# Drive API、Sheets API、Docs API、Calendar API
```

### 2. OAuth（Gmail）

```bash
# 在 Cloud Console 中启用 Gmail API
# https://console.cloud.google.com/apis/library/gmail.googleapis.com

# 运行授权：
python setup.py --client-secret /路径/to/client_secret.json
python setup.py --auth-url
# 在浏览器中打开链接 → 授权 → 将 URL 粘贴回来
python setup.py --auth-code "浏览器中的URL"
```

### 3. 验证

```bash
python google_api.py gmail search "is:unread" --max 5
python google_api.py drive search "report" --max 5
```

能用吗？能用。**你再也不会需要碰它了。**

---

## 🏗 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Hermes Agent                          │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
   ┌────▼────┐              ┌─────▼─────┐
   │  OAuth  │              │  Service  │
   │  Token  │              │  Account  │
   │(Gmail)  │              │(Drive/    │
   │         │              │ Sheets/   │
   └────┬────┘              │ Docs/Cal) │
        │                   └─────┬─────┘
        │                         │
   ┌────▼────┐              ┌─────▼─────┐
   │ Gmail   │              │ Google    │
   │ API     │              │ Drive API │
   └─────────┘              │ Sheets API│
                            │ Docs API  │
                            │ Calendar  │
                            └───────────┘
```

---

## 🛡 故障恢复（如果出了问题）

| 症状 | 原因 | 修复 |
|------|------|------|
| Drive 为空 | Service account 未共享文件夹 | 与 `hermes-drive@...` 共享文件夹 |
| Gmail 400/401 | OAuth 被撤销 | 通过 `setup.py --auth-url` 重新授权 |
| "Precondition check failed" | Service account 尝试访问 Gmail | 已修复 — Gmail 使用 `force_oauth=True` |

完整说明请参阅 `SKILL.md`。

---

## 💡 "哇"效果

> 你只需**花 10 分钟设置一次**。然后你写 `检查邮件` — 六个月后它仍然在工作。没有定时任务。没有提醒。没有 `invalid_grant`。

这不是脚本。这是会自动运行的基础设施。

---

## 📦 作为 Hermes Skill 安装

```bash
hermes skills install https://github.com/bytheby72/anatoli-google-infra
```

或在 `config.yaml` 中添加：
```yaml
skills:
  external_dirs:
    - https://github.com/bytheby72/anatoli-google-infra
```

---

*Built with 🔥 and zero patience for broken OAuth.*
