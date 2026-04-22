# PingWatch — Uptime Monitoring SaaS

Monitors URLs every 5 minutes. Sends alerts via Telegram, Discord, or Email on DOWN and RECOVERY events.

---

## Stack

| Layer | Technology |
|---|---|
| Frontend | Pure HTML + CSS + JS (no framework) |
| Auth & DB | Supabase (CDN, no npm) |
| Scheduler | Netlify Scheduled Functions |
| Alerts | Telegram Bot API / Discord Webhooks / Nodemailer Gmail SMTP |

---

## Deploy steps

### 1. Supabase — add columns to `users`

Run once in **SQL Editor**:

```sql
ALTER TABLE users
  ADD COLUMN IF NOT EXISTS notification_channel text DEFAULT 'telegram',
  ADD COLUMN IF NOT EXISTS discord_webhook_url  text,
  ADD COLUMN IF NOT EXISTS notification_email   text;
```

Make sure the `monitors` and `incidents` tables exist with these schemas:

```sql
-- monitors
CREATE TABLE IF NOT EXISTS monitors (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid REFERENCES auth.users(id) ON DELETE CASCADE,
  url             text NOT NULL,
  name            text NOT NULL,
  is_active       boolean NOT NULL DEFAULT true,
  last_status     text,          -- 'UP' | 'DOWN' | null
  last_checked_at timestamptz,
  created_at      timestamptz DEFAULT now()
);

-- incidents
CREATE TABLE IF NOT EXISTS incidents (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  monitor_id       uuid REFERENCES monitors(id) ON DELETE CASCADE,
  started_at       timestamptz NOT NULL,
  resolved_at      timestamptz,
  duration_seconds integer,
  created_at       timestamptz DEFAULT now()
);
```

Enable **Row Level Security** and add policies so each user can only read/write their own rows.

### 2. Replace placeholder credentials in HTML files

In `index.html` and `dashboard.html`, replace:

```js
window.SUPABASE_URL = '__SUPABASE_URL__';
window.SUPABASE_KEY = '__SUPABASE_PUBLISHABLE_KEY__';
```

with your real values from Supabase → Project Settings → API.

### 3. Netlify environment variables

Set in **Site Settings → Environment Variables**:

| Variable | Value |
|---|---|
| `SUPABASE_URL` | Your Supabase project URL |
| `SUPABASE_SECRET_KEY` | `service_role` key (NOT anon) |
| `TELEGRAM_BOT_TOKEN` | Token from @BotFather |
| `TELEGRAM_CHAT_ID` | Fallback chat ID (optional) |
| `SMTP_EMAIL` | Gmail address for sending alerts |
| `SMTP_PASSWORD` | Gmail App Password |

### 4. Deploy

Drag-and-drop the project folder to Netlify, or push to GitHub and connect via Netlify dashboard.

The function `check-monitors` will run automatically every **5 minutes** via the cron schedule `*/5 * * * *` defined in `netlify.toml`.

---

## Project structure

```
pingwatch/
├── index.html                        # Landing + registration / sign-in
├── dashboard.html                    # User dashboard
├── assets/
│   ├── style.css                     # Design system + all styles
│   └── app.js                        # Supabase client, auth helpers, utilities
├── netlify/
│   └── functions/
│       ├── check-monitors.js         # Scheduled ping + alert function
│       └── package.json              # nodemailer + @supabase/supabase-js
├── netlify.toml                      # Cron schedule + build config
└── README.md
```

---

## Alert messages

**DOWN alert:**
```
🚨 My API is 🔴 DOWN
URL: https://api.example.com
Time: Wed, 22 Apr 2026 18:05:00 GMT
```

**Recovery alert:**
```
✅ My API is 🟢 BACK UP
URL: https://api.example.com
Time: Wed, 22 Apr 2026 18:35:00 GMT
```
