---
name: tramplin-booking
description: Логика записи на занятия для tramplin.pro. Читай этот файл когда нужно реализовать или изменить: запись студента на урок, отображение расписания, отмену/перенос записи, проверку баланса, форму записи через веб.
---

# Skill: Запись на занятия

Всегда читай `AGENT.md` перед этим файлом. Схему таблиц см. в `skill-database.md`.

---

## Типы занятий и правила записи

| Тип (`slug`) | Название | Оплата | Особенности |
|---|---|---|---|
| `intro` | Вводный урок | Бесплатно или предоплата | 1 раз на студента |
| `group` | Групповое занятие | Баланс / курс | До N студентов |
| `individual` | Индивидуальный урок | Баланс / курс | Только 1 студент |
| `self_practice` | Самостоятельная практика | Баланс / разовая оплата | Без преподавателя |
| `rental` | Аренда зала | Разовая оплата / наличные | Без преподавателя |

---

## API Routes

### GET /api/slots
Получить доступные слоты.

```typescript
// src/app/api/slots/route.ts
import { createClient } from '@/lib/supabase/server'
import { z } from 'zod'

const querySchema = z.object({
  from: z.string().datetime(),
  to: z.string().datetime(),
  lesson_type: z.string().optional(),
  teacher_id: z.string().uuid().optional(),
})

export async function GET(request: Request) {
  const supabase = createClient()
  const { searchParams } = new URL(request.url)
  
  const params = querySchema.parse({
    from: searchParams.get('from'),
    to: searchParams.get('to'),
    lesson_type: searchParams.get('lesson_type'),
    teacher_id: searchParams.get('teacher_id'),
  })

  const query = supabase
    .from('slots')
    .select(`
      *,
      teacher:teachers(id, profile:profiles(full_name, avatar_url)),
      lesson_type:lesson_types(*),
      bookings(count)
    `)
    .eq('status', 'available')
    .gte('starts_at', params.from)
    .lte('starts_at', params.to)
    .order('starts_at')

  if (params.lesson_type) query.eq('lesson_type.slug', params.lesson_type)
  if (params.teacher_id) query.eq('teacher_id', params.teacher_id)

  const { data, error } = await query
  if (error) return Response.json({ error: error.message }, { status: 500 })
  
  return Response.json(data)
}
```

### POST /api/bookings
Создать запись на занятие.

```typescript
// src/app/api/bookings/route.ts
const bookingSchema = z.object({
  slot_id: z.string().uuid(),
  payment_type: z.enum(['balance', 'purchase', 'free', 'cash']),
  purchase_id: z.string().uuid().optional(),
})

export async function POST(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: 'Не авторизован' }, { status: 401 })

  const body = bookingSchema.parse(await request.json())

  // Получаем student_id
  const { data: student } = await supabase
    .from('students')
    .select('id, balance_minutes')
    .eq('profile_id', user.id)
    .single()

  if (!student) return Response.json({ error: 'Профиль студента не найден' }, { status: 404 })

  // Проверяем баланс если нужно
  if (body.payment_type === 'balance') {
    const { data: slot } = await supabase
      .from('slots')
      .select('starts_at, ends_at')
      .eq('id', body.slot_id)
      .single()
    
    const durationMin = (new Date(slot!.ends_at).getTime() - new Date(slot!.starts_at).getTime()) / 60000
    
    if (student.balance_minutes < durationMin) {
      return Response.json({ 
        error: `Недостаточно минут на балансе. Нужно: ${durationMin}, есть: ${student.balance_minutes}` 
      }, { status: 422 })
    }
  }

  // Атомарно создаём запись через RPC
  const { data: bookingId, error } = await supabase.rpc('book_slot', {
    p_slot_id: body.slot_id,
    p_student_id: student.id,
    p_payment_type: body.payment_type,
    p_purchase_id: body.purchase_id ?? null,
  })

  if (error) return Response.json({ error: error.message }, { status: 422 })

  // Планируем уведомления (см. skill-notifications.md)
  await scheduleBookingNotifications(bookingId, user.id)

  return Response.json({ booking_id: bookingId }, { status: 201 })
}
```

### PATCH /api/bookings/[id]/cancel
Отмена записи.

```typescript
// Правила отмены:
// - Студент может отменить за 2+ часа до урока
// - При отмене — минуты возвращаются на баланс
// - При отмене менее чем за 2 часа — минуты НЕ возвращаются (настраивается)

export async function PATCH(request: Request, { params }: { params: { id: string } }) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  
  const { data: booking } = await supabase
    .from('bookings')
    .select('*, slot:slots(*), student:students(*)')
    .eq('id', params.id)
    .single()

  if (!booking) return Response.json({ error: 'Запись не найдена' }, { status: 404 })

  const hoursUntilLesson = (new Date(booking.slot.starts_at).getTime() - Date.now()) / 3600000
  const canRefund = hoursUntilLesson >= 2

  // Обновляем статус записи
  await supabase.from('bookings').update({
    status: 'cancelled',
    cancelled_at: new Date().toISOString(),
    cancel_reason: (await request.json()).reason,
  }).eq('id', params.id)

  // Возвращаем минуты если ранняя отмена
  if (canRefund && booking.payment_type === 'balance') {
    const durationMin = (new Date(booking.slot.ends_at).getTime() - 
                         new Date(booking.slot.starts_at).getTime()) / 60000
    await supabase.from('students')
      .update({ balance_minutes: booking.student.balance_minutes + durationMin })
      .eq('id', booking.student.id)
  }

  return Response.json({ refunded: canRefund })
}
```

---

## Компоненты

### WeeklyCalendar
Отображает слоты на неделю, цвет = тип занятия.

```
src/components/calendar/WeeklyCalendar.tsx
- Props: { from: Date, lessonType?: string, teacherId?: string }
- Использует React Query для загрузки слотов
- Клик по слоту → BookingModal
- Цвет берётся из lesson_types.color
```

### BookingModal
Форма подтверждения записи.

```
src/components/booking/BookingModal.tsx
- Показывает детали слота
- Выбор способа оплаты (баланс / курс / разовая)
- Проверяет баланс на клиенте
- Submit → POST /api/bookings
```

### StudentBookingsList  
Список записей студента с возможностью отмены.

```
src/components/booking/StudentBookingsList.tsx
- Вкладки: Предстоящие / Прошедшие
- Кнопка "Отменить" только если > 2 часов
- Кнопка "Перенести" → открывает WeeklyCalendar для выбора нового слота
```

---

## Бизнес-правила (важно!)

1. **Вводный урок** — студент может записаться только 1 раз. Проверять через `bookings` + `lesson_types.slug = 'intro'`.
2. **Групповые занятия** — при достижении `max_students` слот закрывается автоматически.
3. **Перенос** — это отмена старой записи + новая запись. Минуты возвращаются всегда при переносе (даже менее 2ч).
4. **Просроченные слоты** — Cron-задача раз в час переводит `available` слоты в прошлом в `completed`.
5. **Конфликт записей** — нельзя записаться на пересекающиеся по времени слоты.

---

## Cron задачи

Реализуются через Supabase Edge Functions + pg_cron:

```sql
-- Автоматически завершать прошедшие слоты
select cron.schedule(
  'complete-past-slots',
  '0 * * * *',  -- каждый час
  $$
    update public.slots 
    set status = 'completed' 
    where status = 'available' and ends_at < now();
  $$
);
```
