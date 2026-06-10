---
name: tramplin-teacher
description: Кабинет преподавателя tramplin.pro. Читай этот файл когда нужно: реализовать личный кабинет преподавателя, управление расписанием доступности, список своих студентов, отметку посещаемости, просмотр предстоящих уроков.
---

# Skill: Кабинет преподавателя

Всегда читай `AGENT.md` перед этим файлом.

---

## Структура разделов

```
/teacher
├── /schedule         ← Расписание: свои уроки + настройка доступности
├── /students         ← Мои студенты + их прогресс
├── /attendance       ← Отметка посещаемости
└── /profile          ← Профиль, специализация, фото
```

---

## Расписание преподавателя (/teacher/schedule)

### Просмотр своих уроков

```typescript
// src/app/api/teacher/slots/route.ts
export async function GET(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()

  const { searchParams } = new URL(request.url)
  const from = searchParams.get('from') ?? new Date().toISOString()
  const to = searchParams.get('to') ?? new Date(Date.now() + 7 * 86400000).toISOString()

  // Находим teacher_id
  const { data: teacher } = await supabase
    .from('teachers')
    .select('id')
    .eq('profile_id', user!.id)
    .single()

  const { data: slots } = await supabase
    .from('slots')
    .select(`
      *,
      lesson_type:lesson_types(*),
      bookings(
        id, status,
        student:students(
          id, 
          profile:profiles(full_name, phone, avatar_url)
        )
      )
    `)
    .eq('teacher_id', teacher!.id)
    .gte('starts_at', from)
    .lte('starts_at', to)
    .order('starts_at')

  return Response.json(slots)
}
```

### Настройка шаблона доступности

```typescript
// src/app/api/teacher/availability/route.ts

// GET — получить текущий шаблон
export async function GET(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()

  const { data: teacher } = await supabase
    .from('teachers').select('id').eq('profile_id', user!.id).single()

  const { data } = await supabase
    .from('teacher_availability')
    .select('*')
    .eq('teacher_id', teacher!.id)
    .eq('is_active', true)
    .order('day_of_week')

  return Response.json(data)
}

// PUT — обновить шаблон (полная замена)
const availabilitySchema = z.array(z.object({
  day_of_week: z.number().int().min(0).max(6),
  start_time: z.string().regex(/^\d{2}:\d{2}$/),
  end_time: z.string().regex(/^\d{2}:\d{2}$/),
}))

export async function PUT(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  const body = availabilitySchema.parse(await request.json())

  const { data: teacher } = await supabase
    .from('teachers').select('id').eq('profile_id', user!.id).single()

  // Деактивируем старый шаблон
  await supabase.from('teacher_availability')
    .update({ is_active: false })
    .eq('teacher_id', teacher!.id)

  // Создаём новый
  if (body.length > 0) {
    await supabase.from('teacher_availability').insert(
      body.map(slot => ({
        teacher_id: teacher!.id,
        day_of_week: slot.day_of_week,
        start_time: slot.start_time,
        end_time: slot.end_time,
      }))
    )
  }

  return Response.json({ ok: true })
}
```

### Компонент WeeklyAvailabilityEditor

```
src/components/teacher/WeeklyAvailabilityEditor.tsx

- Визуальная сетка: 7 дней × временные блоки (9:00 - 22:00)
- Клик по ячейке = добавить/убрать блок доступности
- Drag для выделения диапазона
- "Сохранить" → PUT /api/teacher/availability
- Предупреждение: изменение шаблона не отменяет уже созданные слоты
```

---

## Мои студенты (/teacher/students)

```typescript
// src/app/api/teacher/students/route.ts
// Студенты = те, кто записывался на уроки к этому преподавателю

export async function GET(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()

  const { data: teacher } = await supabase
    .from('teachers').select('id').eq('profile_id', user!.id).single()

  // Уникальные студенты через bookings → slots → teacher
  const { data } = await supabase
    .from('bookings')
    .select(`
      student:students(
        id,
        balance_minutes,
        notes,
        profile:profiles(full_name, phone, avatar_url),
        bookings(
          id, status, created_at,
          slot:slots(starts_at, ends_at, lesson_type:lesson_types(name))
        )
      )
    `)
    .eq('slot.teacher_id', teacher!.id)
    .eq('status', 'confirmed')
    .order('created_at', { ascending: false })

  // Дедупликация студентов
  const uniqueStudents = [...new Map(
    data?.map(b => [b.student.id, b.student])
  ).values()]

  return Response.json(uniqueStudents)
}
```

### Заметки о студенте

```typescript
// PATCH /api/teacher/students/[id]/notes
// Преподаватель может добавлять заметки о прогрессе студента

export async function PATCH(request: Request, { params }: { params: { id: string } }) {
  const supabase = createClient()
  const { notes } = await request.json()

  // Проверяем что этот студент действительно занимался у данного преподавателя
  // (RLS это не покрывает — нужна явная проверка)
  
  await supabase.from('students')
    .update({ notes })
    .eq('id', params.id)

  return Response.json({ ok: true })
}
```

---

## Отметка посещаемости (/teacher/attendance)

```typescript
// PATCH /api/teacher/bookings/[id]/attendance
const attendanceSchema = z.object({
  status: z.enum(['completed', 'no_show']),
  minutes_spent: z.number().int().optional(),  // Фактически потраченное время
})

export async function PATCH(
  request: Request,
  { params }: { params: { id: string } }
) {
  const supabase = createClient()
  const body = attendanceSchema.parse(await request.json())

  // Проверяем что урок принадлежит этому преподавателю
  const { data: { user } } = await supabase.auth.getUser()
  const { data: teacher } = await supabase
    .from('teachers').select('id').eq('profile_id', user!.id).single()

  const { data: booking } = await supabase
    .from('bookings')
    .select('*, slot:slots(*)')
    .eq('id', params.id)
    .single()

  if (booking?.slot.teacher_id !== teacher!.id) {
    return Response.json({ error: 'Нет доступа' }, { status: 403 })
  }

  // Проверяем что урок уже прошёл
  if (new Date(booking.slot.ends_at) > new Date()) {
    return Response.json({ error: 'Урок ещё не завершён' }, { status: 422 })
  }

  await supabase.from('bookings').update({
    status: body.status,
    minutes_spent: body.minutes_spent,
  }).eq('id', params.id)

  // Если no_show — минуты НЕ возвращаются (можно настроить)

  return Response.json({ ok: true })
}
```

### Компонент AttendanceList

```
src/components/teacher/AttendanceList.tsx

- Показывает записи за прошедшие 7 дней без отметки
- Для каждой строки: имя студента, тип урока, время
- Кнопки: "Был" (completed) / "Не пришёл" (no_show)
- После отметки — строка исчезает из списка
- Счётчик непроверенных записей в навигации
```

---

## Инициирование переноса

Преподаватель НЕ переносит самостоятельно — только отправляет запрос администратору.

```typescript
// POST /api/teacher/transfer-requests
const transferRequestSchema = z.object({
  slot_id: z.string().uuid(),
  reason: z.string().min(10),
  preferred_dates: z.array(z.string().datetime()).optional(),
})

// Создаёт запись в отдельной таблице transfer_requests
// Администратор видит их в /admin/bookings и обрабатывает
```

```sql
create table public.transfer_requests (
  id          uuid primary key default gen_random_uuid(),
  slot_id     uuid not null references public.slots(id),
  teacher_id  uuid not null references public.teachers(id),
  reason      text not null,
  preferred_dates timestamptz[],
  status      text default 'pending' check (status in ('pending', 'approved', 'rejected')),
  created_at  timestamptz default now()
);
```
