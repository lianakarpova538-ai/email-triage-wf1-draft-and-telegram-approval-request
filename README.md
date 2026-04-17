# email-triage-wf1-draft-and-telegram-approval-request

<img width="1482" height="527" alt="image" src="https://github.com/user-attachments/assets/41575ddb-f862-4216-ad8c-0f1d20a14036" />


## Что сделано

Автоматизация обработки входящей почты в **n8n (self-hosted)**:

- Триггер **Gmail** → нормализация тела письма (plain/HTML → поля для пайплайна).
- Классификация письма через **LLM** (тип письма, приоритет, нужен ли ответ, действие).
- Парсинг ответа модели в **Code** + **Merge** исходных данных письма с результатом классификации.
- **Switch** по `emailType` (ветки: работа, промо, транзакционные, важные, личные).
- Для веток с черновиком ответа: **Markdown** (при необходимости) → **Write email** → подготовка записи одобрения → **Google Sheets** (лог `PENDING`) → **Telegram** с текстом `APPROVE / REJECT` и `approvalId`.

**Workflow 2** (отдельный workflow): приём ответа в Telegram → поиск строки в Google Sheets → при `APPROVE` — отправка письма получателю; при `REJECT` — обновление статуса без отправки.

## Процесс работы и промпты

### Нормализация письма (Code после Gmail)

**Идея:**
> Сохранить все поля Gmail, выделить `emailText` / `emailTextFull`, собрать `htmlForMarkdown` для узла Markdown.

**С чем столкнулась:**
- После узла с ответом LLM «сырое» письмо терялось, если не делать **Merge** или не пробрасывать `...$json`.
- Поля тела письма в n8n называются по-разному (`bodyPlain`, `bodyHtml`, `snippet`).

**Что помогло:**
- Явное разделение plain/HTML и fallback `snippet`.
- **Merge** после классификации: один поток с исходным письмом, второй — с полями AI.

  
<img width="1860" height="514" alt="image" src="https://github.com/user-attachments/assets/8e7de335-9326-4dc3-895f-fea316d013a6" />


### Switch по типам писем

| 1 — Work | Черновик → Sheets → Telegram (одобрение) |
| 2 — Promotion | Отдельная рассылка / промо-почта |
| 3 — Transactional | Извлечение фактов → уведомление в Telegram |
| 4 — Sensitive | Уведомление в Telegram (без автоотправки) |
| 5 — Personal | Как work: черновик → Sheets → Telegram |


<img width="1863" height="774" alt="image" src="https://github.com/user-attachments/assets/b2c45a85-dd03-41c4-bed3-4eafad5365c5" />


### Подготовка одобрения + Google Sheets + Telegram

**Code `prep approval record`:**
- `approvalId`, `recipientEmail` (разбор `From`), `draftBody`, `originalSubject`, `threadId`, `approvalStatus: PENDING`, `createdAt`.

**Google Sheets:**
- Таблица `pending_approvals`: колонки `approvalId`, `recipientEmail`, `originalSubject`, `draftBody`, `threadId`, `approvalStatus`, `createdAt`, `from`, `emailType`.

**Telegram:**
```text
Approval needed: {{$json.approvalId}}
...
APPROVE {{$json.approvalId}}
REJECT {{$json.approvalId}}
```


<img width="655" height="660" alt="image" src="https://github.com/user-attachments/assets/8f0fd816-9c95-4357-ba88-f0628f87c41b" />

