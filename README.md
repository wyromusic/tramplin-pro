# 🤸 tramplin.pro

Платформа для школы батутного спорта — запись на занятия, оплата, личные кабинеты.

## Роли
- **Студент** — запись, оплата, уведомления
- **Преподаватель** — расписание, студенты, посещаемость  
- **Админ** — дашборд, финансы, управление

## Стек
Next.js 14 · TypeScript · Supabase · Tailwind · shadcn/ui · ЮKassa · Web Push · SMS

## Структура `.claude/`
Папка содержит AGENT.md и skill-файлы для разработки с Claude:
- `AGENT.md` — главный контекст проекта
- `skill-database.md` — схема БД и миграции
- `skill-booking.md` — запись на занятия
- `skill-notifications.md` — SMS + Push
- `skill-payments.md` — ЮKassa
- `skill-auth.md` — авторизация и роли
- `skill-admin-dashboard.md` — админ-панель
- `skill-teacher-portal.md` — кабинет преподавателя
