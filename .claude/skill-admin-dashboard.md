---
name: tramplin-admin
description: Административная панель tramplin.pro. Читай этот файл когда нужно реализовать любой раздел для администратора: дашборд с метриками, управление расписанием, переносы уроков, финансовые отчёты, управление пользователями.
---

# Skill: Административная панель

Всегда читай `AGENT.md` перед этим файлом.

---

## Структура разделов

```
/admin
├── /dashboard        ← Главный экран: KPI, занятость, выручка
├── /schedule         ← Расписание всей школы (создание/редактирование слотов)
├── /bookings         ← Все записи + переносы + отмены
├── /students         ← Список студентов, балансы, история
├── /teachers         ← Список преподавателей, доступность
├── /courses          ← Управление курсами и ценами
├── /payments         ← Финансы, история платежей
└── /notifications    ← Ручные рассылки уведомлений
```

---

## Главный дашборд (/admin/dashboard)

### Ключевые метрики (KPI карточки)

```typescript
// src/app/(admin)/admin/dashboard/page.tsx

async function getDashboardStats() {
  const supabase = createClient()
  const today = new Date()
  const monthStart = new Date(today.getFullYear(), today.getMonth(), 1).toISOString()

  const [
    { count: todayBookings },
    { count: monthBookings },
    { data: revenue },
    { count: activeStudents },
    { data: todaySlots },
  ] = await Promise.all([
    // Записи сегодня
    supabase.from('bookings')
      .select('*', { count: 'exact', head: true })
      .eq('status', 'confirmed')
      .gte('created_at', new Date().toDateString()),

    // Записи за месяц
    supabase.from('bookings')
      .select('*', { count: 'exact', head: true })
      .eq('status', 'confirmed')
      .gte('created_at', monthStart),

    // Выручка за месяц
    supabase.from('payments')
      .select('amount')
      .eq('status', 'succeeded')
      .gte('created_at', monthStart),

    // Активные студенты (хоть одна запись в этом месяце)
    supabase.from('bookings')
      .select('student_id', { count: 'exact', head: true })
      .gte('created_at', monthStart),

    // Расписание сегодня
    supabase.from('slots')
      .select('*, teacher:teachers(profile:profiles(full_name)), lesson_type:lesson_types(*), bookings(count)')
      .gte('starts_at', new Date().toDateString())
      .lt('starts_at', new Date(Date.now() + 86400000).toDateString())
      .order('starts_at'),
  ])

  const totalRevenue = revenue?.reduce((sum, p) => sum + Number(p.amount), 0) ?? 0

  return { todayBookings, monthBookings, totalRevenue, activeStudents, todaySlots }
}
```

### Виджеты дашборда

- **График выручки** — по дням за последние 30 дней (recharts LineChart)
- **Таблица занятости** — % заполненности слотов по дням недели
- **Топ преподаватели** — по количеству проведённых уроков
- **Последние переносы** — 5 последних изменений расписания

---

## Управление расписанием (/admin/schedule)

### Создание слота

```typescript
// src/app/api/admin/slots/route.ts
const createSlotSchema = z.object({
  teacher_id: z.string().uuid().optional(),
  lesson_type_id: z.string().uuid(),
  starts_at: z.string().datetime(),
  ends_at: z.string().datetime(),
  max_students: z.number().int().min(1).max(30),
  notes: z.string().optional(),
  // Повторение
  repeat: z.enum(['none', 'weekly', 'biweekly']).default('none'),
  repeat_until: z.string().datetime().optional(),
})

export async function POST(request: Request) {
  // Проверка admin роли (всегда!)
  await requireAdmin(request)
  
  const body = createSlotSchema.parse(await request.json())
  const supabase = createAdminClient()

  const slots = []
  let currentDate = new Date(body.starts_at)
  const endDate = body.repeat_until ? new Date(body.repeat_until) : currentDate
  const durationMs = new Date(body.ends_at).getTime() - new Date(body.starts_at).getTime()

  // Генерируем слоты с учётом повторений
  while (currentDate <= endDate) {
    slots.push({
      teacher_id: body.teacher_id,
      lesson_type_id: body.lesson_type_id,
      starts_at: currentDate.toISOString(),
      ends_at: new Date(currentDate.getTime() + durationMs).toISOString(),
      max_students: body.max_students,
      notes: body.notes,
    })

    if (body.repeat === 'weekly') currentDate = new Date(currentDate.getTime() + 7 * 86400000)
    else if (body.repeat === 'biweekly') currentDate = new Date(currentDate.getTime() + 14 * 86400000)
    else break
  }

  const { data, error } = await supabase.from('slots').insert(slots).select()
  if (error) return Response.json({ error: error.message }, { status: 422 })
  
  return Response.json({ created: data.length }, { status: 201 })
}
```

---

## Переносы уроков (/admin/bookings)

```typescript
// Перенос = отмена старой записи + создание новой
// src/app/api/admin/bookings/[id]/transfer/route.ts

const transferSchema = z.object({
  new_slot_id: z.string().uuid(),
  reason: z.string().optional(),
  notify_student: z.boolean().default(true),
})

export async function POST(
  request: Request,
  { params }: { params: { id: string } }
) {
  await requireAdmin(request)
  const body = transferSchema.parse(await request.json())
  const supabase = createAdminClient()

  // Транзакция через RPC
  const { data, error } = await supabase.rpc('transfer_booking', {
    p_booking_id: params.id,
    p_new_slot_id: body.new_slot_id,
    p_reason: body.reason ?? 'Перенос администратором',
  })

  if (error) return Response.json({ error: error.message }, { status: 422 })

  // Уведомить студента
  if (body.notify_student) {
    // TODO: вызов skill-notifications
  }

  return Response.json({ new_booking_id: data })
}
```

```sql
-- SQL функция переноса (атомарно)
create or replace function transfer_booking(
  p_booking_id uuid,
  p_new_slot_id uuid,
  p_reason text
) returns uuid language plpgsql security definer as $$
declare
  v_old_booking record;
  v_new_booking_id uuid;
begin
  select * into v_old_booking from public.bookings where id = p_booking_id for update;

  -- Отменяем старую запись
  update public.bookings set 
    status = 'cancelled',
    cancelled_at = now(),
    cancel_reason = p_reason
  where id = p_booking_id;

  -- Возвращаем слот в доступные (если он был один)
  update public.slots set status = 'available' where id = v_old_booking.slot_id;

  -- Создаём новую запись (без списания минут — это перенос)
  insert into public.bookings (slot_id, student_id, status, payment_type, purchase_id)
  values (p_new_slot_id, v_old_booking.student_id, 'confirmed', 
          v_old_booking.payment_type, v_old_booking.purchase_id)
  returning id into v_new_booking_id;

  return v_new_booking_id;
end;
$$;
```

---

## Финансовый отчёт (/admin/payments)

```typescript
// Фильтры: период, статус, студент
// Экспорт в CSV

async function getPaymentsReport(from: string, to: string) {
  const supabase = createAdminClient()
  
  const { data } = await supabase
    .from('payments')
    .select(`
      id, amount, status, description, created_at,
      student:students(profile:profiles(full_name, phone))
    `)
    .eq('status', 'succeeded')
    .gte('created_at', from)
    .lte('created_at', to)
    .order('created_at', { ascending: false })

  return data
}

// Экспорт в CSV
function exportToCSV(payments: any[]) {
  const rows = payments.map(p => [
    p.created_at,
    p.student.profile.full_name,
    p.student.profile.phone,
    p.description,
    p.amount,
  ])
  
  const csv = [
    ['Дата', 'Студент', 'Телефон', 'Описание', 'Сумма (₽)'],
    ...rows,
  ].map(r => r.join(';')).join('\n')

  return new Blob(['\uFEFF' + csv], { type: 'text/csv;charset=utf-8' })  // BOM для Excel
}
```

---

## UI компоненты админки

```
src/components/admin/
├── StatsCard.tsx          — KPI карточка (число + изменение + иконка)
├── RevenueChart.tsx       — График выручки (recharts)
├── OccupancyTable.tsx     — Таблица занятости по дням
├── AdminCalendar.tsx      — Расписание с возможностью редактирования
├── BookingRow.tsx         — Строка записи с кнопками отмены/переноса
├── StudentCard.tsx        — Карточка студента с балансом и историей
└── TransferModal.tsx      — Модалка для переноса урока
```

---

## Хелпер: проверка admin роли в API

```typescript
// src/lib/auth/require-admin.ts
import { createClient } from '@/lib/supabase/server'

export async function requireAdmin(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Response('Не авторизован', { status: 401 })

  const { data: profile } = await supabase
    .from('profiles').select('role').eq('id', user.id).single()

  if (profile?.role !== 'admin') throw new Response('Нет доступа', { status: 403 })
  return user
}
```
