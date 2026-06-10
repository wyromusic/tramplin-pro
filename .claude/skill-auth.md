---
name: tramplin-auth
description: Аутентификация и роли для tramplin.pro. Читай этот файл когда нужно: настроить вход/регистрацию, проверить роль пользователя, защитить роут, получить текущего пользователя, разобраться с Supabase Auth и RLS.
---

# Skill: Авторизация и роли

Всегда читай `AGENT.md` перед этим файлом.

---

## Роли и доступы

| Роль | Регистрация | Доступные разделы |
|------|------------|-------------------|
| `student` | Самостоятельно через форму | `/`, `/booking`, `/my-bookings`, `/my-courses`, `/profile` |
| `teacher` | Создаёт только admin | `/teacher/*` |
| `admin` | Создаёт только другой admin | `/admin/*` |

---

## Supabase клиенты

```typescript
// src/lib/supabase/server.ts — для Server Components и API Routes
import { createServerClient as createSupabaseServerClient } from '@supabase/ssr'
import { cookies } from 'next/headers'
import type { Database } from '@/types/database'

export function createClient() {
  const cookieStore = cookies()
  return createSupabaseServerClient<Database>(
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
  // Только на сервере! service_role обходит RLS
  return createSupabaseServerClient<Database>(
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

## Хук: текущий пользователь (клиент)

```typescript
// src/hooks/useCurrentUser.ts
import { useEffect, useState } from 'react'
import { createClient } from '@/lib/supabase/client'
import type { Profile } from '@/types/database'

export function useCurrentUser() {
  const [profile, setProfile] = useState<Profile | null>(null)
  const [loading, setLoading] = useState(true)
  const supabase = createClient()

  useEffect(() => {
    supabase.auth.getUser().then(async ({ data: { user } }) => {
      if (!user) { setLoading(false); return }
      
      const { data } = await supabase
        .from('profiles')
        .select('*')
        .eq('id', user.id)
        .single()
      
      setProfile(data)
      setLoading(false)
    })

    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      async (event, session) => {
        if (!session) { setProfile(null); return }
        const { data } = await supabase
          .from('profiles').select('*').eq('id', session.user.id).single()
        setProfile(data)
      }
    )

    return () => subscription.unsubscribe()
  }, [])

  return { profile, loading, isStudent: profile?.role === 'student',
           isTeacher: profile?.role === 'teacher', isAdmin: profile?.role === 'admin' }
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

  // Неавторизованные → на /login
  if (!session && (path.startsWith('/my-') || path.startsWith('/admin') || path.startsWith('/teacher'))) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Роутинг по ролям
  if (session && (path.startsWith('/admin') || path.startsWith('/teacher'))) {
    const { data: profile } = await supabase
      .from('profiles').select('role').eq('id', session.user.id).single()

    if (path.startsWith('/admin') && profile?.role !== 'admin') {
      return NextResponse.redirect(new URL('/', request.url))
    }
    if (path.startsWith('/teacher') && profile?.role !== 'teacher' && profile?.role !== 'admin') {
      return NextResponse.redirect(new URL('/', request.url))
    }
  }

  return response
}

export const config = {
  matcher: ['/my-:path*', '/admin/:path*', '/teacher/:path*'],
}
```

---

## Регистрация студента

```typescript
// src/app/api/auth/register/route.ts
const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  full_name: z.string().min(2),
  phone: z.string().optional(),
})

export async function POST(request: Request) {
  const supabase = createClient()
  const body = schema.parse(await request.json())

  // 1. Создаём пользователя в Supabase Auth
  const { data, error } = await supabase.auth.signUp({
    email: body.email,
    password: body.password,
  })

  if (error || !data.user) {
    return Response.json({ error: error?.message ?? 'Ошибка регистрации' }, { status: 400 })
  }

  // 2. Создаём профиль (роль student по умолчанию)
  const adminClient = createAdminClient()
  await adminClient.from('profiles').insert({
    id: data.user.id,
    role: 'student',
    full_name: body.full_name,
    phone: body.phone,
  })

  // 3. Создаём запись студента
  await adminClient.from('students').insert({ profile_id: data.user.id })

  return Response.json({ ok: true }, { status: 201 })
}
```

---

## Создание преподавателя (только admin)

```typescript
// src/app/api/admin/teachers/route.ts
export async function POST(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  
  // Проверяем что запрашивающий — admin
  const { data: profile } = await supabase
    .from('profiles').select('role').eq('id', user!.id).single()
  
  if (profile?.role !== 'admin') {
    return Response.json({ error: 'Нет доступа' }, { status: 403 })
  }

  const body = await request.json()
  const adminClient = createAdminClient()

  // Создаём учётную запись
  const { data: newUser } = await adminClient.auth.admin.createUser({
    email: body.email,
    password: body.password,
    email_confirm: true,
  })

  await adminClient.from('profiles').insert({
    id: newUser!.user!.id,
    role: 'teacher',
    full_name: body.full_name,
    phone: body.phone,
  })

  await adminClient.from('teachers').insert({
    profile_id: newUser!.user!.id,
    bio: body.bio,
    specialization: body.specialization,
  })

  return Response.json({ ok: true }, { status: 201 })
}
```

---

## Важные правила

1. **Никогда не проверять роль только на фронте** — всегда дублировать проверку в API Route
2. **RLS — основная линия защиты** — настраивать политики в `skill-database.md`
3. **service_role** — только в серверных API Routes и webhook обработчиках, никогда в клиентском коде
4. **Email подтверждение** — настроить в Supabase Dashboard для студентов
