# PingWatch — Полная инструкция по запуску

Пошаговое руководство: от нулевой конфигурации до работающего сайта с уведомлениями.

---

## Что нужно заранее

| Сервис | Для чего | Стоимость |
|---|---|---|
| [Supabase](https://supabase.com) | База данных + аутентификация | Бесплатно |
| [Netlify](https://netlify.com) | Хостинг + scheduled функции | Бесплатно |
| Telegram (опционально) | Уведомления в Telegram | Бесплатно |
| Discord (опционально) | Уведомления в Discord | Бесплатно |
| Gmail (опционально) | Email-уведомления | Бесплатно |

---

## Шаг 1. Supabase — создать проект

1. Зайдите на [supabase.com](https://supabase.com) → **Start your project**
2. Создайте новый проект, задайте имя и пароль базы данных (сохраните пароль)
3. Дождитесь запуска проекта (около 1 минуты)
4. Перейдите: **Project Settings → API**
5. Скопируйте и сохраните два значения:
   - **Project URL** → `SUPABASE_URL`
   - **anon public** key → `SUPABASE_PUBLISHABLE_KEY`
   - **service_role** key → `SUPABASE_SECRET_KEY` (нажмите `Reveal`, этот ключ секретный)

---

## Шаг 2. Supabase — создать таблицы

Откройте: **SQL Editor** (левое меню) → нажмите **New query** → вставьте SQL ниже → нажмите **Run**.

```sql
-- Таблица пользователей
CREATE TABLE IF NOT EXISTS public.users (
  id                  uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email               text NOT NULL,
  telegram_chat_id    text,
  notification_channel text DEFAULT 'telegram',
  discord_webhook_url text,
  notification_email  text,
  created_at          timestamptz DEFAULT now()
);

-- Таблица мониторов
CREATE TABLE IF NOT EXISTS public.monitors (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid REFERENCES public.users(id) ON DELETE CASCADE,
  url             text NOT NULL,
  name            text NOT NULL,
  is_active       boolean NOT NULL DEFAULT true,
  last_status     text,
  last_checked_at timestamptz,
  created_at      timestamptz DEFAULT now()
);

-- Таблица инцидентов
CREATE TABLE IF NOT EXISTS public.incidents (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  monitor_id       uuid REFERENCES public.monitors(id) ON DELETE CASCADE,
  started_at       timestamptz NOT NULL,
  resolved_at      timestamptz,
  duration_seconds integer,
  created_at       timestamptz DEFAULT now()
);
```

---

## Шаг 3. Supabase — настроить Row Level Security (RLS)

Запустите ещё один SQL-запрос, чтобы каждый пользователь видел только свои данные:

```sql
-- RLS для users
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users: own row" ON public.users
  FOR ALL USING (auth.uid() = id);

-- RLS для monitors
ALTER TABLE public.monitors ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Monitors: own rows" ON public.monitors
  FOR ALL USING (auth.uid() = user_id);

-- RLS для incidents (доступ через monitors)
ALTER TABLE public.incidents ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Incidents: via own monitors" ON public.incidents
  FOR ALL USING (
    monitor_id IN (
      SELECT id FROM public.monitors WHERE user_id = auth.uid()
    )
  );
```

---

## Шаг 4. Supabase — разрешить доступ service_role к users через joined-запросы

Функция Netlify использует `service_role` (обходит RLS), поэтому дополнительных прав не нужно. Но нужно убедиться, что в таблице `users` поле `id` совпадает с `auth.users.id`. Это уже настроено в SQL выше.

---

## Шаг 5. Подставить ваши ключи в HTML-файлы

Откройте **`index.html`** и **`dashboard.html`** в любом текстовом редакторе.

Найдите эти строки (они есть в обоих файлах):

```js
window.SUPABASE_URL = '__SUPABASE_URL__';
window.SUPABASE_KEY = '__SUPABASE_PUBLISHABLE_KEY__';
```

Замените на реальные значения:

```js
window.SUPABASE_URL = 'https://xxxxxxxxxxxx.supabase.co';
window.SUPABASE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';  // anon key
```

> ⚠️ Здесь используется **anon** (публичный) ключ — он безопасен для фронтенда.  
> `service_role` key никогда не вставляйте во фронтенд — только на Netlify.

---

## Шаг 6. Настроить Telegram-бот (если выбрали Telegram)

### 6a. Создать бота
1. Откройте Telegram → найдите **@BotFather**
2. Отправьте `/newbot`
3. Введите имя бота (например, `PingWatch Alerts`) и username (например, `pingwatch_mysite_bot`)
4. Скопируйте **HTTP API token** — это `TELEGRAM_BOT_TOKEN`

### 6b. Получить Chat ID
1. Найдите в Telegram бота **@userinfobot**
2. Отправьте `/start`
3. Он ответит вашим **Id** (числом) — это и есть `TELEGRAM_CHAT_ID`

> Также Chat ID нужно ввести при регистрации на сайте.

---

## Шаг 7. Настроить Discord Webhook (если выбрали Discord)

1. Откройте нужный Discord-сервер
2. Зайдите в **Server Settings → Integrations → Webhooks**
3. Нажмите **New Webhook**
4. Выберите канал, дайте имя (например, `PingWatch`)
5. Нажмите **Copy Webhook URL**
6. Этот URL пользователь вводит при регистрации на сайте

---

## Шаг 8. Настроить Gmail для Email-уведомлений (если выбрали Email)

Gmail требует **App Password** — обычный пароль не работает.

1. Включите двухфакторную аутентификацию на Gmail:  
   [myaccount.google.com/security](https://myaccount.google.com/security) → **2-Step Verification**
2. Перейдите: [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Выберите **Mail** + **Other (Custom name)**, введите `PingWatch`
4. Скопируйте сгенерированный **16-символьный пароль** — это `SMTP_PASSWORD`
5. `SMTP_EMAIL` — ваш Gmail-адрес (`yourname@gmail.com`)

---

## Шаг 9. Деплой на Netlify

### 9a. Создать аккаунт
Зайдите на [netlify.com](https://netlify.com) → Sign up (можно через GitHub или Google).

### 9b. Загрузить проект
**Вариант 1 — перетащить папку:**
1. Откройте [app.netlify.com](https://app.netlify.com)
2. В разделе **Sites** перетащите папку проекта `pingwatch/` прямо на страницу
3. Netlify автоматически задеплоит сайт

**Вариант 2 — через GitHub (рекомендуется для обновлений):**
1. Создайте репозиторий на GitHub, загрузите папку проекта
2. В Netlify: **Add new site → Import an existing project → GitHub**
3. Выберите репозиторий → **Deploy site**

### 9c. Задать переменные окружения
Откройте: **Site Settings → Environment Variables → Add a variable**

Добавьте по одной:

| Ключ | Значение |
|---|---|
| `SUPABASE_URL` | `https://xxxx.supabase.co` |
| `SUPABASE_SECRET_KEY` | `service_role` key из Supabase |
| `TELEGRAM_BOT_TOKEN` | Токен от @BotFather (если Telegram) |
| `TELEGRAM_CHAT_ID` | Ваш Chat ID (резервный, если не задан в профиле) |
| `SMTP_EMAIL` | Gmail-адрес (если Email) |
| `SMTP_PASSWORD` | App Password от Gmail (если Email) |

После добавления переменных — **Deploys → Trigger deploy → Deploy site**, чтобы пересобрать с новыми переменными.

---

## Шаг 10. Проверить работу scheduled функции

1. Откройте: **Netlify → Functions** (левое меню)
2. Должна появиться функция `check-monitors`
3. Чтобы проверить немедленно: нажмите на функцию → **Trigger function** (или подождите 5 минут)
4. В разделе **Logs** увидите вывод вида:
   ```
   [CHECK] "My API" → UP
   [CHECK] "Shop" → DOWN (CHANGED)
   [ALERT] channel=telegram monitor="Shop" status=DOWN
   ```

---

## Шаг 11. Проверить весь флоу

1. Откройте ваш сайт (URL выдаёт Netlify, например `https://magical-llama-123.netlify.app`)
2. Зарегистрируйтесь → выберите канал уведомлений → введите Chat ID / Webhook / Email
3. Добавьте монитор с URL — например, `https://httpstat.us/503` (это всегда возвращает ошибку, удобно для теста)
4. Подождите до 5 минут — должно прийти уведомление **DOWN**
5. Удалите этот монитор → добавьте `https://httpstat.us/200` (всегда UP) — придёт **BACK UP**

---

## Шаг 12. Supabase Auth — настроить Email (опционально)

По умолчанию Supabase требует подтверждения email при регистрации. Чтобы отключить для теста:

1. **Authentication → Providers → Email**
2. Выключить **Confirm email**

Для продакшена — наоборот, оставьте включённым и настройте SMTP в **Project Settings → Auth → SMTP Settings**.

---

## Итог: что должно работать

| Функция | Как проверить |
|---|---|
| Регистрация и вход | Зарегистрируйтесь через форму на сайте |
| Добавление монитора | Кнопка **Add monitor** в дашборде |
| Статусы UP/DOWN | Появляются после первого запуска функции |
| Уведомление DOWN | Добавьте `https://httpstat.us/503` |
| Уведомление RECOVERY | Сайт падает, потом добавьте рабочий URL |
| История инцидентов | Правая колонка дашборда |
| Пауза монитора | Переключатель рядом с монитором |
| Удаление монитора | Кнопка с иконкой корзины |

---

## Частые ошибки

### `relation "users" does not exist`
Вы забыли выполнить SQL из Шага 2. Откройте Supabase → SQL Editor → запустите таблицы.

### Уведомления не приходят
- Проверьте переменные окружения в Netlify (опечатки в ключах)
- Для Telegram: убедитесь, что вы написали боту хотя бы `/start`
- Для Gmail: убедитесь, что используете App Password, а не обычный пароль

### Функция не запускается
- В `netlify.toml` должна быть строка `schedule = "*/5 * * * *"` под `[functions."check-monitors"]`
- Scheduled Functions работают только на задеплоенном сайте, не при локальном запуске

### После деплоя сайт не видит данные
- Убедитесь, что в HTML подставлены правильные `SUPABASE_URL` и `SUPABASE_KEY`
- Ключ на фронтенде — **anon** (не service_role)

### `new row violates row-level security policy`
Запустите SQL из Шага 3 (RLS-политики).
