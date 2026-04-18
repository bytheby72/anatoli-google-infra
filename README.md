# 🔥 Google Workspace Infrastructure for Hermes Agent

**Настроил один раз — забыл навсегда.**

---

## 😤 Боль: почему Google постоянно отваливается

Если ты настраивал доступ к Google Drive или Gmail для бота/скрипта — ты знаешь эту боль:

- **OAuth токен протухает** через час, refresh token отзывается Google "по подозрению"
- **"Precondition check failed"** — API вроде включён, токен вроде свежий, но Gmail не работает
- **Service account** работает для Drive, но **не может зайти в личную почту Gmail**
- Каждые 2 недели приходится заново лезть в браузер, кликать "разрешить", копировать URL, вставлять в терминал...

Это не автоматизация. Это новая работа.

---

## ✨ Решение: двухуровневая архитектура

Этот skill решает проблему раз и навсегда. Никакого ручного re-auth. Никаких танцев с бубном.

| Сервис | Метод | Почему это вечно |
|--------|-------|------------------|
| **Drive** | Service Account | JSON-ключ. Не протухает. Никогда. |
| **Sheets** | Service Account | Тот же ключ. |
| **Docs** | Service Account | Тот же ключ. |
| **Calendar** | Service Account | Тот же ключ. |
| **Gmail** | OAuth + Auto-Refresh | Refresh token живёт годами. Обновляется сам. |

### Ключевой инсайт

Google не даёт одним способом доступ ко всему. Service account — железобетонный, но слеп к Gmail. OAuth — гибкий, но капризный.

**Мы используем оба:**
- **Service account** → Drive, Sheets, Docs, Calendar (всё, что не Gmail)
- **OAuth** → Gmail (с автоматическим refresh)

Hermes сам выбирает нужный метод для каждого API. Ты просто пишешь:

```bash
проверь драйв
проверь почту
```

---

## 🚀 Быстрый старт

### 1. Service Account (Drive/Sheets/Docs/Calendar)

```bash
# Создай в Google Cloud Console:
# https://console.cloud.google.com/iam-admin/serviceaccounts
# → Create key → JSON → сохрани как:
~/.hermes/google_service_account.json

# Включи API:
# Drive API, Sheets API, Docs API, Calendar API
```

### 2. OAuth (Gmail)

```bash
# Включи Gmail API в Cloud Console
# https://console.cloud.google.com/apis/library/gmail.googleapis.com

# Запусти авторизацию:
python setup.py --client-secret /путь/к/client_secret.json
python setup.py --auth-url
# Открой ссылку в браузере → авторизуй → вставь URL обратно
python setup.py --auth-code "URL_ИЗ_БРАУЗЕРА"
```

### 3. Проверь

```bash
python google_api.py gmail search "is:unread" --max 5
python google_api.py drive search "report" --max 5
```

Работает? Работает. **Больше никогда не придётся трогать.**

---

## 🏗 Архитектура

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

## 🛡 Recovery (если всё же сломалось)

| Симптом | Причина | Фикс |
|---------|---------|------|
| Drive пустой | Service account не расшарен на папку | Поделись папкой с `hermes-drive@...` |
| Gmail 400/401 | OAuth отозван | Переавторизуй через `setup.py --auth-url` |
| "Precondition check failed" | Service account лезет в Gmail | Уже пофикшено — используется `force_oauth=True` для Gmail |

Полная инструкция в `SKILL.md`.

---

## 💡 Эффект "вау"

> Ты настраиваешь это **один раз за 10 минут**. Потом пишешь `проверь почту` — и через 6 месяцев оно всё ещё работает. Без кронов. Без напоминаний. Без `invalid_grant`.

Это не скрипт. Это инфраструктура, которая просто живёт.

---

## 📦 Установка как Hermes Skill

```bash
hermes skills install https://github.com/bytheby72/anatoli-google-infra
```

Или добавь в `config.yaml`:
```yaml
skills:
  external_dirs:
    - https://github.com/bytheby72/anatoli-google-infra
```

---

## 🧠 Автор

Настроено для **Anatoli** через Hermes Agent. 
Если это читаешь не ты — бери, клонируй, адаптируй под себя.

---

*Built with 🔥 and zero patience for broken OAuth.*
