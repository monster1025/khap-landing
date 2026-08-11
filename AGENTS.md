# AGENTS.md — khap-landing

Статический продающий лендинг «Королевский Хап» — единственная HTML-страница, отдаётся на корне `https://khap.ru/` (главная страница домена). Презентует продукт (Telegram-бот, находящий ценовые ошибки и скидки на Wildberries), реальные тарифы подписки и ведёт пользователя оформить подписку в боте «Стражник» (`https://t.me/king_gate_keeper_bot`).

## Стек

- Один файл `index.html` — вся страница: инлайновые `<style>`/`<script>`, SVG-иконки, без внешних хотлинков на картинки. Единственная внешняя зависимость — шрифты Google Fonts (Cinzel + Manrope).
- **Docker**: `Dockerfile` — `FROM nginx:alpine`, копирует `index.html` в `/usr/share/nginx/html/`. Без стадии сборки — контент готов как есть.
- **docker-compose** — прод-запуск: сервис `khap-landing`, образ `gitea.yandex5.ru/monster/khap-landing:master`, порт хоста `5060` → контейнера `80`.

## Точки входа

- Локально: открыть `index.html` прямо в браузере, либо `docker build -t khap-landing:test . && docker run --rm -p 5060:80 khap-landing:test` → `http://localhost:5060/`.
- Прод: Docker-образ по `Dockerfile`, поднимается сервисом `khap-landing` в `docker-compose.yaml` (порт `5060:80`).
- CI/CD: `.github/workflows/docker-image.yml` — на push в `master` собирает и пушит образ в `gitea.yandex5.ru/monster/khap-landing`, затем по SSH деплоит на 192.168.1.41 (папка `docker.services/khap-landing`).

## Конфигурация

- Переменные окружения не требуются — сайт полностью статический, без бэкенда и без обращений к MongoDB в проде.
- Тарифы (`7/14/30/60 дней`, цены, описание) зашиты в `index.html` как снапшот коллекции `subscription.fare` (MongoDB `king-mongo`) на момент публикации. Источник правды по тарифам — сама БД; при изменении тарифов карточки в `index.html` нужно обновить вручную.
- Ссылка на бота (`https://t.me/king_gate_keeper_bot`) захардкожена во всех CTA-кнопках `index.html`.
- Внешняя маршрутизация: nginx на прод-хосте 192.168.1.41 отдаёт этот сервис по точному `location = /` для домена `khap.ru` (не catch-all!) — конфиг nginx живёт вне репозитория, на хосте (`docker.services/nginx/nginx/settings/site-confs/khap.ru.conf`).

## Структура директорий

```
khap-landing/
├── index.html                          # вся страница лендинга
├── Dockerfile                          # nginx:alpine + COPY index.html
├── docker-compose.yaml                 # сервис khap-landing, порт 5060:80
├── .github/workflows/docker-image.yml  # CI: сборка образа + SSH-деплой на 192.168.1.41
├── .gitignore
├── README.md
└── AGENTS.md
```

## Паттерны добавления функциональности

- Правки контента/дизайна — прямо в `index.html` (секции: hero, "как это работает", возможности, тарифы, соц.доказательство, FAQ, футер).
- Обновление тарифов — синхронизировать секцию `#pricing` в `index.html` с актуальными данными коллекции `subscription.fare` в MongoDB (`king-mongo`, 192.168.1.43, база `subscription`).

## Известные особенности

- **Важно при изменении nginx-маршрутизации**: `location = /` на прод-хосте — точное совпадение по пути запроса, без учёта query-строки. Реальные return/callback-URL платёжных провайдеров (ЮMoney/ЮKassa/Qiwi, обрабатываются `bot/gatekeeper`/`bot/gatekeeper-payment`) используют только префикс `/payments/...`, поэтому этот сервис не пересекается с платёжными флоу. Не менять этот `location` на нестрогий/regex-catch-all — это заберёт себе трафик `/payments/*`, `/success.html`, `/donate.html` и т.п., которые должны оставаться на `gatekeeper-payment`.
- Юзернейм бота "Стражник" в других частях платформы (FAQ, `bot/wbproductinfo`, и др.) местами указан как `@king_gatekeeper_bot` (без второго подчёркивания) — это расхождение с реально используемым в этом лендинге и в `Gatekeeper.Shared/Constants.cs` `@king_gate_keeper_bot`. Проверено вручную в Telegram: именно `@king_gate_keeper_bot` — живой бот "Королевский Стражник" с фото и описанием.
