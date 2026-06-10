---
name: tramplin-payments
description: Интеграция с ЮKassa для tramplin.pro. Читай этот файл когда нужно: принять оплату, обработать webhook от ЮKassa, создать платёж, проверить статус, вернуть деньги, работать с балансом студента.
---

# Skill: Оплата (ЮKassa)

Всегда читай `AGENT.md` перед этим файлом. Схему таблиц payments и purchases см. в `skill-database.md`.

---

## Переменные окружения

```env
YUKASSA_SHOP_ID=ваш_shop_id
YUKASSA_SECRET_KEY=секретный_ключ
YUKASSA_WEBHOOK_SECRET=секрет_для_проверки_webhook
NEXT_PUBLIC_BASE_URL=https://tramplin.pro
```

---

## Клиент ЮKassa

```typescript
// src/lib/yukassa/client.ts

const YUKASSA_API = 'https://api.yookassa.ru/v3'

interface CreatePaymentParams {
  amount: number          // в рублях
  description: string
  returnUrl: string
  metadata?: Record<string, string>
  idempotencyKey: string
}

interface YukassaPayment {
  id: string
  status: 'pending' | 'waiting_for_capture' | 'succeeded' | 'cancelled'
  confirmation: {
    type: string
    confirmation_url: string
  }
  amount: { value: string; currency: string }
  metadata?: Record<string, string>
}

export async function createPayment(params: CreatePaymentParams): Promise<YukassaPayment> {
  const credentials = Buffer.from(
    `${process.env.YUKASSA_SHOP_ID}:${process.env.YUKASSA_SECRET_KEY}`
  ).toString('base64')

  const res = await fetch(`${YUKASSA_API}/payments`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Basic ${credentials}`,
      'Idempotence-Key': params.idempotencyKey,
    },
    body: JSON.stringify({
      amount: {
        value: params.amount.toFixed(2),
        currency: 'RUB',
      },
      confirmation: {
        type: 'redirect',
        return_url: params.returnUrl,
      },
      capture: true,  // Автоматическое списание
      description: params.description,
      metadata: params.metadata,
    }),
  })

  if (!res.ok) {
    const error = await res.json()
    throw new Error(`ЮKassa ошибка: ${error.description}`)
  }

  return res.json()
}

export async function getPayment(yukassaPaymentId: string): Promise<YukassaPayment> {
  const credentials = Buffer.from(
    `${process.env.YUKASSA_SHOP_ID}:${process.env.YUKASSA_SECRET_KEY}`
  ).toString('base64')

  const res = await fetch(`${YUKASSA_API}/payments/${yukassaPaymentId}`, {
    headers: { 'Authorization': `Basic ${credentials}` },
  })

  return res.json()
}
```

---

## API Routes

### POST /api/payments/create
Инициировать оплату курса или пакета часов.

```typescript
// src/app/api/payments/create/route.ts
import { createClient } from '@/lib/supabase/server'
import { createPayment } from '@/lib/yukassa/client'
import { z } from 'zod'
import { randomUUID } from 'crypto'

const schema = z.object({
  course_id: z.string().uuid(),
})

export async function POST(request: Request) {
  const supabase = createClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return Response.json({ error: 'Не авторизован' }, { status: 401 })

  const { course_id } = schema.parse(await request.json())

  // Получаем информацию о курсе
  const { data: course } = await supabase
    .from('courses')
    .select('*')
    .eq('id', course_id)
    .eq('is_active', true)
    .single()

  if (!course) return Response.json({ error: 'Курс не найден' }, { status: 404 })

  const { data: student } = await supabase
    .from('students')
    .select('id')
    .eq('profile_id', user.id)
    .single()

  const idempotencyKey = randomUUID()

  // Создаём платёж в нашей БД (статус pending)
  const { data: payment } = await supabase
    .from('payments')
    .insert({
      student_id: student!.id,
      amount: course.price,
      status: 'pending',
      description: `Покупка: ${course.name}`,
      idempotency_key: idempotencyKey,
      metadata: { course_id, student_id: student!.id },
    })
    .select()
    .single()

  // Создаём платёж в ЮKassa
  const yukassaPayment = await createPayment({
    amount: course.price,
    description: `Трамплин: ${course.name}`,
    returnUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/payment/result?payment_id=${payment!.id}`,
    metadata: { 
      internal_payment_id: payment!.id,
      course_id,
      student_id: student!.id,
    },
    idempotencyKey,
  })

  // Сохраняем ID от ЮKassa
  await supabase.from('payments').update({
    yukassa_payment_id: yukassaPayment.id,
    yukassa_payment_url: yukassaPayment.confirmation.confirmation_url,
  }).eq('id', payment!.id)

  return Response.json({
    payment_url: yukassaPayment.confirmation.confirmation_url,
  })
}
```

### POST /api/payments/webhook
Обработка вебхука от ЮKassa.

```typescript
// src/app/api/payments/webhook/route.ts
import crypto from 'crypto'

export async function POST(request: Request) {
  const body = await request.text()
  
  // Проверяем подпись (ЮKassa отправляет IP-фильтрованные запросы,
  // дополнительно можно проверить через shared secret в metadata)
  
  const event = JSON.parse(body)
  
  if (event.type !== 'payment.succeeded' && event.type !== 'payment.cancelled') {
    return Response.json({ ok: true })
  }

  const supabase = createAdminClient()  // service_role для обхода RLS
  const yukassaPayment = event.object

  // Находим наш платёж
  const { data: payment } = await supabase
    .from('payments')
    .select('*')
    .eq('yukassa_payment_id', yukassaPayment.id)
    .single()

  if (!payment) {
    console.error('Webhook: платёж не найден', yukassaPayment.id)
    return Response.json({ ok: true })
  }

  if (event.type === 'payment.succeeded') {
    await handlePaymentSucceeded(supabase, payment, yukassaPayment)
  } else if (event.type === 'payment.cancelled') {
    await supabase.from('payments').update({ status: 'cancelled' }).eq('id', payment.id)
  }

  return Response.json({ ok: true })
}

async function handlePaymentSucceeded(supabase: any, payment: any, yukassaPayment: any) {
  // 1. Обновляем статус платежа
  await supabase.from('payments')
    .update({ status: 'succeeded', updated_at: new Date().toISOString() })
    .eq('id', payment.id)

  const metadata = payment.metadata as { course_id?: string; student_id: string }

  // 2. Создаём покупку
  if (metadata.course_id) {
    const { data: course } = await supabase
      .from('courses')
      .select('*')
      .eq('id', metadata.course_id)
      .single()

    await supabase.from('purchases').insert({
      student_id: metadata.student_id,
      course_id: metadata.course_id,
      minutes_total: course.total_minutes,
      price_paid: payment.amount,
      payment_id: payment.id,
      expires_at: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000).toISOString(), // 1 год
    })
  }

  // 3. Отправить уведомление студенту
  // (см. skill-notifications.md)
}
```

---

## Страница результата платежа

```typescript
// src/app/payment/result/page.tsx
// ЮKassa редиректит сюда после оплаты с ?payment_id=...

export default async function PaymentResultPage({
  searchParams,
}: {
  searchParams: { payment_id: string }
}) {
  const supabase = createServerClient()
  
  const { data: payment } = await supabase
    .from('payments')
    .select('status, description, amount')
    .eq('id', searchParams.payment_id)
    .single()

  // Статус может быть ещё pending — ЮKassa присылает webhook асинхронно
  // Делаем polling или просим подождать
  
  if (payment?.status === 'succeeded') {
    return <PaymentSuccess amount={payment.amount} description={payment.description} />
  }
  
  return <PaymentPending paymentId={searchParams.payment_id} />
}
```

---

## Безопасность

1. **Idempotency Key** — всегда передавать, чтобы при ретрае не создался дублирующий платёж
2. **Webhook** — ЮKassa шлёт с фиксированных IP (192.168.0.0/24 и тп) — настроить whitelist на уровне инфраструктуры
3. **service_role только в webhook** — никогда не передавать service_role на клиент
4. **Не доверять клиенту** — сумму платежа всегда брать из БД, не из тела запроса

---

## Возврат средств

```typescript
export async function createRefund(yukassaPaymentId: string, amount?: number) {
  const credentials = Buffer.from(
    `${process.env.YUKASSA_SHOP_ID}:${process.env.YUKASSA_SECRET_KEY}`
  ).toString('base64')

  // amount = null → полный возврат
  const body: any = { payment_id: yukassaPaymentId }
  if (amount) {
    body.amount = { value: amount.toFixed(2), currency: 'RUB' }
  }

  const res = await fetch('https://api.yookassa.ru/v3/refunds', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Basic ${credentials}`,
      'Idempotence-Key': randomUUID(),
    },
    body: JSON.stringify(body),
  })

  return res.json()
}
```

Возвраты инициирует только **admin** через интерфейс админки.
