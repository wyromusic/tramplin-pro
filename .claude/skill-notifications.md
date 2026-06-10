---
name: tramplin-notifications
description: SMS и Push уведомления для tramplin.pro. Читай этот файл когда нужно: отправить SMS через smsc.ru, настроить Web Push без Firebase, запланировать напоминания перед уроком, реализовать подписку на push-уведомления.
---

# Skill: Уведомления (SMS + Web Push)

Всегда читай `AGENT.md` перед этим файлом.

---

## Провайдеры — только российские или независимые

| Тип | Провайдер | Переменная окружения | Почему |
|-----|-----------|---------------------|--------|
| SMS | **smsc.ru** (СМС Центр) | `SMSC_LOGIN`, `SMSC_PASSWORD` | Российский, стабильный, дешёвый |
| Push | **Web Push (VAPID)** — свой сервер | `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_EMAIL` | Не зависит от Google/Firebase |

❌ **НЕ использовать:** Twilio (заблокирован), Firebase Cloud Messaging (нестабилен в РФ), OneSignal (иностранный).

---

## SMS через smsc.ru

### Настройка аккаунта smsc.ru
1. Зарегистрироваться на smsc.ru
2. Пополнить баланс
3. Зарегистрировать имя отправителя `TRAMPLIN` (занимает ~3 дня, нужен договор)
4. До регистрации имени — отправлять от имени smsc.ru (работает без регистрации)

### Отправка SMS

```typescript
// src/lib/notifications/sms.ts

export interface SMSResult {
  success: boolean
  messageId?: string
  error?: string
}

export async function sendSMS(phone: string, message: string): Promise<SMSResult> {
  // Нормализуем номер: +79991234567 → 79991234567
  const normalizedPhone = phone.replace(/\D/g, '').replace(/^8/, '7')
  
  if (!/^7\d{10}$/.test(normalizedPhone)) {
    return { success: false, error: 'Неверный формат номера' }
  }

  const params = new URLSearchParams({
    login: process.env.SMSC_LOGIN!,
    psw: process.env.SMSC_PASSWORD!,
    phones: normalizedPhone,
    mes: message,
    fmt: '3',           // JSON ответ
    charset: 'utf-8',
    sender: process.env.SMSC_SENDER ?? 'TRAMPLIN',
    cost: '0',          // Не запрашивать стоимость
  })

  try {
    const res = await fetch(`https://smsc.ru/sys/send.php?${params}`, {
      signal: AbortSignal.timeout(10000), // 10 секунд таймаут
    })

    if (!res.ok) {
      return { success: false, error: `HTTP ${res.status}` }
    }

    const data = await res.json()

    if (data.error) {
      console.error(`[SMS] Ошибка для ${normalizedPhone}: ${data.error} (код ${data.error_code})`)
      return { success: false, error: data.error }
    }

    console.log(`[SMS] Отправлено на ${normalizedPhone}, ID: ${data.id}`)
    return { success: true, messageId: String(data.id) }

  } catch (e) {
    const msg = e instanceof Error ? e.message : 'Неизвестная ошибка'
    console.error(`[SMS] Сетевая ошибка:`, msg)
    return { success: false, error: msg }
  }
}
```

### Проверка баланса smsc.ru (для мониторинга)

```typescript
export async function getSMSBalance(): Promise<number | null> {
  const params = new URLSearchParams({
    login: process.env.SMSC_LOGIN!,
    psw: process.env.SMSC_PASSWORD!,
    fmt: '3',
    cur: '1',
  })
  
  try {
    const res = await fetch(`https://smsc.ru/sys/balance.php?${params}`)
    const data = await res.json()
    return data.balance ?? null
  } catch {
    return null
  }
}
```

---

## Web Push — без Firebase, через VAPID

### Генерация VAPID ключей (один раз)

```bash
npx web-push generate-vapid-keys
# Сохрани оба ключа в .env.local!
```

### Таблица подписок

```sql
create table public.push_subscriptions (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references public.profiles(id) on delete cascade,
  endpoint    text not null,
  p256dh      text not null,
  auth        text not null,
  user_agent  text,
  created_at  timestamptz default now(),
  unique(user_id, endpoint)
);
```

### Клиент: подписка на push

```typescript
// src/lib/notifications/push-client.ts
'use client'

function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4)
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/')
  const rawData = atob(base64)
  return Uint8Array.from([...rawData].map((c) => c.charCodeAt(0)))
}

export async function subscribeToPush(): Promise<boolean> {
  if (!('serviceWorker' in navigator) || !('PushManager' in window)) return false

  const permission = await Notification.requestPermission()
  if (permission !== 'granted') return false

  try {
    const reg = await navigator.serviceWorker.ready
    const subscription = await reg.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(
        process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
      ),
    })

    await fetch('/api/notifications/subscribe', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(subscription.toJSON()),
    })

    return true
  } catch (e) {
    console.error('[Push] Ошибка подписки:', e)
    return false
  }
}
```

### Сервер: отправка Push

```typescript
// src/lib/notifications/push-server.ts
import webpush from 'web-push'

webpush.setVapidDetails(
  `mailto:${process.env.VAPID_EMAIL}`,
  process.env.VAPID_PUBLIC_KEY!,
  process.env.VAPID_PRIVATE_KEY!
)

export async function sendPushToUser(
  userId: string,
  payload: { title: string; body: string; url?: string }
): Promise<void> {
  const supabase = createAdminClient()
  
  const { data: subs } = await supabase
    .from('push_subscriptions')
    .select('*')
    .eq('user_id', userId)

  if (!subs?.length) return

  await Promise.allSettled(
    subs.map(async (sub) => {
      try {
        await webpush.sendNotification(
          { endpoint: sub.endpoint, keys: { p256dh: sub.p256dh, auth: sub.auth } },
          JSON.stringify(payload)
        )
      } catch (err: any) {
        if (err.statusCode === 410) {
          // Подписка устарела — удаляем
          await supabase.from('push_subscriptions').delete().eq('id', sub.id)
        }
      }
    })
  )
}
```

### Service Worker (public/sw.js)

```javascript
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {}
  event.waitUntil(
    self.registration.showNotification(data.title ?? 'Трамплин', {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-72.png',
      data: { url: data.url ?? '/' },
    })
  )
})

self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  event.waitUntil(clients.openWindow(event.notification.data.url))
})
```

---

## Шаблоны уведомлений

```typescript
// src/lib/notifications/templates.ts

export const templates = {
  booking_confirmed: (lesson: string, date: string, time: string) =>
    `✅ Запись подтверждена: ${lesson} ${date} в ${time}. tramplin.pro`,

  reminder_24h: (lesson: string, time: string) =>
    `⏰ Завтра: ${lesson} в ${time}. Ждём вас! tramplin.pro`,

  reminder_2h: (lesson: string, time: string) =>
    `⏰ Через 2 часа: ${lesson} в ${time}. tramplin.pro`,

  booking_cancelled: (lesson: string, date: string) =>
    `❌ Запись на ${lesson} ${date} отменена. Вопросы: tramplin.pro`,

  lesson_transferred: (lesson: string, oldDate: string, newDate: string) =>
    `📅 Перенос: ${lesson} с ${oldDate} на ${newDate}. tramplin.pro`,

  otp_code: (code: string) =>
    `Ваш код для входа на tramplin.pro: ${code}. Действует 5 минут.`,

  welcome: (name?: string) =>
    `Добро пожаловать${name ? `, ${name}` : ''}! Вы зарегистрированы на tramplin.pro`,
}
```

---

## Планировщик напоминаний

Запускается после создания записи. Cron проверяет каждые 15 минут.

```typescript
// src/lib/notifications/scheduler.ts
export async function scheduleReminders(bookingId: string) {
  const supabase = createAdminClient()
  
  const { data: booking } = await supabase
    .from('bookings')
    .select('student_id, slot:slots(starts_at, lesson_type:lesson_types(name))')
    .eq('id', bookingId)
    .single()

  if (!booking) return

  const startsAt = new Date(booking.slot.starts_at)
  const lessonName = booking.slot.lesson_type.name
  const time = startsAt.toLocaleTimeString('ru', { hour: '2-digit', minute: '2-digit' })
  const date = startsAt.toLocaleDateString('ru', { day: 'numeric', month: 'long' })

  // Создаём запланированные уведомления
  await supabase.from('notifications').insert([
    {
      user_id: booking.student_id,
      booking_id: bookingId,
      type: 'sms',
      event: 'reminder_24h',
      send_at: new Date(startsAt.getTime() - 24 * 3600 * 1000).toISOString(),
      content: templates.reminder_24h(lessonName, time),
      status: 'pending',
    },
    {
      user_id: booking.student_id,
      booking_id: bookingId,
      type: 'sms',
      event: 'reminder_2h',
      send_at: new Date(startsAt.getTime() - 2 * 3600 * 1000).toISOString(),
      content: templates.reminder_2h(lessonName, time),
      status: 'pending',
    },
  ])
}
```

### Таблица notifications (добавить поле send_at)

```sql
alter table public.notifications 
add column if not exists send_at timestamptz;

-- Индекс для cron-выборки
create index idx_notifications_pending 
on public.notifications(send_at) 
where status = 'pending';
```

### Cron endpoint (systemd timer на VPS — основной вариант)

```typescript
// src/app/api/cron/send-reminders/route.ts
export async function GET(request: Request) {
  // Защита: проверяем секретный заголовок от cron
  if (request.headers.get('x-cron-secret') !== process.env.CRON_SECRET) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const supabase = createAdminClient()
  const now = new Date().toISOString()

  const { data: pending } = await supabase
    .from('notifications')
    .select('*, profile:profiles(phone)')
    .eq('status', 'pending')
    .eq('type', 'sms')
    .lte('send_at', now)
    .limit(50)

  for (const notif of pending ?? []) {
    const result = await sendSMS(notif.profile.phone, notif.content)
    await supabase.from('notifications').update({
      status: result.success ? 'sent' : 'failed',
      sent_at: result.success ? now : null,
      error: result.error ?? null,
    }).eq('id', notif.id)
  }

  return Response.json({ processed: pending?.length ?? 0 })
}
```

**На VPS** — через systemd timer каждые 15 минут:
```bash
# /etc/systemd/system/tramplin-reminders.timer
[Timer]
OnCalendar=*:0/15
```
