# Portretto — запуск на Netlify

Ключи fal.ai и Stripe хранятся только на сервере Netlify (в переменных окружения).
Клиенты видят только сайт: покупают пакет портретов через Stripe и тратят их.

```
public/index.html            — сайт
netlify/functions/api.mjs    — сервер: /api/* (генерация, баланс, оплата, webhook)
netlify/lib/core.mjs         — общая логика (настройки, хранилище, Stripe, fal.ai)
netlify.toml, package.json   — настройки Netlify
```

> Важно: перетаскивание папки в Netlify (Netlify Drop) **не запускает серверные функции**.
> Нужен деплой через GitHub — ниже всё делается мышкой, без терминала.

---

## 1. Загрузить проект на GitHub

1. Зайди на github.com → **New repository** → название, например `portretto` → **Private** → **Create repository**.
2. На странице репозитория нажми **uploading an existing file**.
3. Распакуй архив и перетащи **содержимое** папки `portretto-netlify` (папки `public`, `netlify` и файлы `netlify.toml`, `package.json`, `README.md`) → **Commit changes**.

## 2. Подключить к Netlify

1. app.netlify.com → **Add new project → Import an existing project → GitHub** → выбери репозиторий.
2. Настройки сборки менять не нужно (они в `netlify.toml`) → **Deploy**.
3. Адрес сайта будет вида `https://имя.netlify.app`. Его можно поменять или подключить свой домен.

## 3. Stripe

1. dashboard.stripe.com → **Developers → API keys** → скопируй **Secret key** (сначала тестовый `sk_test_…`).
2. **Developers → Webhooks → Add endpoint**:
   - URL: `https://ТВОЙ-САЙТ/api/stripe-webhook`
   - события: `checkout.session.completed` и `checkout.session.async_payment_succeeded`
   - после создания скопируй **Signing secret** (`whsec_…`).

## 4. Переменные окружения в Netlify

**Project configuration → Environment variables → Add a variable** (область — все или минимум *Functions*):

| Переменная | Обязательно | Что это |
|---|---|---|
| `FAL_KEY` | да | ключ fal.ai (fal.ai/dashboard/keys) |
| `STRIPE_SECRET_KEY` | да | `sk_test_…` для теста, потом `sk_live_…` |
| `STRIPE_WEBHOOK_SECRET` | да | `whsec_…` из шага 3 |
| `CURRENCY` | нет | валюта, по умолчанию `eur` |
| `PACKAGES` | нет | пакеты, см. ниже |
| `FREE_PORTRAITS` | нет | бесплатных портретов новому клиенту, по умолчанию `0` |
| `MODEL` | нет | `both` (Nano Banana Pro + FLUX 2 Pro, по умолчанию), `fal-ai/nano-banana-pro/edit`, `fal-ai/flux-2-pro/edit`, `fal-ai/nano-banana-2/edit` |
| `VARIANTS` | нет | вариантов за попытку, 1–4, по умолчанию `2` |
| `ROUNDS` | нет | максимум попыток до цели сходства, 1–5, по умолчанию `3` |
| `TARGET` | нет | цель сходства лица в %, по умолчанию `95` |
| `RESOLUTION` | нет | `1K`, `2K` (по умолчанию) или `4K` |
| `HD` | нет | `1` — HD-кожа при сохранении (по умолчанию), `0` — выкл. |
| `JOB_MAX_GENERATIONS` | нет | потолок генераций на 1 портрет, по умолчанию `VARIANTS × ROUNDS + 2` |
| `ADMIN_TOKEN` | нет | длинный секрет для ручного начисления портретов |
| `SITE_URL` | нет | адрес сайта для возврата после оплаты (обычно не нужен) |

После добавления или изменения переменных: **Deploys → Trigger deploy → Deploy project**.

**Пакеты** (цена в центах):

```json
[{"id":"p1","portraits":1,"price":399},{"id":"p5","portraits":5,"price":1499},{"id":"p15","portraits":15,"price":3499}]
```

## 5. Проверка

1. Открой сайт → нажми на баланс в шапке → выбери пакет.
2. Тестовая карта Stripe: `4242 4242 4242 4242`, любая будущая дата, любой CVC.
3. После оплаты на сайте появится «Оплата прошла ✨ +5 на балансе».
4. Всё работает → замени `STRIPE_SECRET_KEY` на `sk_live_…`, создай такой же webhook в live-режиме, обнови `STRIPE_WEBHOOK_SECRET`, сделай **Trigger deploy**.

## Как считается баланс

- 1 портрет = одно создание или одна правка, включая автоповторы до цели сходства, перевод запроса и HD-сохранение.
- Если не получилось ни одного результата — портрет автоматически возвращается.
- Баланс привязан к ID клиента в браузере. Перенести на другое устройство: баланс → «ID аккаунта» → скопировать и вставить там.
- Оплата зачисляется ровно один раз (и по webhook, и при возврате на сайт — без дублей).

## Начислить портреты вручную

Клиент присылает свой ID (баланс → «ID аккаунта»). Задай `ADMIN_TOKEN` и выполни:

```bash
curl -X POST https://ТВОЙ-САЙТ/api/admin/grant \
  -H "content-type: application/json" -H "x-admin-token: ТВОЙ_ADMIN_TOKEN" \
  -d '{"cid":"ID_КЛИЕНТА","add":5}'
```

Где данные: Netlify → проект → **Blobs** (хранилище `portretto`: `users/…` — балансы, `sessions/…` — оплаты, `jobs/…` — обработки).
