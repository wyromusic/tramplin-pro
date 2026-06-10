---
name: tramplin-auth
description: Аутентификация и роли для tramplin.pro. Читай этот файл когда нужно: реализовать вход по SMS, проверить роль пользователя, защитить роут, получить текущего пользователя, настроить Supabase Auth или кастомную SMS-авторизацию.
---

# Skill: Авторизация (вход по SMS) и роли

Всегда читай `AGENT.md` перед этим файлом.

---

## Метод входа — только номер телефона + SMS-код

Email НЕ используется как основной метод. Целевая аудитория — обычные люди с телефоном.

**Флоу:**
1. Пользователь вводит номер телефона в формате `+7XXXXXXXXXX`
2. Сервер генерирует 4-значный OTP код
3. Код отправляется через smsc.ru (см. `skill-notifications.md`)
4. Код хранится в БД таблице `otp_codes` с TTL 5 минут
5. Пользователь вводит код — сервер проверяет
6. При успехе — Supabase создаёт сессию (JWT в cookie)

---

## Supabase Auth: Phone OTP

Supabase поддерживает Phone OTP нативно, но его SMS-провайдер (Twilio) заблокирован в РФ.
**Решение:** Отключить встроенный SMS Supabase, реализовать OTP самостоятельно через smsc.ru.

```sql
-- Таблица для хранения OTP кодов
create table public.otp_codes (
  id          uuid primary key default gen_random_uuid(),
  phone       text not null,
  code        text not null,
  expires_at  timestamptz not null default (now() + interval '5 minutes'),
  used        boolean default false,
  created_at  timestamptz default now()
);

-- Индекс для быстрого поиска
create index idx_otp_phone on public.otp_codes(phone, expires_at);

-- Автоудаление старых кодов (через pg_cron или триггер)
```

---

## API Routes

### POST /api/auth/send-otp
Отправить SMS с кодом.

```typescript
// src/app/api/auth/send-otp/route.ts
import { z } from 'zod'
import { createAdminClient } from '@/lib/supabase/server'
import { sendSMS } from '@/lib/notifications/sms'

const schema = z.object({
  phone: z.string().regex(/^\+7\d{10}$/, 'Формат: +7XXXXXXXXXX'),
})

export async function POST(request: Request) {
  const { phone } = schema.parse(await request.json())
  const supabase = createAdminClient()

  // Проверяем rate limit: не больше 3 кодов за 10 минут
  const { count } = await supabase
    .from('otp_codes')
    .select('*', { count: 'exact', head: true })
    .eq('phone', phone)
    .gte('created_at', new Date(Date.now() - 10 * 60 * 1000).toISOString())

  if ((count ?? 0) >= 3) {
    return Response.json(
      { error: 'Слишком много попыток. Подождите 10 минут.' },
      { status: 429 }
    )
  }

  // Генерируем 4-значный код
  const code = Math.floor(1000 + Math.random() * 9000).toString()

  // Сохраняем в БД
  await supabase.from('otp_codes').insert({
    phone,
    code,
    expires_at: new Date(Date.now() + 5 * 60 * 1000).toISOString(),
  })

  // Отправляем SMS через smsc.ru
  const sent = await sendSMS(phone, `Ваш код для tramplin.pro: ${code}. Действует 5 минут.`)

  if (!sent) {
    return Response.json({ error: 'Ошибка отправки SMS. Попробуйте ещё раз.' }, { status: 500 })
  }

  return Response.json({ ok: true })
}
```

### POST /api/auth/verify-otp
Проверить код и войти.

```typescript
// src/app/api/auth/verify-otp/route.ts
const schema = z.object({
  phone: z.string().regex(/^\+7\d{10}$/),
  code: z.string().length(4),
})

export async function POST(request: Request) {
  const { phone, code } = schema.parse(await request.json())
  const supabase = createAdminClient()

  // Ищем действующий код
  const { data: otp } = await supabase
    .from('otp_codes')
    .select('*')
    .eq('phone', phone)
    .eq('code', code)
    .eq('used', false)
    .gte('expires_at', new Date().toISOString())
    .order('created_at', { ascending: false })
    .limit(1)
    .single()

  if (!otp) {
    return Response.json({ error: 'Неверный или истёкший код' }, { status: 401 })
  }

  // Помечаем код как использованный
  await supabase.from('otp_codes').update({ used: true }).eq('id', otp.id)

  // Находим или создаём пользователя
  let { data: profile } = await supabase
    .from('profiles')
    .select('id, role')
    .eq('phone', phone)
    .single()

  if (!profile) {
    // Новый пользователь — создаём через Supabase Auth
    const { data: authUser } = await supabase.auth.admin.createUser({
      phone,
      phone_confirm: true,
      user_metadata: { role: 'student' },
    })
    
    profile = { id: authUser.user!.id, role: 'student' }
    
    await supabase.from('profiles').insert({
      id: authUser.user!.id,
      phone,
      role: 'student',
    })
    
    await supabase.from('students').insert({
      profile_id: authUser.user!.id,
    })
  }

  // Создаём сессию
  const { data: session } = await supabase.auth.admin.createSession({
    user_id: profile.id,
  })

  // Возвращаем токен (фронт сохранит в cookie)
  return Response.json({
    access_token: session?.access_token,
    role: profile.role,
  })
}
```

---

## Supabase клиенты

```typescript
// src/lib/supabase/server.ts
import { createServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/database'

export function createClient() {
  const cookieStore = cookies()
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get: (name) => cookieStore.get(name)?.value,
        set: (name, value, options) => cookieStore.set({ name, value, ...options }),
        remove: (name, options) => cookieStore.set({ name, value: '', ...options }),
      },
    }
  )
}

export function createAdminClient() {
  // ⚠️ ТОЛЬКО на сервере! service_role обходит RLS
  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    { cookies: { get: () => '', set: () => {}, remove: () => {} } }
  )
}
```

```typescript
// src/lib/supabase/client.ts — для Client Components
import { createBrowserClient } from '@supabase/ssr'
import type { Database } from '@/types/database'

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

---

## Middleware: защита роутов

```typescript
// src/middleware.ts
import { createServerClient } from '@supabase/ssr'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  const response = NextResponse.next()
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get: (name) => request.cookies.get(name)?.value,
        set: (name, value, options) => response.cookies.set({ name, value, ...options }),
        remove: (name, options) => response.cookies.set({ name, value: '', ...options }),
      },
    }
  )

  const { data: { session } } = await supabase.auth.getSession()
  const path = request.nextUrl.pathname

  // Неавторизованных — на /login
  if (!session && (path.startsWith('/my') || path.startsWith('/admin') || path.startsWith('/teacher'))) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Проверяем роль для защищённых разделов
  if (session && (path.startsWith('/admin') || path.startsWith('/teacher'))) {
    const { data: profile } = await supabase
      .from('profiles').select('role').eq('id', session.user.id).single()

    if (path.startsWith('/admin') && profile?.role !== 'admin') {
      return NextResponse.redirect(new URL('/', request.url))
    }
    if (path.startsWith('/teacher') && !['teacher', 'admin'].includes(profile?.role ?? '')) {
      return NextResponse.redirect(new URL('/', request.url))
    }
  }

  return response
}

export const config = {
  matcher: ['/my/:path*', '/admin/:path*', '/teacher/:path*'],
}
```

---

## Хук: текущий пользователь (клиент)

```typescript
// src/hooks/useCurrentUser.ts
'use client'
import { useEffect, useState } from 'react'
import { createClient } from '@/lib/supabase/client'

export function useCurrentUser() {
  const [profile, setProfile] = useState<any>(null)
  const [loading, setLoading] = useState(true)
  const supabase = createClient()

  useEffect(() => {
    supabase.auth.getUser().then(async ({ data: { user } }) => {
      if (!user) { setLoading(false); return }
      const { data } = await supabase
        .from('profiles').select('*').eq('id', user.id).single()
      setProfile(data)
      setLoading(false)
    })
  }, [])

  return {
    profile,
    loading,
    isStudent: profile?.role === 'student',
    isTeacher: profile?.role === 'teacher',
    isAdmin: profile?.role === 'admin',
  }
}
```

---

## Создание преподавателя (только admin)

Преподаватели не регистрируются сами — их создаёт администратор.
Преподаватель получает SMS с информацией о своём аккаунте.

```typescript
// POST /api/admin/teachers
// Создаёт пользователя с ролью teacher
// Отправляет SMS: "Вы добавлены как преподаватель tramplin.pro. Войдите по номеру +7..."
```

---

## Важные правила безопасности

1. **Никогда не проверять роль только на фронте** — всегда дублировать в API Route
2. **service_role** — только в серверных обработчиках, никогда в браузере
3. **OTP rate limit** — максимум 3 попытки за 10 минут с одного номера
4. **OTP TTL** — 5 минут, после — недействителен
5. **RLS** — основная линия защиты данных в БД
6. **Логировать** все попытки входа (phone, ip, результат) в таблицу `auth_logs`
