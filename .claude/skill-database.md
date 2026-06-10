---
name: tramplin-database
description: Схема базы данных для tramplin.pro. Читай этот файл когда нужно создать миграцию, добавить таблицу, написать запрос, настроить RLS политики, или разобраться в связях между сущностями.
---

# Skill: База данных (PostgreSQL / Supabase)

Всегда читай `AGENT.md` перед этим файлом.

---

## Порядок создания таблиц (важно для FK)

Таблицы создаются строго в таком порядке — каждая ссылается только на уже существующие:

1. `profiles` (зависит от `auth.users`)
2. `teachers` (зависит от `profiles`)
3. `students` (зависит от `profiles`)
4. `lesson_types`
5. `courses` (зависит от `lesson_types`)
6. `slots` (зависит от `teachers`, `lesson_types`)
7. `payments` — ⚠️ создаём ДО `purchases` и `bookings`
8. `purchases` (зависит от `students`, `courses`, `payments`)
9. `bookings` (зависит от `slots`, `students`, `purchases`)
10. `notifications` (зависит от `profiles`, `bookings`)
11. `teacher_availability` (зависит от `teachers`)
12. `transfer_requests` (зависит от `slots`, `teachers`)
13. `otp_codes` — для SMS-авторизации
14. `auth_logs` — лог попыток входа
15. `push_subscriptions` (зависит от `profiles`)

---

## Схема таблиц

### 1. profiles
```sql
create table public.profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  role        text not null check (role in ('student', 'teacher', 'admin')),
  full_name   text not null default '',
  phone       text unique,           -- основной идентификатор (вход по телефону)
  email       text,                  -- опционально, только для уведомлений
  avatar_url  text,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now()
);

create index idx_profiles_phone on public.profiles(phone);
```

### 2. teachers
```sql
create table public.teachers (
  id              uuid primary key default gen_random_uuid(),
  profile_id      uuid not null unique references public.profiles(id) on delete cascade,
  bio             text,
  specialization  text[],
  is_active       boolean default true,
  created_at      timestamptz default now()
);
```

### 3. students
```sql
create table public.students (
  id              uuid primary key default gen_random_uuid(),
  profile_id      uuid not null unique references public.profiles(id) on delete cascade,
  balance_minutes int not null default 0 check (balance_minutes >= 0),
  notes           text,
  created_at      timestamptz default now()
);
```

### 4. lesson_types
```sql
create table public.lesson_types (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  slug          text unique not null,
  -- Допустимые slug: intro | group | individual | self_practice | rental
  duration_min  int not null check (duration_min > 0),
  price         numeric(10,2),       -- null = только по курсу/балансу
  max_students  int not null default 1,
  color         text default '#6366f1',  -- цвет в календаре
  is_active     boolean default true
);

-- Заполняем начальные типы
insert into public.lesson_types (name, slug, duration_min, price, max_students, color) values
  ('Вводный урок',            'intro',         60, null,    1,  '#22c55e'),
  ('Групповое занятие',       'group',         60, 800,     8,  '#3b82f6'),
  ('Индивидуальный урок',     'individual',    60, 2500,    1,  '#f59e0b'),
  ('Самостоятельная практика','self_practice', 60, 500,     10, '#8b5cf6'),
  ('Аренда зала',             'rental',        60, 3000,    20, '#ec4899');
```

### 5. courses
```sql
create table public.courses (
  id              uuid primary key default gen_random_uuid(),
  name            text not null,
  description     text,
  lesson_type_id  uuid references public.lesson_types(id),
  total_minutes   int not null check (total_minutes > 0),
  price           numeric(10,2) not null check (price >= 0),
  expires_days    int default 365,    -- срок действия пакета в днях
  is_active       boolean default true,
  created_at      timestamptz default now()
);
```

### 6. slots
```sql
create table public.slots (
  id              uuid primary key default gen_random_uuid(),
  teacher_id      uuid references public.teachers(id),   -- null = зал без преподавателя
  lesson_type_id  uuid not null references public.lesson_types(id),
  starts_at       timestamptz not null,
  ends_at         timestamptz not null,
  max_students    int not null default 1,
  status          text not null default 'available'
                  check (status in ('available', 'booked', 'cancelled', 'completed')),
  notes           text,
  created_at      timestamptz default now(),

  constraint slots_valid_duration check (ends_at > starts_at)
);

create index idx_slots_starts_at on public.slots(starts_at);
create index idx_slots_teacher_id on public.slots(teacher_id);
create index idx_slots_status_starts on public.slots(status, starts_at);
```

### 7. payments (создаём ДО bookings и purchases!)
```sql
create table public.payments (
  id                  uuid primary key default gen_random_uuid(),
  student_id          uuid not null references public.students(id),
  amount              numeric(10,2) not null check (amount > 0),
  currency            text not null default 'RUB',
  status              text not null default 'pending'
                      check (status in ('pending', 'waiting_for_capture', 'succeeded', 'cancelled')),
  yukassa_payment_id  text unique,
  yukassa_payment_url text,
  idempotency_key     text unique not null,
  description         text,
  metadata            jsonb default '{}',
  created_at          timestamptz default now(),
  updated_at          timestamptz default now()
);

create index idx_payments_student on public.payments(student_id);
create index idx_payments_yukassa on public.payments(yukassa_payment_id);
```

### 8. purchases
```sql
create table public.purchases (
  id              uuid primary key default gen_random_uuid(),
  student_id      uuid not null references public.students(id),
  course_id       uuid references public.courses(id),
  minutes_total   int not null check (minutes_total > 0),
  minutes_used    int not null default 0 check (minutes_used >= 0),
  price_paid      numeric(10,2) not null,
  payment_id      uuid references public.payments(id),
  expires_at      timestamptz,
  created_at      timestamptz default now(),

  constraint purchases_used_lte_total check (minutes_used <= minutes_total)
);

create index idx_purchases_student on public.purchases(student_id);
```

### 9. bookings
```sql
create table public.bookings (
  id            uuid primary key default gen_random_uuid(),
  slot_id       uuid not null references public.slots(id),
  student_id    uuid not null references public.students(id),
  status        text not null default 'confirmed'
                check (status in ('pending', 'confirmed', 'cancelled', 'completed', 'no_show')),
  payment_type  text check (payment_type in ('balance', 'purchase', 'free', 'cash')),
  purchase_id   uuid references public.purchases(id),   -- ⚠️ purchases уже существует
  minutes_spent int,
  cancelled_at  timestamptz,
  cancel_reason text,
  created_at    timestamptz default now(),

  unique(slot_id, student_id)
);

create index idx_bookings_student on public.bookings(student_id);
create index idx_bookings_slot on public.bookings(slot_id);
create index idx_bookings_status on public.bookings(status);
```

### 10. notifications
```sql
create table public.notifications (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid not null references public.profiles(id),
  booking_id  uuid references public.bookings(id),
  type        text not null check (type in ('sms', 'push')),
  event       text not null,
  -- event values: reminder_24h | reminder_2h | booking_confirmed |
  --               booking_cancelled | lesson_transferred | welcome | otp
  status      text not null default 'pending'
              check (status in ('pending', 'sent', 'failed')),
  content     text not null,
  send_at     timestamptz,            -- когда отправить (null = немедленно)
  sent_at     timestamptz,
  error       text,
  created_at  timestamptz default now()
);

-- Индекс для cron: быстрый поиск pending уведомлений по времени
create index idx_notifications_pending 
on public.notifications(send_at) 
where status = 'pending';
```

### 11. teacher_availability
```sql
create table public.teacher_availability (
  id            uuid primary key default gen_random_uuid(),
  teacher_id    uuid not null references public.teachers(id) on delete cascade,
  day_of_week   int not null check (day_of_week between 0 and 6),
  -- 0=воскресенье, 1=понедельник, ..., 6=суббота
  start_time    time not null,
  end_time      time not null,
  is_active     boolean not null default true,

  constraint availability_valid_time check (end_time > start_time)
);
```

### 12. transfer_requests
```sql
create table public.transfer_requests (
  id              uuid primary key default gen_random_uuid(),
  slot_id         uuid not null references public.slots(id),
  teacher_id      uuid not null references public.teachers(id),
  reason          text not null,
  preferred_dates timestamptz[],
  status          text not null default 'pending'
                  check (status in ('pending', 'approved', 'rejected')),
  admin_note      text,
  created_at      timestamptz default now()
);
```

### 13. otp_codes (для SMS-авторизации)
```sql
create table public.otp_codes (
  id          uuid primary key default gen_random_uuid(),
  phone       text not null,
  code        text not null,
  expires_at  timestamptz not null default (now() + interval '5 minutes'),
  used        boolean not null default false,
  created_at  timestamptz default now()
);

create index idx_otp_phone_expires on public.otp_codes(phone, expires_at)
  where used = false;

-- Автоочистка: удаляем использованные и старые коды
create or replace function cleanup_otp_codes() returns void language sql as $$
  delete from public.otp_codes 
  where used = true or expires_at < now() - interval '1 hour';
$$;
```

### 14. auth_logs
```sql
create table public.auth_logs (
  id          uuid primary key default gen_random_uuid(),
  phone       text not null,
  event       text not null check (event in ('otp_sent', 'otp_verified', 'otp_failed', 'login', 'logout')),
  ip          text,
  user_agent  text,
  created_at  timestamptz default now()
);

create index idx_auth_logs_phone on public.auth_logs(phone, created_at);
```

### 15. push_subscriptions
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

---

## RLS политики

```sql
-- Включаем RLS на всех пользовательских таблицах
alter table public.profiles enable row level security;
alter table public.students enable row level security;
alter table public.bookings enable row level security;
alter table public.payments enable row level security;
alter table public.purchases enable row level security;
alter table public.notifications enable row level security;
alter table public.push_subscriptions enable row level security;

-- Профили: видишь только свой
create policy "profiles_select_own" on public.profiles
  for select using (auth.uid() = id);

create policy "profiles_update_own" on public.profiles
  for update using (auth.uid() = id);

-- Admin видит всё (через отдельную политику)
create policy "profiles_admin_all" on public.profiles
  for all using (
    exists (
      select 1 from public.profiles p
      where p.id = auth.uid() and p.role = 'admin'
    )
  );

-- Записи: студент видит свои
create policy "bookings_student_select" on public.bookings
  for select using (
    student_id in (
      select id from public.students where profile_id = auth.uid()
    )
  );

-- Записи: преподаватель видит записи на свои слоты
create policy "bookings_teacher_select" on public.bookings
  for select using (
    slot_id in (
      select s.id from public.slots s
      join public.teachers t on t.id = s.teacher_id
      where t.profile_id = auth.uid()
    )
  );

-- Платежи: только свои
create policy "payments_student_select" on public.payments
  for select using (
    student_id in (
      select id from public.students where profile_id = auth.uid()
    )
  );
```

---

## Транзакционная функция: запись на урок

```sql
create or replace function public.book_slot(
  p_slot_id     uuid,
  p_student_id  uuid,
  p_payment_type text,
  p_purchase_id uuid default null
) returns uuid
language plpgsql security definer as $$
declare
  v_booking_id  uuid;
  v_slot        record;
  v_duration_min int;
  v_booked_count int;
begin
  -- Блокируем слот для предотвращения race condition
  select * into v_slot from public.slots 
  where id = p_slot_id for update;

  if v_slot.status = 'cancelled' then
    raise exception 'Слот отменён';
  end if;

  -- Считаем текущее количество записей
  select count(*) into v_booked_count
  from public.bookings
  where slot_id = p_slot_id and status not in ('cancelled');

  if v_booked_count >= v_slot.max_students then
    raise exception 'Слот заполнен';
  end if;

  v_duration_min := extract(epoch from (v_slot.ends_at - v_slot.starts_at)) / 60;

  -- Создаём запись
  insert into public.bookings (slot_id, student_id, status, payment_type, purchase_id)
  values (p_slot_id, p_student_id, 'confirmed', p_payment_type, p_purchase_id)
  returning id into v_booking_id;

  -- Списываем минуты
  if p_payment_type = 'balance' then
    update public.students
    set balance_minutes = balance_minutes - v_duration_min
    where id = p_student_id
    and balance_minutes >= v_duration_min;

    if not found then
      raise exception 'Недостаточно минут на балансе';
    end if;

  elsif p_payment_type = 'purchase' and p_purchase_id is not null then
    update public.purchases
    set minutes_used = minutes_used + v_duration_min
    where id = p_purchase_id
    and (minutes_total - minutes_used) >= v_duration_min
    and (expires_at is null or expires_at > now());

    if not found then
      raise exception 'Недостаточно минут в пакете или пакет истёк';
    end if;
  end if;

  -- Закрываем слот если он на одного
  if v_slot.max_students = 1 then
    update public.slots set status = 'booked' where id = p_slot_id;
  end if;

  return v_booking_id;
end;
$$;
```

---

## Типы TypeScript

Генерировать автоматически через Supabase CLI после каждой миграции:
```bash
npx supabase gen types typescript --project-id znuquyllsrbpijbgymih > src/types/database.ts
```

---

## Миграции

- Папка: `supabase/migrations/`
- Формат имени: `YYYYMMDDHHMMSS_короткое_описание.sql`
- Никогда не редактировать уже применённые миграции — только новые
- Тестировать локально: `npx supabase start`
- Применить: `npx supabase db push`
