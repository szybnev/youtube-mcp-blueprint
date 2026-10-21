# YouTube MCP: собственный сервер для публикации видео агентом

Задача: дать ИИ-агенту (Claude Code, Codex, Cursor и др.) инструменты для выкладки готового выпуска на YouTube через [Model Context Protocol](https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro) (MCP). Поверх официального YouTube Data API v3, с авторизацией владельца канала и под его контролем.

Репозиторий описывает задачу: что нужно, что технически возможно, какие готовые реализации есть и где документация. Кода сервера здесь пока нет.

## Контекст

Агент уже готовит подкаст к публикации целиком: монтаж с плашками, Shorts, субтитры, обложки, описания с главами, A/B-набор названий и обложек. На выходе папка с файлами. Последний шаг — выкладка на YouTube — пока делается руками в YouTube Studio:

| Файл | Куда в YouTube |
|---|---|
| основное видео (4K MP4) | загрузка, название, описание с главами, теги, язык |
| `*.srt` / `*.vtt` | дорожка субтитров |
| обложка 1280×720 | основная обложка |
| 3 названия + 3 обложки | тест «Тест и сравнение» (A/B) |
| 8–10 Shorts + описания | загрузка каждого, «Связанное видео» на основной ролик |
| время выхода | отложенная публикация |

Цель — свести ручную работу к проверке и одной кнопке «Опубликовать».

## Что можно через API, а что нет

| Действие | API | Метод / источник |
|---|---|---|
| Загрузить видео и Shorts, задать метаданные и статус | да | [`videos.insert`](https://developers.google.com/youtube/v3/docs/videos/insert), [возобновляемая загрузка](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol) |
| Отложенная публикация | да | `status.publishAt` в `videos.insert` / [`videos.update`](https://developers.google.com/youtube/v3/docs/videos/update) |
| Изменить название, описание, теги | да | [`videos.update`](https://developers.google.com/youtube/v3/docs/videos/update) |
| Загрузить субтитры (SRT/VTT с таймкодами) | да | [`captions.insert`](https://developers.google.com/youtube/v3/docs/captions/insert) |
| Поставить обложку | да | [`thumbnails.set`](https://developers.google.com/youtube/v3/docs/thumbnails/set), до 50 МБ с сентября 2026 ([история изменений](https://developers.google.com/youtube/v3/revision_history)) |
| Добавить в плейлист | да | [`playlistItems.insert`](https://developers.google.com/youtube/v3/docs/playlistItems/insert) |
| Статистика и аналитика | да | [YouTube Analytics API](https://developers.google.com/youtube/analytics) |
| A/B-тест названий и обложек | **нет** | только в Studio: [A/B test titles & thumbnails](https://support.google.com/youtube/answer/16391400) |
| «Связанное видео» у Shorts | **нет** | только в Studio: [Add a related video to your Shorts](https://support.google.com/youtube/answer/14075157) |

Главы отдельного API не требуют: YouTube строит их из таймкодов в описании ([Video Chapters](https://support.google.com/youtube/answer/9884579)).

### Квоты

По [Getting started](https://developers.google.com/youtube/v3/getting-started) и [калькулятору квот](https://developers.google.com/youtube/v3/determine_quota_cost) у нового проекта по умолчанию:

- 100 вызовов `videos.insert` в сутки — отдельная корзина с июня 2026;
- 100 вызовов `search.list` в сутки;
- 10 000 единиц в сутки на всё остальное: `captions.insert` — 400, `videos.update`, `thumbnails.set`, `playlistItems.insert` — по 50.

Выпуск с 8 Shorts — это 9 загрузок и около 1 000 единиц. Лимиты не мешают.

### Риски

- **Публичная публикация из непроверенного проекта.** Документация противоречит сама себе. На странице [`videos.insert`](https://developers.google.com/youtube/v3/docs/videos/insert) сказано, что такие видео не ограничены приватным режимом. В [описании ресурса](https://developers.google.com/youtube/v3/docs/videos), в [истории изменений](https://developers.google.com/youtube/v3/revision_history) (28.07.2020) и в справке [Videos locked as private](https://support.google.com/youtube/answer/7300965) — наоборот, приватны до аудита. Вопрос открыт в [трекере Google](https://issuetracker.google.com/issues/572863050). Безопасный режим: загружать `private`, публиковать отдельным подтверждённым шагом. Аудит и увеличение квот — через [форму YouTube API](https://support.google.com/youtube/contact/yt_api_form).
- **Тестовый режим OAuth.** Если экран согласия в статусе Testing, refresh-токен живёт 7 дней ([Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)) и вход придётся повторять. Постоянный токен требует перевести приложение в Production и, для чувствительных scopes, пройти [проверку приложения](https://support.google.com/cloud/answer/13463073).
- **Полный доступ к каналу.** Scope `youtube.force-ssl` (нужен для субтитров) даёт агенту право удалять видео и писать комментарии. Сервер должен сам ограничивать набор действий.

## Требования к собственному серверу

### Инструменты MCP

| Инструмент | Что делает | Ограничения |
|---|---|---|
| `upload_video` | возобновляемая загрузка файла + метаданные | всегда `private`; прогресс через [progress notifications](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/progress) |
| `set_thumbnail` | обложка для видео | проверка 1280×720, размер |
| `upload_captions` | SRT/VTT, язык, имя дорожки | файл с таймкодами |
| `update_metadata` | название, описание (главы), теги, язык | — |
| `add_to_playlist` | добавить видео в плейлист | — |
| `get_video_status` | статус обработки, приватность, ссылки | только чтение |
| `schedule_publish` / `publish` | `publishAt` или `public` | только после явного подтверждения пользователя ([elicitation](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation)) |
| `publish_episode` | весь выпуск по манифесту: видео, субтитры, обложка, Shorts | идемпотентно: повторный запуск не создаёт дублей |

Сознательно вне рамок: удаление видео, комментарии, A/B-тест и «Связанное видео». Последние два в API отсутствуют; для них сервер выводит чек-лист ручных шагов в Studio.

### Манифест выпуска

Сервер принимает один файл с путями и метаданными: видео, описание, субтитры, обложка, список Shorts с описаниями, время публикации. Состояние (загруженные `videoId`) сохраняется рядом, чтобы повторный запуск продолжал, а не дублировал.

### Авторизация и хранение

- [OAuth для установленных приложений](https://developers.google.com/identity/protocols/oauth2/native-app) (Desktop client, loopback redirect, PKCE).
- Минимальные [scopes](https://developers.google.com/identity/protocols/oauth2/scopes): `youtube.upload` + `youtube.force-ssl` (субтитры); `youtube.readonly` для чтения.
- Refresh-токен — в системном хранилище ключей (Keychain), не в файле проекта.
- Локальный сервер по stdio ([Local Server Security](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/local-server-security)). Удалённый вариант — только с авторизацией по [MCP Authorization](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization).

### Критерии готовности

- Тестовый выпуск (видео + SRT + обложка + 2 Shorts) загружается одной командой агента в статусе `private`. Повторный запуск ничего не дублирует.
- Публикация невозможна без подтверждения пользователя.
- Ошибки квоты и сети не теряют прогресс загрузки: загрузка возобновляется.
- Проверено в [MCP Inspector](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector) и в одном клиенте (Claude Code или Codex).

## Готовые реализации

Данные на октябрь 2026. Код этих проектов не проверялся; перед выдачей OAuth-доступа к каналу их нужно читать.

### С записью (OAuth владельца канала)

| Проект | Язык, лицензия | Что умеет | Замечания |
|---|---|---|---|
| [anwerj/youtube-uploader-mcp](https://github.com/anwerj/youtube-uploader-mcp) | Go, MIT | загрузка, статус и `publishAt`, обложка, субтитры, плейлисты | ближе всего к задаче; субтитры требуют `youtube.force-ssl` |
| [pauling-ai/youtube-mcp-server](https://github.com/pauling-ai/youtube-mcp-server) | Python, MIT | Data + Analytics + Reporting API, загрузка, обложки, плейлисты, комментарии | ~40 инструментов, широкий доступ |
| [miller-joe/youtube-mcp](https://github.com/miller-joe/youtube-mcp) ([npm](https://www.npmjs.com/package/@miller-joe/youtube-mcp)) | TypeScript, MIT | OAuth PKCE, метаданные, субтитры SRT/VTT, обложки, плейлисты, комментарии, аналитика | загрузки видео нет; удаление с подтверждением по названию; данные о квоте загрузки устарели |
| [i1s-abhishek/youtube-studio-mcp](https://github.com/i1s-abhishek/youtube-studio-mcp) | Python, MIT | метаданные, обложки, комментарии, аналитика | без загрузки видео |

### Только чтение (поиск, транскрипты, статистика)

| Проект | Язык, лицензия | Чем полезен |
|---|---|---|
| [ZubeidHendricks/youtube-mcp-server](https://github.com/ZubeidHendricks/youtube-mcp-server) | TypeScript, MIT | самый популярный; видео, каналы, плейлисты, транскрипты |
| [kirbah/mcp-youtube](https://github.com/kirbah/mcp-youtube) | TypeScript, MIT | компактные ответы для LLM |
| [ShawhinT/yt-mcp](https://github.com/ShawhinT/yt-mcp), [ShawhinT/yt-mcp-remote](https://github.com/ShawhinT/yt-mcp-remote) | Python | учебные примеры локального и удалённого сервера |

Ни одна реализация не умеет A/B-тест и «Связанное видео»: этого нет в самом API.

## Документация

### YouTube

- [YouTube Data API v3](https://developers.google.com/youtube/v3) · [Getting started](https://developers.google.com/youtube/v3/getting-started) · [Authentication](https://developers.google.com/youtube/v3/guides/authentication)
- [Uploading a video](https://developers.google.com/youtube/v3/guides/uploading_a_video) · [Resumable uploads](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol)
- Ресурсы: [videos](https://developers.google.com/youtube/v3/docs/videos) · [captions.insert](https://developers.google.com/youtube/v3/docs/captions/insert) · [thumbnails.set](https://developers.google.com/youtube/v3/docs/thumbnails/set) · [playlistItems.insert](https://developers.google.com/youtube/v3/docs/playlistItems/insert) · [ошибки](https://developers.google.com/youtube/v3/docs/errors)
- [Калькулятор квот](https://developers.google.com/youtube/v3/determine_quota_cost) · [История изменений API](https://developers.google.com/youtube/v3/revision_history)
- [YouTube Analytics API](https://developers.google.com/youtube/analytics)
- Правила: [API Services Terms of Service](https://developers.google.com/youtube/terms/api-services-terms-of-service) · [Developer Policies](https://developers.google.com/youtube/terms/developer-policies) · [форма аудита и квот](https://support.google.com/youtube/contact/yt_api_form)
- Справка: [A/B-тест](https://support.google.com/youtube/answer/16391400) · [Связанное видео для Shorts](https://support.google.com/youtube/answer/14075157) · [Главы](https://support.google.com/youtube/answer/9884579) · [Субтитры](https://support.google.com/youtube/answer/2734796) · [Videos locked as private](https://support.google.com/youtube/answer/7300965)
- [Официальные примеры кода](https://github.com/youtube/api-samples)

### Google

- [OAuth 2.0 overview](https://developers.google.com/identity/protocols/oauth2) · [OAuth для установленных приложений](https://developers.google.com/identity/protocols/oauth2/native-app) · [Список scopes](https://developers.google.com/identity/protocols/oauth2/scopes)
- [Проверка OAuth-приложений](https://support.google.com/cloud/answer/13463073)
- [Включить YouTube Data API в Cloud Console](https://console.cloud.google.com/apis/library/youtube.googleapis.com)
- Клиентские библиотеки: [Python](https://github.com/googleapis/google-api-python-client) · [Node.js](https://github.com/googleapis/google-api-nodejs-client)

### MCP

- [Что такое MCP](https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro) · [Архитектура](https://modelcontextprotocol.io/docs/2026-07-28/learn/architecture) · [Серверы](https://modelcontextprotocol.io/docs/2026-07-28/learn/server-concepts)
- [Build an MCP server](https://modelcontextprotocol.io/docs/2026-07-28/develop/build-server) · [Подключение локальных серверов](https://modelcontextprotocol.io/docs/2026-07-28/develop/connect-local-servers) · [SDK](https://modelcontextprotocol.io/docs/2026-07-28/sdk)
- Спецификация 2026-07-28: [обзор](https://modelcontextprotocol.io/specification/2026-07-28) · [Tools](https://modelcontextprotocol.io/specification/2026-07-28/server/tools) · [Elicitation](https://modelcontextprotocol.io/specification/2026-07-28/client/elicitation)
- Безопасность: [Security Best Practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices) · [Local Server Security](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/local-server-security) · [Authorization](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/authorization)
- Инструменты: [MCP Inspector](https://modelcontextprotocol.io/docs/2026-07-28/tools/inspector) · SDK: [Python](https://github.com/modelcontextprotocol/python-sdk) · [TypeScript](https://github.com/modelcontextprotocol/typescript-sdk)
