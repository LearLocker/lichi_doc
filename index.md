---
title: "Инструкция по типовым проблемам интеграции с маркетплейсами"
description: "Руководство по решению стандартных ошибок интеграции с маркетплейсами (Ozon, Lamoda, Kaspi, Namshi, CDEK)"
layout: default
---

# 🧾 Инструкция по типовым проблемам интеграции с маркетплейсами

Документ описывает основные сценарии устранения типовых ошибок интеграции между маркетплейсами и CRM-системой.

---

## 📦 1. Проблема: Выгрузить **не существующий заказ**

### 🧩 Описание
Необходимо выгрузить заказ, которого нет в локальной базе интеграции (отсутствует или выгружен некорректно).

### ✅ Проверка
1. Откройте **локальную базу интеграции** с маркетплейсом.  
2. Найдите запись по идентификатору заказа:
   - Если **записи нет вообще** → выгрузите заказ через **внутренний номер заказа маркетплейса**.
   - Если **запись есть**, но поле `bitrix_order_id` имеет значение `not_sent` или `send_error_bitrix` → выгрузите заказ по **номеру отправления**.

---

### ⚙️ Методы для выгрузки

| Маркетплейс | Метод API | Сценарий |
|--------------|------------|-----------|
| **Ozon** | `api/order/update_order` | Через внутренний номер (если записи нет) или по номеру отправления (если запись есть, но не выгружена) |

#### 🔧 Примеры запросов Ozon

**Если есть `posting_number`:**
```http
POST http://ozon.spb.lichishop.com/api/order/update_order
Content-Type: application/json

{
  "id": "0145451723-0102-1",
  "name": "posting_number"
}
```

**Если записи нет:**
```http
POST http://ozon.spb.lichishop.com/api/order/update_order
Content-Type: application/json

{
  "id": "32520914509",
  "name": "ozon_order_id"
}
```

---

| **Lamoda** | `upload_order` | Аналогично Ozon |

#### 🔧 Примеры запросов Lamoda

**Если есть `posting_number`:**
```http
POST https://lamodakz-dev.spb.lichishop.com/api/order/update_order
Content-Type: multipart/form-data

lamoda_order_number=KZ250909-140244
```

**Если записи нет:**
```http
POST https://lamodakz-dev.spb.lichishop.com/api/order/update_order
Content-Type: multipart/form-data

lamoda_order_id=22372314
```

---

| **Kaspi** | `upload-to-lichi` | Выгрузка заказа с сервера маркетплейса |

```http
GET https://kaspi-api.lichi.one/api/orders/upload-to-lichi/{kaspi_order_code}
```
`{kaspi_order_code}` — код заказа, например: `690748450`

---

| **Namshi** | `notifyOrder` | Отправка уведомления о заказе для создания записи в CRM |

```http
POST https://namshi-api.lichi.one/api/order/notify
Content-Type: application/json

{
  "order_nr": "NMAEC6DR0E6519BQ"
}
```

---

## 🧾 2. Проблема: Выгрузить **существующий заказ**

### 🧩 Описание
В локальной базе маркетплейса уже есть запись с `bitrix_order_id`, но **в CRM не создана сделка**.

### 🔍 Возможная причина
Ошибка при оплате или сбой синхронизации статуса заказа.

---

### ⚙️ Решение

1. Проверьте статус заказа и при необходимости **повторно вызовите хук платежки**:
   - Используйте метод **`change_order_status`** для повторной отправки статуса заказа в CRM.

**Для Ozon и Lamoda:**
```http
POST http://{domain}/api/order/change_order_status
Content-Type: application/json

{
  "order_ids": [
    "3339442",
    "3339443",
    "3339444",
    "3339445",
    "3339446"
  ]
}
```

**Для Kaspi:**
```http
POST https://kaspi-api.lichi.one/api/internal/confirm-payment
Content-Type: application/json

{
  "lichi_order_ids": [
    "3339442",
    "3339443",
    "3339444",
    "3339445",
    "3339446"
  ]
}
```

---

**Для Namshi:**  
После вызова `change_order_status` выполните:
```http
GET https://192.168.217.44/?do=order-status&id=3661702&pay=89&shop=ae&lang=en
```

- Подождите ~5 минут — заказ может синхронизироваться автоматически.  
- Если нет — выполните синхронизацию вручную:
  1. Откройте заказ в CRM → вкладка **«Сделки»**.  
  2. Перейдите на вкладку **«Проверка наличия»**.  
  3. После успешной синхронизации вызовите `appendcomment`:

```http
GET https://namshi-api.lichi.one/api/order/3753173/comment
```

---

## 🛍️ 3. Проблема: **Выгрузка каталога Lamoda**

### 🧩 Описание
Требуется перенести каталог товаров с **RU-версии Lamoda** на **KZ-версию**.

---

### ⚙️ Решение

#### 1. Вызов метода
```http
POST https://lamodakz-dev.spb.lichishop.com/api/product/save_csv
Content-Type: multipart/form-data

file={file}
headers=xlsx
separator=,
```

#### 2. Параметры

| Параметр | Описание |
|-----------|-----------|
| `headers` | Принимает `xlsx` или `csv` — зависит от формата файла. |
| `separator` | Символ-разделитель (используется только для CSV). |

#### 3. Ответ Lamoda

```xml
<?xml version="1.0" encoding="UTF-8"?>
<SuccessResponse>
  <Head>
    <RequestId>cb106552-87f3-450b-aa8b-412246a24b34</RequestId>
    <RequestAction>ProductCreate</RequestAction>
    <ResponseType/>
    <Timestamp>2016-06-22T04:40:14+0200</Timestamp>
  </Head>
  <Body/>
</SuccessResponse>
```

#### 4. Проверка статуса импорта

По `RequestId` из ответа можно проверить статус импорта методом **`FeedStatus`**.  
Удобно использовать [Lamoda SellerCenter API Explorer](https://sellercenter.lamoda.kz/api-explorer) — API-ключ и User ID уже предзаполнены.

---

## 🚚 4. Проблема: OZON ↔ CDEK Integration

### 🧩 Описание
Иногда для заказа, полученного с **Ozon**, не создаётся накладная в **СДЭК**.

---

### ⚙️ Проверка
1. Найдите идентификатор отправления в формате `OZ-{ozon_posting_number}`.  
2. Проверьте в [личном кабинете CDEK](https://lk.cdek.ru/order-history), создана ли заявка на доставку.

---

### 🚨 Если заявка **не создана**
- Напишите в **чат поддержки CDEK (Ozon × Lichi)** с просьбой создать заказ вручную.
- Укажите идентификатор `OZ-{ozon_posting_number}`.

---

### 🧰 Диагностика ошибок

Ошибки доставки хранятся в базе **`logistic_v2`**.

| Таблица | Назначение |
|----------|-------------|
| `outbox` | События интеграции (отправка данных в службы доставки) |
| `outbox_errors` | Ошибки обработки событий из `outbox` |

#### Шаги проверки
1. Найдите запись в `outbox` по `identifier`:  
   - `ProcessDeliveryJob:Package:{package_id}`  
   - `GenerateDeliveryDocumentsJob:Package:{package_id}`
2. В таблице `outbox_errors` найдите ошибки с тем же `outbox_id`.

> 💬 Частая ошибка — `v2_entity_not_found_im_number`, связана с несинхронизированными идентификаторами между Ozon и CDEK.

---

## 📘 Итоговая таблица

| № | Проблема | Описание | Решение |
|---|-----------|-----------|----------|
| 1 | Не существующий заказ | Отсутствует в локальной БД интеграции | Проверить БД → вызвать `upload_order` / `notifyOrder` |
| 2 | Существующий заказ без сделки | В CRM не создана сделка | Проверить оплату → вызвать `change_order_status` → (Namshi — `appendcomment`) |
| 3 | Выгрузка каталога Lamoda | Перенос каталога с RU на KZ | `save_csv` → проверить `FeedStatus` |
| 4 | OZON ↔ CDEK | Отсутствует накладная | Проверить заявку в CDEK → при отсутствии — написать в чат → анализ `outbox` |

---

© 2025 · Интеграционная команда **Lichi**  
Документ предназначен для внутренних нужд службы поддержки и разработчиков интеграций.
