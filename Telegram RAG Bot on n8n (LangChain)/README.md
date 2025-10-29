# Telegram RAG Bot on n8n (LangChain)

Учебный workflow для Telegram‑бота, который:
- Индексирует документы, присланные в чат (/index), создаёт эмбеддинги и сохраняет их в векторное хранилище.
- Отвечает на вопросы по проиндексированным данным (/ask) через семантический поиск и LangChain Q&A Chain.

## Архитектура
- Telegram Trigger — приём сообщений и команд.
- Состояние (workflowStaticData) — контроль ожиданий: файл для /index, текст для /ask.
- Индексация: Get File → Default Data Loader → Embeddings OpenAI → Simple Vector Store (in‑memory).
- Q&A: Vector Store Retriever → Question and Answer Chain (Chat Model) → Telegram Send.

## Требования
- n8n 1.** с включёнными LangChain‑нодами.
- Аккаунт Telegram BotFather (bot token).
- API‑ключ LLM/Embeddings (например, OpenAI‑совместимый).
- Публичный URL/туннель для Telegram webhook, либо режим polling.

## Установка
1. Импортируйте JSON (study‑project‑clean) в n8n.
2. Создайте Credentials:
   - Telegram API: Bot Token.
   - OpenAI (или совместимый): API‑ключ и при необходимости Base URL.
3. Откройте узлы Telegram (Trigger/Send) и укажите созданные credentials.
4. Откройте узлы Embeddings OpenAI и OpenAI Chat Model — укажите credentials и нужные модели.
5. Запустите workflow (Active). Настройте webhook URL, если требуется.

## Команды
- /index — бот ожидает файл (PDF/MD/TXT/HTML) или URL следующим сообщением.
- /ask — бот ожидает текст вопроса; выполняет поиск и отвечает кратко со ссылками на источники.

## Как это работает
- После /index:
  - Бот принимает документ, Default Data Loader извлекает текст и добавляет metadata (title, chat_id, author, mime).
  - Embeddings OpenAI создаёт эмбеддинги чанков.
  - Simple Vector Store сохраняет эмбеддинги и metadata.
  - Бот подтверждает количество добавленных данных.
- После /ask:
  - Vector Store Retriever находит релевантные фрагменты.
  - Q&A Chain формирует ответ на основе контекста и выбранной chat‑модели.
  - Бот отправляет ответ и список источников (из metadata).

## Параметры по умолчанию
- Разделение текста: настраивается в Default Data Loader (простое/рекурсивное).
- top_k (Retriever): рекомендуем 3–5.
- Температура модели: ниже для фактичности (0–0.3).
- Simple Vector Store: неперсистентный, общий для инстанса (только для разработки).

## Миграция в прод
- Замените Simple Vector Store на Pinecone/Qdrant/Supabase (нужные ноды).
- Используйте namespace/collection или поля chat_id/project в metadata для изоляции пользователей.
- Добавьте планировщик переиндексации и логику очистки (delete по фильтрам или namespace).

## Безопасность
- Ограничьте доступ к боту по списку user_id (фильтр).
- Не храните секреты в узлах — только в Credentials.
- Держите журнал попыток и ошибок (отдельный workflow/таблица).

## Ограничения
- Simple Vector Store очищается при рестарте.
- sub‑nodes (Retriever/Loader) ожидают один item; для батчей используйте разбиение.
- Telegram Markdown/HTML требует экранирования спецсимволов.

## Тестирование
1) /index → отправьте PDF → получите подтверждение.
2) /index → отправьте текст вместо файла → получите подсказку.
3) /ask → отправьте вопрос → получите ответ + источники.
4) /ask → отправьте файл без текста → получите подсказку.

## Настройка формата ответов
- Редактируйте промпт в Question and Answer Chain (инструкция “отвечай кратко, цитируй источники”).
- Формируйте список источников из metadata: title и source_url (если присутствует).

## Полезное
- Для мульти‑пользовательских сценариев храните chat_id в metadata и фильтруйте поиск/очистку.
- Для персистентности и масштабирования используйте Supabase/pgvector, Qdrant или Pinecone.
