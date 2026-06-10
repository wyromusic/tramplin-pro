---
name: tramplin-notifications
description: SMS и Push уведомления для tramplin.pro. Читай этот файл когда нужно: отправить уведомление студенту, настроить напоминания перед уроком, реализовать подписку на Push, разобраться с SMS провайдером.
---

# Skill: Уведомления (SMS + Web Push)

Всегда читай `AGENT.md` перед этим файлом.

---

## Провайдеры

| Тип | Провайдер | Переменная окружения |
|-----|-----------|---------------------|
| SMS | SMSC.ru | `SMSC_LOGIN`, `SMSC_PASSWORD` |
| Push | Web Push (VAPID) | `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_EMAIL` |

SMSC.ru выбран за наличие русского номера отправителя и простой API. Альтернатива: SMS.ru.

---

## Генерация VAPID ключей

```bash
npx web-push generate-vapid-keys
# Сохрани в .env.local
```

---

## SMS: отправка

```typescript
// src/lib/notifications/sms.ts

interface SmscResponse {
  id?: string
  cnt?: number
  error?: string
  error_code?: string
}

export async function sendSMS(phone: string, message: string): Promise<boolean> {
  // Нормализуем номер: +79991234567 → 79991234567
  const normalizedPhone = phone.replace(/\D/g, '').replace(/^8/, '7')
  
  const params = new URLSearchParams({
    login: process.env.SMSC_LOGIN!,
    psw: process.env.SMSC_PASSWORD!,
    phones: normalizedPhone,
    mes: message,
    fmt: '3',        // JSON ответ
    charset: 'utf-8',
    sender: 'TRAMPLIN',  // Имя отправителя (нужно зарегистрировать в SMSC)
  })

  try {
    const res = await fetch(`https://smsc.ru/sys/send.php?${params}`)
    const data: SmscResponse = await res.json()
    
    if (data.error) {
      console.error(`SMS ошибка [${normalizedPhone}]:`, data.error, data.error_code)
      return false
    }
    return true
  } catch (e) {
    console.error('SMS fetch ошибка:', e)
    return false
  }
}
```

---

## Push: подписка и отправка

### Клиент: подписка на уведомления

```typescript
// src/lib/notifications/push-client.ts

export async function subscribeToPush(): Promise<PushSubscription | null> {
  if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
    console.warn('Push уведомления не поддерживаются')
    return null
  }

  const permission = await Notification.requestPermission()
  if (permission !== 'granted') return null

  const registration = await navigator.serviceWorker.ready
  
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(
      process.env.NEXT_PUBLIC_VAPID_PUBLIC_KEY!
    ),
  })

  // Сохраняем подписку на сервере
  await fetch('/api/notifications/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription),
  })

  return subscription
}

function urlBase64ToUint8Array(base64String: string): Uint8Array {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4)
  const base64 = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/')
  const rawData = atob(base64)
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)))
}
```

### Таблица для хранения подписок

```sql
create table public.push_subscriptions (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references public.profiles(id) on delete cascade,
  endpoint    text not null,
  p256dh      text not null,
  auth        text not null,
  created_at  timestamptz default now(),
  unique(user_id, endpoint)
);
```

### API: сохранение подписки

```typescript
// src/app/api/notifications/subscribe/route.ts
export async function POST(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: 'Не авторизован' }, { status: 401 })

  const sub = await request.json()
  
  await supabase.from('push_subscriptions').upsert({
    user_id: user.id,
    endpoint: sub.endpoint,
    p256dh: sub.keys.p256dh,
    auth: sub.keys.auth,
  }, { onConflict: 'user_id,endpoint' })

  return Response.json({ ok: true })
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

export async function sendPushToUser(userId: string, payload: {
  title: string
  body: string
  url?: string
}): Promise<void> {
  const supabase = createAdminClient()  // service_role для чтения подписок
  
  const { data: subs } = await supabase
    .from('push_subscriptions')
    .select('*')
    .eq('user_id', userId)

  if (!subs?.length) return

  await Promise.allSettled(
    subs.map(sub =>
      webpush.sendNotification(
        { endpoint: sub.endpoint, keys: { p256dh: sub.p256dh, auth: sub.auth } },
        JSON.stringify(payload)
      ).catch(async (err) => {
        // Удаляем устаревшие подписки (410 Gone)
        if (err.statusCode === 410) {
          await supabase.from('push_subscriptions').delete().eq('id', sub.id)
        }
      })
    )
  )
}
```

### Service Worker

```javascript
// public/sw.js
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
  event.waitUntil(
    clients.openWindow(event.notification.data.url)
  )
})
```

Регистрировать SW в `src/app/layout.tsx`:
```typescript
useEffect(() => {
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js')
  }
}, [])
```

---

## Шаблоны уведомлений

```typescript
// src/lib/notifications/templates.ts

export const NotificationTemplates = {
  booking_confirmed: (lessonName: string, date: string, time: string) => ({
    sms: `Запись подтверждена! ${lessonName} ${date} в ${time}. tramplin.pro`,
    push: {
      title: '✅ Запись подтверждена',
      body: `${lessonName} — ${date} в ${time}`,
      url: '/my-bookings',
    },
  }),

  reminder_24h: (lessonName: string, time: string) => ({
    sms: `Напоминание: завтра ${lessonName} в ${time}. Ждём вас! tramplin.pro`,
    push: {
      title: '⏰ Урок завтра',
      body: `${lessonName} в ${time}`,
      url: '/my-bookings',
    },
  }),

  reminder_2h: (lessonName: string, time: string) => ({
    sms: `Напоминание: сегодня ${lessonName} в ${time} (через 2 часа). tramplin.pro`,
    push: {
      title: '⏰ Урок через 2 часа',
      body: `${lessonName} в ${time}`,
      url: '/my-bookings',
    },
  }),

  booking_cancelled: (lessonName: string, date: string) => ({
    sms: `Ваша запись на ${lessonName} ${date} отменена. Вопросы: tramplin.pro`,
    push: {
      title: '❌ Запись отменена',
      body: `${lessonName} — ${date}`,
      url: '/my-bookings',
    },
  }),

  lesson_transferred: (lessonName: string, oldDate: string, newDate: string) => ({
    sms: `Перенос занятия: ${lessonName} перенесён с ${oldDate} на ${newDate}. tramplin.pro`,
    push: {
      title: '📅 Занятие перенесено',
      body: `${lessonName}: ${oldDate} → ${newDate}`,
      url: '/my-bookings',
    },
  }),
}
```

---

## Планировщик напоминаний

Запускается после создания записи.

```typescript
// src/lib/notifications/scheduler.ts
export async function scheduleBookingNotifications(
  bookingId: string,
  userId: string
) {
  const supabase = createAdminClient()
  
  const { data: booking } = await supabase
    .from('bookings')
    .select('*, slot:slots(starts_at, ends_at, lesson_type:lesson_types(name))')
    .eq('id', bookingId)
    .single()

  const startsAt = new Date(booking!.slot.starts_at)
  const lessonName = booking!.slot.lesson_type.name
  const timeStr = startsAt.toLocaleTimeString('ru', { hour: '2-digit', minute: '2-digit' })
  const dateStr = startsAt.toLocaleDateString('ru', { day: 'numeric', month: 'long' })

  // Запланировать уведомления в таблице notifications
  const notifications = [
    {
      user_id: userId,
      booking_id: bookingId,
      event: 'reminder_24h',
      // send_at: за 24ч до урока (обрабатывается cron)
    },
    {
      user_id: userId,
      booking_id: bookingId,
      event: 'reminder_2h',
      // send_at: за 2ч до урока
    },
  ]

  await supabase.from('notifications').insert(
    notifications.map(n => ({
      ...n,
      type: 'push',  // или sms, зависит от предпочтений пользователя
      content: JSON.stringify(
        NotificationTemplates[n.event as keyof typeof NotificationTemplates](
          lessonName, dateStr, timeStr
        )
      ),
      status: 'pending',
    }))
  )
}
```

### Cron для отправки напоминаний

```sql
-- Каждые 15 минут проверяем и отправляем напоминания
select cron.schedule(
  'send-reminders',
  '*/15 * * * *',
  $$ select net.http_post(
    url := current_setting('app.edge_function_url') || '/send-reminders',
    headers := jsonb_build_object('Authorization', 'Bearer ' || current_setting('app.service_key'))
  ) $$
);
```

Или через Vercel Cron (в `vercel.json`):
```json
{
  "crons": [
    { "path": "/api/cron/send-reminders", "schedule": "*/15 * * * *" }
  ]
}
```
