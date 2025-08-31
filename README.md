# GiftBuyer

Веб-панель и бот для **мониторинга и автопокупки Telegram-подарков** на базе **Flask + SQLAlchemy + Pyrogram** (backend) и **React + Vite** (frontend).
Поддерживается работа с несколькими аккаунтами, просмотр баланса звёзд, список доступных подарков с предпросмотром **.tgs (Lottie)**, хранилище каналов с фильтрами (по цене и supply), отправка уведомлений в каналы/ЛС и **автопокупка лимитных подарков на нужные каналы**.

---

## Ключевые возможности

* 🔐 Регистрация/логин (httpOnly cookie, 7 дней).
* 👤 Несколько **API-профилей** (api\_id/api\_hash) и **несколько Telegram-аккаунтов** на пользователя.
* ⭐ Просмотр Premium, **баланс звёзд**, принудительное обновление данных аккаунтов со стримом стадий (NDJSON).
* 🎁 **Подарки (Gifts)**: список, ручное/фоновое обновление, SSE-стрим, предпросмотр .tgs (серверный кэш).
* 📡 **Каналы**: добавление по `channel_id` (формат `-100…`), проверка членства, хранение названия и фильтров **цена/supply**, CRUD.
* 🤖 **Автопокупка** лимитных подарков на каналы по заданным фильтрам, с отчётами в ЛС и подробными причинами, если не купилось.
* 🔔 **Уведомления** о новых подарках в указанные каналы/ЛС (со стикером и кратким текстом).
* 🧵 Бэкграунд-воркер подарков, устойчивый к падениям и переподключениям, с «горячим» перечитыванием аккаунтов.
---

## Как работает автопокупка (кратко про алгоритм)

Файл: `backend/services/autobuy_service.py`

1. **Сбор входных данных**

   * Берём **все аккаунты** пользователя и узнаём их **баланс звёзд**.
   * Берём **все каналы** пользователя и их фильтры:

     * диапазон `price_min..price_max`;
     * диапазон `supply_min..supply_max` (если `null` — трактуется как «без ограничения»).
   * Берём **новые подарки**, пришедшие от воркера.
     ⚠️ В боевом режиме покупаем **только лимитные** (`is_limited=True`). Нелимитные сразу «скипаем» с причиной.

2. **Фильтрация и приоритезация подарков**

   * Отсекаем «мусор» (некорректные `id/price/supply`).
   * Сортируем лимитки по ключу: **меньше supply → дороже → случайность**, чтобы сначала ловить самые дефицитные и ценные.

3. **Планирование («план покупок»)**
   Для **каждого аккаунта** (в порядке убывания баланса) распределяем покупки:

   * Для конкретного подарка ищется **лучший подходящий канал** (по фильтрам цены/supply).
   * Рассчитывается максимально возможное количество отправок с учётом **остатка баланса** аккаунта и **доступности** подарка.
   * В план добавляются реальные операции: `(account_id, channel_id, gift_id, price, supply)`.
   * Если на этапе планирования что-то не сошлось (нет подходящих каналов, не хватило звёзд и пр.), это фиксируется в `plan_skips`.

4. **Исполнение плана**

   * Идём по плану и на каждой операции вызываем:

     ```
     await tg_client.send_gift(chat_id=<ID канала>, gift_id=<ID подарка>)
     ```
   * Успех/неуспех детально логируются в статистике **по каналам** и **по аккаунтам**.

5. **Отчёты**

   * Формируется отчёт: общая сводка, по подаркам (✅/⏭️/❌), по каналам и по аккаунтам.
   * Отчёт уходит в **ЛС всех ваших аккаунтов** (через ваш Bot Token из «Настроек»).

6. **Параллельно** воркер шлёт **уведомления** о новых подарках в каналы/ЛС.
   В `gifts_service` покупка и уведомления запускаются одновременно (`asyncio.gather`), покупка **не блокирует** уведомления.

---

## Архитектура и структура проекта

```
GiftBuyer/
  backend/
    app.py
    auth.py
    db.py
    logger.py
    models.py
    requirements.txt
    migrate_app_db.py
    sessions/                 # .session (gitignored)
    instance/
      gifts_cache/            # .tgs gzip-кэш (gitignored)
    routes/
      __init__.py
      auth.py
      account.py
      misc.py
      gifts.py                # API подарков + Lottie-кэш + SSE
      settings.py             # API настроек пользователя (Bot token)
      channels.py             # API каналов (CRUD)
    services/
      __init__.py
      accounts_service.py
      gifts_service.py        # воркер подарков, SSE-шина, parallel buy+notify
      settings_service.py
      notify_gifts_service.py # отправка стикера/текста в каналы/ЛС
      session_locks_service.py
      tg_clients_service.py   # единый Pyrogram-клиент + tg_call/tg_shutdown
      channels_service.py     # нормализация channel_id, probe, валидация
      pyro_login.py           # добавление аккаунтов (в т.ч. Premium)
      autobuy_service.py      # планировщик и исполнитель автопокупки + отчёты в ЛС
  frontend/
    index.html
    package.json
    package-lock.json
    src/
      api.js                  # fetch-обёртка + 401
      App.jsx
      auth.js
      main.jsx
      notify.js
      styles.css
      utils/
        openCentered.js
      ui/
        ModalStack.jsx
      pages/
        Dashboard.jsx
        GiftsPage.jsx
        SettingsPage.jsx
        Login.jsx
        Register.jsx
      components/
        AccountList.jsx
        AddApiModal.jsx
        AddAccountModal.jsx
        SelectApiProfileModal.jsx
        ConfirmModal.jsx
        EditChannelModal.jsx
```

> `backend/sessions/`, `backend/instance/` и их содержимое игнорируются в Git.

---

## Требования

* **Python** 3.11+
* **Node** 18+ / npm 9+
* Рабочий **api\_id/api\_hash** (my.telegram.org → API development tools)
* Ваш **Bot Token** (BotFather), чтобы слать .tgs-превью и отчёты в ЛС
* Чаты, куда бот имеет право писать (если хотите уведомления в каналы)

---

## Установка и запуск (локально)

### Backend

```bash
cd backend
python -m venv .venv
# Windows:
.\.venv\Scripts\activate
# macOS/Linux:
# source .venv/bin/activate

pip install -r requirements.txt

# (опционально) миграция/инициализация БД,
# если у вас есть перенос схемы:
python -m backend.migrate_app_db

# запуск dev-сервера (Flask)
python -m backend.app     # http://localhost:5000
```

**Переменные окружения (необязательно):**

* `SECRET_KEY` — секрет Flask.
* `GIFTS_DIR` — каталог с данными подарков (по умолчанию `gifts_data`).
* `GIFTS_CACHE_DIR` — кэш .tgs (по умолчанию `backend/instance/gifts_cache`).
* `GIFTS_ACCS_TTL` — период (сек) переобновления списка аккаунтов воркером (по умолчанию `60`).

### Frontend

```bash
cd frontend
npm i
npm run dev     # http://localhost:5173
```

При необходимости настройте прокси `/api` → `http://localhost:5000` во `vite.config.js`.

---

## Быстрый старт в UI

1. Откройте `http://localhost:5173`, зарегистрируйтесь/войдите.
2. Создайте **API-профиль** (api\_id/api\_hash).
3. Добавьте **Telegram-аккаунт** (телефон → код → при необходимости пароль).
4. Зайдите на «**Подарки**»: посмотрите список/предпросмотр .tgs.
5. В «**Настройках**» сохраните **Bot Token** — без него не будет стикеров/отчётов.
6. В «**Каналах**» добавляйте `channel_id` вида `-100…`, задавайте **диапазоны цены и supply**.
7. Включите **автопокупку** (поле `gifts_autorefresh` у пользователя; в UI или напрямую в БД).
   Воркер начнёт мониторить новые подарки и, если они **лимитные** и попадают в фильтры — будет **покупать на каналы**.

---

## Автопокупка — правила и приоритеты

* Покупаем **только лимитные** подарки (`is_limited=True`). Нелимитные фиксируются как «пропущены» с причиной.
* Для каждого подарка подбирается **лучший канал** среди подходящих по фильтрам `price` и `supply`:

  * Если `supply_min/max` у канала `null` — считаем «без ограничения».
  * Чем **уже** «окно» по supply и чем **выше** `price_max`, тем **выше приоритет** канала.
* План распределяется **по всем аккаунтам** (сортируются по убыванию баланса):

  * На каждый аккаунт ставится столько покупок, сколько позволяет **баланс** и **доступность** подарка.
* Отправка происходит именно **в канал**: `send_gift(chat_id=<ID канала>, gift_id=...)`.
* Подробные причины, почему не купили:

  * `no_channel_match` — не нашлось каналов, где и `price`, и `supply` укладываются в фильтры.
  * `not_enough_stars` / `insufficient_account_balance` — не хватило звёзд.
  * `send_gift_failed` — сам вызов `send_gift` провалился (нет прав писать в канал, не член канала, gift больше недоступен, требуются Premium, и т.п.).
  * `invalid/price/supply` — подарок не прошёл базовую валидацию.
* В ЛС уходит **человечный отчёт** с эмодзи: сводка, по подаркам (что и куда купили, либо почему пропущено), по каналам и по аккаунтам.

---

## Деплой (production)

> В dev-режиме Flask подходит для разработки, **не** для продакшена. Ниже два варианта: **systemd + Gunicorn** и **Docker Compose**.

### Вариант A — systemd + Gunicorn (Linux)

**1) Подготовка окружения**

```bash
# корень репо
python -m venv .venv
source .venv/bin/activate
pip install -r backend/requirements.txt
cd frontend && npm ci && npm run build && cd ..
```

Раздавать `frontend/dist/` можно любым веб-сервером (nginx/caddy), а `/api` проксировать на backend.

**2) WSGI-сервер**

Файл `wsgi.py` (в `backend/`), если его нет:

```python
from backend.app import create_app
app = create_app()
```

**3) Gunicorn запуск**
Так как внутри backend есть асинхронщина + фоновые потоки, удерживаем **1 воркер**, но даём **многопоточность**:

```bash
cd backend
../.venv/bin/gunicorn wsgi:app \
  --bind 0.0.0.0:5000 \
  --workers 1 \
  --threads 8 \
  --timeout 120
```

> На Windows вместо Gunicorn используйте **Waitress**:
>
> ```bash
> pip install waitress
> waitress-serve --listen=0.0.0.0:5000 backend.wsgi:app
> ```

**4) systemd-юнит**

`/etc/systemd/system/giftbuyer.service`:

```ini
[Unit]
Description=GiftBuyer backend
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/opt/GiftBuyer/backend
Environment="SECRET_KEY=change-me"
Environment="GIFTS_DIR=/opt/GiftBuyer/gifts_data"
Environment="GIFTS_CACHE_DIR=/opt/GiftBuyer/backend/instance/gifts_cache"
Environment="GIFTS_ACCS_TTL=60"
ExecStart=/opt/GiftBuyer/.venv/bin/gunicorn wsgi:app --bind 0.0.0.0:5000 --workers 1 --threads 8 --timeout 120
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now giftbuyer
```

**5) Nginx (пример)**

```nginx
server {
  listen 80;
  server_name your.domain;

  # статика фронта
  root /opt/GiftBuyer/frontend/dist;

  location / {
    try_files $uri /index.html;
  }

  # API
  location /api/ {
    proxy_pass http://127.0.0.1:5000/api/;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
  }

  # SSE (если используете)
  location /api/gifts/stream {
    proxy_pass http://127.0.0.1:5000/api/gifts/stream;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Cache-Control "no-cache";
  }
}
```

### Вариант B — Docker Compose

`docker-compose.yml` (минимальный пример):

```yaml
version: "3.8"
services:
  backend:
    build: ./backend
    environment:
      SECRET_KEY: "change-me"
      GIFTS_DIR: /data/gifts_data
      GIFTS_CACHE_DIR: /app/instance/gifts_cache
      GIFTS_ACCS_TTL: "60"
    volumes:
      - ./backend/sessions:/app/sessions
      - ./data:/data
    ports:
      - "5000:5000"
    restart: unless-stopped

  frontend:
    build: ./frontend
    ports:
      - "5173:80"
    restart: unless-stopped
```

> Внутри контейнера backend можно использовать gunicorn как в варианте A (Dockerfile подправьте соответствующим образом).
> Не забудьте смонтировать `sessions/` и `gifts_cache/`/`gifts_data` на хост для персистентности.

---

## Настройки и важные поля

* **User.gifts\_autorefresh** — вкл/выкл автопокупку.
* **UserSettings.bot\_token** — используется для отправки .tgs-стикеров и текстов в каналы/ЛС (уведомления и отчёты).
* **Channel**:

  * `channel_id` вида `-100…` (обязательно).
  * `price_min` / `price_max` — в звёздах.
  * `supply_min` / `supply_max` — общее издание (если `null`, ограничения нет).
* **Account**:

  * привязан к API-профилю;
  * в UI виден баланс звёзд; баланс запрашивается при планировании.

---

## Отчёты и логи

* В **консоли**: подробно логируется план/исполнение, причины ошибок по каналам/аккаунтам, глобальные пропуски.
* В **ЛС (всем вашим аккаунтам)**:

```
🧾 Отчёт автопокупки
📊 Новых: 3 | Куплено: 2 | Пропущено: 1

📦 По подаркам:
• ✅ 123456 | 25⭐ | supply=1000 → ch=-100111 acc=1337
• ⏭️ 654321 | 50⭐ | supply=∞ → non-limited (skipped)
• ❌ 222333 | 10⭐ | supply=500 → причина ch=-100222: insufficient_account_balance acc=1338 bal=5 need=10

🛰️ По каналам:
• -100111: plan=1 ok=1 fail=0 reasons=0
• -100222: plan=1 ok=0 fail=0 reasons=1

👤 По аккаунтам:
• acc=1337: plan=1 💰spent=25 start=100 end=75 buys=1
• acc=1338: plan=1 💰spent=0 start=5 end=5 buys=0
```

---

## Частые проблемы и решения

* **`send_gift_failed`**
  Проверьте:

  * аккаунт **состоит** в канале;
  * у бота есть право писать (для уведомлений), и **аккаунт** имеет право отправлять gift;
  * gift уже не закончился/не изменился;
  * gift **не требует Premium**, если у аккаунта его нет;
  * баланс достаточен.

* **Нет покупок, «no\_channel\_match»**
  Подарок не попал в рамки ни одного канала. Проверьте `price_min/max` и `supply_min/max`.

* **Ничего не видно в «Подарках»**
  Нажмите ручное обновление или дождитесь воркера. Проверьте токен бота (для предпросмотра .tgs).

* **Windows продакшен**
  Gunicorn недоступен. Используйте `waitress-serve` или разворачивайте на Linux-сервере.

---

## Безопасность

* Храним токен в httpOnly-cookie.
* Сессии Pyrogram лежат в `backend/sessions/` (держите приватно).
* Бот-токен хранится в БД в таблице настроек; ограничьте доступ к БД и серверу.

---

## Лицензия

Apache-2.0 © 2025 Vova Orig. См. LICENSE и NOTICE.

---
