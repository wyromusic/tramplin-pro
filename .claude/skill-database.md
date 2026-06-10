---
name: tramplin-database
description: Схема базы данных Supabase для tramplin.pro. Читай этот файл когда нужно создать миграцию, добавить таблицу, написать запрос, настроить RLS политики, или разобраться в связях между сущностями.
---

# Skill: База данных (Supabase / PostgreSQL)

Всегда читай `AGENT.md` перед этим файлом.

---

## Схема таблиц

### users (управляется Supabase Auth)
```sql
-- Расширение профиля пользователя
create table public.profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  role        text not null check (role in ('student', 'teacher', 'admin')),
  full_name   text not null,
  phone       text,
  avatar_url  text,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now()
);
```

### teachers
```sql
create table public.teachers (
  id          uuid primary key default gen_random_uuid(),
  profile_id  uuid not null references public.profiles(id) on delete cascade,
  bio         text,
  specialization text[],  -- ['батут', 'акробатика', 'детские группы']
  is_active   boolean default true,
  created_at  timestamptz default now()
);
```

### students
```sql
create table public.students (
  id              uuid primary key default gen_random_uuid(),
  profile_id      uuid not null references public.profiles(id) on delete cascade,
  balance_minutes int default 0,  -- баланс в минутах
  notes           text,           -- заметки для преподавателей/админа
  created_at      timestamptz default now()
);
```

### lesson_types
```sql
create table public.lesson_types (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,  -- 'Вводный урок', 'Групповое занятие', etc.
  slug          text unique not null,  -- 'intro', 'group', 'individual', 'self_practice', 'rental'
  duration_min  int not null,   -- продолжительность в минутах
  price         numeric(10,2),  -- цена за разовое занятие (null если только по курсу)
  max_students  int default 1,  -- макс. студентов в группе
  color         text,           -- цвет в календаре (#hex)
  is_active     boolean default true
);
```

### courses
```sql
create table public.courses (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  description   text,
  lesson_type_id uuid references public.lesson_types(id),
  total_minutes int not null,   -- суммарно минут в курсе
  price         numeric(10,2) not null,
  is_active     boolean default true,
  created_at    timestamptz default now()
);
```

### slots (расписание)
```sql
create table public.slots (
  id            uuid primary key default gen_random_uuid(),
  teacher_id    uuid references public.teachers(id),  -- null = зал без преподавателя
  lesson_type_id uuid not null references public.lesson_types(id),
  starts_at     timestamptz not null,
  ends_at       timestamptz not null,
  max_students  int not null default 1,
  status        text not null default 'available'
                check (status in ('available', 'booked', 'cancelled', 'completed')),
  notes         text,
  created_at    timestamptz default now(),
  
  constraint valid_duration check (ends_at > starts_at)
);

create index idx_slots_starts_at on public.slots(starts_at);
create index idx_slots_teacher_id on public.slots(teacher_id);
create index idx_slots_status on public.slots(status);
```

### bookings (записи)
```sql
create table public.bookings (
  id            uuid primary key default gen_random_uuid(),
  slot_id       uuid not null references public.slots(id),
  student_id    uuid not null references public.students(id),
  status        text not null default 'confirmed'
                check (status in ('pending', 'confirmed', 'cancelled', 'completed', 'no_show')),
  payment_type  text check (payment_type in ('balance', 'purchase', 'free', 'cash')),
  purchase_id   uuid references public.purchases(id),
  minutes_spent int,            -- фактически потрачено минут (заполняется после урока)
  cancelled_at  timestamptz,
  cancel_reason text,
  created_at    timestamptz default now(),
  
  unique(slot_id, student_id)   -- нельзя записаться на один слот дважды
);

create index idx_bookings_student_id on public.bookings(student_id);
create index idx_bookings_slot_id on public.bookings(slot_id);
```

### purchases (покупки курсов и пакетов)
```sql
create table public.purchases (
  id              uuid primary key default gen_random_uuid(),
  student_id      uuid not null references public.students(id),
  course_id       uuid references public.courses(id),
  minutes_total   int not null,
  minutes_used    int default 0,
  price_paid      numeric(10,2) not null,
  payment_id      uuid references public.payments(id),
  expires_at      timestamptz,  -- срок действия пакета
  created_at      timestamptz default now()
);
```

### payments (платежи ЮKassa)
```sql
create table public.payments (
  id                  uuid primary key default gen_random_uuid(),
  student_id          uuid not null references public.students(id),
  amount              numeric(10,2) not null,
  currency            text default 'RUB',
  status              text not null default 'pending'
                      check (status in ('pending', 'waiting_for_capture', 'succeeded', 'cancelled')),
  yukassa_payment_id  text unique,  -- ID платежа в ЮKassa
  yukassa_payment_url text,         -- URL для редиректа на оплату
  idempotency_key     text unique not null,  -- для идемпотентности
  description         text,
  metadata            jsonb,        -- доп. данные (course_id, slot_id и тп)
  created_at          timestamptz default now(),
  updated_at          timestamptz default now()
);
```

### notifications
```sql
create table public.notifications (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references public.profiles(id),
  booking_id    uuid references public.bookings(id),
  type          text not null check (type in ('sms', 'push')),
  event         text not null,  -- 'reminder_24h', 'reminder_2h', 'booking_confirmed', 'booking_cancelled', 'transfer'
  status        text default 'pending' check (status in ('pending', 'sent', 'failed')),
  content       text not null,
  sent_at       timestamptz,
  error         text,
  created_at    timestamptz default now()
);
```

### teacher_availability (шаблоны доступности преподавателей)
```sql
create table public.teacher_availability (
  id            uuid primary key default gen_random_uuid(),
  teacher_id    uuid not null references public.teachers(id) on delete cascade,
  day_of_week   int not null check (day_of_week between 0 and 6),  -- 0=вс, 1=пн...
  start_time    time not null,
  end_time      time not null,
  is_active     boolean default true,
  
  constraint valid_time check (end_time > start_time)
);
```

---

## RLS политики (Row Level Security)

```sql
-- Включить RLS на всех таблицах
alter table public.profiles enable row level security;
alter table public.bookings enable row level security;
alter table public.payments enable row level security;
-- ... и тд для всех таблиц

-- Профили: видишь только свой, admin видит все
create policy "profiles_own" on public.profiles
  for select using (auth.uid() = id);

create policy "profiles_admin" on public.profiles
  for all using (
    exists (select 1 from public.profiles p 
            where p.id = auth.uid() and p.role = 'admin')
  );

-- Записи: студент видит свои, преподаватель — свои слоты, admin — все
create policy "bookings_student_own" on public.bookings
  for select using (
    student_id in (
      select id from public.students where profile_id = auth.uid()
    )
  );

create policy "bookings_teacher" on public.bookings
  for select using (
    slot_id in (
      select s.id from public.slots s
      join public.teachers t on t.id = s.teacher_id
      where t.profile_id = auth.uid()
    )
  );

-- Платежи: только свои
create policy "payments_own" on public.payments
  for select using (
    student_id in (
      select id from public.students where profile_id = auth.uid()
    )
  );
```

---

## Хелперы (Supabase функции)

### Списание минут при записи
```sql
create or replace function book_slot(
  p_slot_id uuid,
  p_student_id uuid,
  p_payment_type text,
  p_purchase_id uuid default null
) returns uuid language plpgsql security definer as $$
declare
  v_booking_id uuid;
  v_slot record;
  v_minutes int;
begin
  -- Блокируем слот для предотвращения race condition
  select * into v_slot from public.slots 
  where id = p_slot_id for update;
  
  if v_slot.status != 'available' then
    raise exception 'Слот уже недоступен';
  end if;
  
  v_minutes := extract(epoch from (v_slot.ends_at - v_slot.starts_at)) / 60;
  
  -- Создаём запись
  insert into public.bookings (slot_id, student_id, status, payment_type, purchase_id)
  values (p_slot_id, p_student_id, 'confirmed', p_payment_type, p_purchase_id)
  returning id into v_booking_id;
  
  -- Списываем минуты с баланса или покупки
  if p_payment_type = 'balance' then
    update public.students set balance_minutes = balance_minutes - v_minutes
    where id = p_student_id;
  elsif p_payment_type = 'purchase' and p_purchase_id is not null then
    update public.purchases set minutes_used = minutes_used + v_minutes
    where id = p_purchase_id;
  end if;
  
  -- Обновляем статус слота если он индивидуальный
  update public.slots set status = 'booked' 
  where id = p_slot_id and max_students = 1;
  
  return v_booking_id;
end;
$$;
```

---

## Типы для TypeScript

Генерировать автоматически через Supabase CLI:
```bash
npx supabase gen types typescript --project-id YOUR_PROJECT_ID > src/types/database.ts
```

После каждой миграции — перегенерировать типы!

---

## Миграции

- Хранятся в `supabase/migrations/`
- Именование: `YYYYMMDDHHMMSS_описание.sql`
- Никогда не редактировать уже применённые миграции — создавать новые
- Тестировать локально через `supabase start` перед деплоем
