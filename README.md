# 🏨 Hotel Revenue Analytics Project

> **Комплексный анализ выручки отеля с расчётом чистой прибыли, отмен бронирований и эффективности каналов продаж**

---

## 📋 Содержание

1. [О проекте](#-о-проекте)
2. [Структура данных](#-структура-данных)
3. [Ключевые метрики (KPIs)](#-ключевые-метрики-kpis)
4. [Установка и запуск](#-установка-и-запуск)
5. [Аналитические запросы](#-аналитические-запросы)
6. [Инсайты и рекомендации](#-инсайты-и-рекомендации)
7. [Структура репозитория](#-структура-репозитория)

---

## 📊 О проекте

Этот проект представляет собой полный цикл аналитики для отельного бизнеса на основе данных за 3 года (2018–2020). Основная цель — оценка показателей бизнеса на основе выручки, количества бронирований, среднего чека с учётом:

- ✅ Комиссий каналов продаж  
- ✅ Выручки от питания (F&B)  
- ✅ Процента отмен бронирований  
- ✅ Сегментации клиентов  

**Cтек:**
- `MS SQL Server`

---
## Создание базы данных в MS SQL Server 
В первую очередь, на сервере была создана новая база данных "Project", куда данные импортировались из Excel файла

<img width="333" height="329" alt="image" src="https://github.com/user-attachments/assets/b3f1d113-9dd8-44bc-a6c8-b762fce80d7c" />



## 🗄️ Структура данных

### Основные таблицы

| Таблица | Описание | Записей |
|---------|----------|---------|
| `sale_2018` | Данные о бронированиях за 2018 год | ~22 000 |
| `sale_2019` | Данные о бронированиях за 2019 год | ~79 000 |
| `sale_2020` | Данные о бронированиях за 2020 год | ~40 000 |
| `market_segment` | Справочник комиссий по сегментам | 8 |
| `meal_cost` | Справочник стоимости питания | 5 |

### Схема данных `#all_sales`

```sql
CREATE TABLE #all_sales (
    hotel                       VARCHAR(20),
    is_canceled                 INT,
    lead_time                   INT,
    arrival_date_year           INT,
    arrival_date_month          VARCHAR(10),
    arrival_date_week_number    INT,
    arrival_date_day_of_month   INT,
    stays_in_weekend_nights     INT,
    stays_in_week_nights        INT,
    adults                      INT,
    children                    INT,
    babies                      INT,
    meal                        VARCHAR(10),
    country                     VARCHAR(10),
    market_segment              VARCHAR(20),
    distribution_channel        VARCHAR(20),
    is_repeated_guest           INT,
    previous_cancellations      INT,
    previous_bookings_not_canceled INT,
    reserved_room_type          VARCHAR(1),
    assigned_room_type          VARCHAR(1),
    booking_changes             INT,
    deposit_type                VARCHAR(10),
    agent                       INT,
    company                     INT,
    days_in_waiting_list        INT,
    customer_type               VARCHAR(20),
    adr                         DECIMAL(10,2),
    required_car_parking_spaces INT,
    total_of_special_requests   INT,
    reservation_status          VARCHAR(10),
    reservation_status_date     DATETIME
);
```

### Справочники

**`market_segment`** — комиссии каналов продаж:

| market_segment | Discount |
|----------------|----------|
| Direct | 0.10 |
| Corporate | 0.15 |
| Online TA | 0.30 |
| Offline TA/TO | 0.30 |
| Complementary | 1.00 |

**`meal_cost`** — стоимость питания на человека:

| meal | Cost |
|------|------|
| BB | 12.99 |
| HB | 17.99 |
| FB | 21.99 |
| SC | 35.00 |

---

## 📈 Ключевые метрики (KPIs)

| Метрика | Формула | Описание |
|---------|---------|----------|
| **Cancellation Rate** | `Отменённые / Всего × 100` | Процент отмен бронирований |
| **Gross Revenue** | `ADR × Ночи` | Валовая выручка номерного фонда |
| **Net Revenue** | `Gross − Комиссия + F&B` | Чистая выручка отеля |
| **ADR** | `Выручка / Ночи` | Средняя цена за ночь |
| **ALOS** | `Ночи / Бронирования` | Средняя длительность проживания |
| **Net Retention Ratio** | `Net / Gross` | Доля сохранённой выручки |

---

## 🚀 Установка и запуск

### 1. Подготовка данных

Из-за того, что данные были импортированы из Excel, было решено привести их к более удобным типам данных, для этого использовался Alter table. 
После этого, нужно было привести строковые значения "Null" в тип данных Null. 
```sql
USE [Project];

-- Приведение типов данных (для всех таблиц 2018–2020)
ALTER TABLE sale_2018 ALTER COLUMN hotel VARCHAR(20);
ALTER TABLE sale_2018 ALTER COLUMN is_canceled INT;
-- ... (остальные поля)

-- Очистка NULL-значений из Excel
UPDATE sale_2018 SET company = NULLIF(company, 'NULL');
-- ... (остальные поля)
```

### 2. Объединение данных за 3 года

```sql
SELECT * INTO #all_sales FROM (
    SELECT * FROM sale_2018 UNION ALL
    SELECT * FROM sale_2019 UNION ALL
    SELECT * FROM sale_2020
) AS t;
```

### 3. Запуск аналитических запросов

Все запросы находятся в файле [`queries.sql`](./queries.sql). Выполняйте последовательно по секциям.

---

## 🔍 Аналитические запросы

### 1. Динамика метрик отеля по годам (YoY)

```sql
WITH YearlyMetrics AS (
    SELECT 
        arrival_date_year AS year,
        COUNT(*) AS total_bookings,
        SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS canceled_bookings,
        ROUND(CAST(SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS DECIMAL(10,2)) / COUNT(*) * 100, 0) AS cancellation_rate_pct,
        SUM(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) * adr ELSE 0 END) AS total_revenue,
        ROUND(AVG(CASE WHEN is_canceled = 0 THEN adr ELSE NULL END), 0) AS avg_adr,
        ROUND(AVG(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) ELSE NULL END), 0) AS avg_length_of_stay
    FROM #all_sales
    GROUP BY arrival_date_year
)
SELECT 
    year,
    total_bookings,
    canceled_bookings,
    cancellation_rate_pct,
    ROUND(total_revenue, 0) AS total_revenue,
    avg_adr,
    avg_length_of_stay,
    -- Выручка прошлого года (LAG)
    ROUND(LAG(total_revenue) OVER (ORDER BY year), 0) AS prev_year_revenue,
    -- Абсолютное изменение
    ROUND(total_revenue - LAG(total_revenue) OVER (ORDER BY year), 0) AS revenue_change_abs,
    -- Процентное изменение YoY (ключевая метрика)
    ROUND(
        (total_revenue - LAG(total_revenue) OVER (ORDER BY year)) * 100.0 / 
        NULLIF(LAG(total_revenue) OVER (ORDER BY year), 0), 
        2
    ) AS revenue_change_yoy_pct
FROM YearlyMetrics
ORDER BY year;
```

<img width="1040" height="114" alt="image" src="https://github.com/user-attachments/assets/2aac2026-0445-4318-ba2f-86b94973392f" />

Выводы: 
1. Процент отмен на протяжении трех лет стабильно высокий - 37% 
2. В 2019 году выручка резко поднялась на 259%, однако в следующего году упала на 39%
3. Средняя стоимость за ночь за 3 года выросла с 89 до 111 $.  


### 2. Динамика метрик по типам отелей

```sql
Select arrival_date_year, hotel as type_hotel, 
        COUNT(*) AS total_bookings,
        SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS canceled_bookings,
        ROUND(CAST(SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS DECIMAL(10,2)) / COUNT(*) * 100, 0) AS cancellation_rate_pct,
        SUM(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) * adr ELSE 0 END) AS total_revenue,
        ROUND(AVG(CASE WHEN is_canceled = 0 THEN adr ELSE NULL END), 0) AS avg_adr,
        ROUND(AVG(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) ELSE NULL END), 0) AS avg_length_of_stay
FROM #all_sales
GROUP BY arrival_date_year,hotel
ORDER BY arrival_date_year
```

<img width="788" height="145" alt="image" src="https://github.com/user-attachments/assets/9de75ae3-6a36-4db9-a302-143adb154a74" />

Выводы: 
1. Основная категория отелей в доле выручки и бронирований - отели в черте города.
2. У отелей в черте города процент отмен выше почти в 2 раза, чем у resort отелей.
3. Средняя стоимость за ночь и среднее количество дней отдыха выше у resort отелей.

### 3. Чистая выручка по сегментам (Net Retention)

```sql
WITH SegmentFinancials AS (
    SELECT 
        s.market_segment,
        s.is_canceled,
        CAST(s.adr AS DECIMAL(18,4)) * (s.stays_in_weekend_nights + s.stays_in_week_nights) AS gross_rev,
        CAST(s.adr AS DECIMAL(18,4)) * (s.stays_in_weekend_nights + s.stays_in_week_nights) * ISNULL(ms.Discount, 0) AS commission,
        ISNULL(mc.Cost, 0) * (s.adults + s.children) * (s.stays_in_weekend_nights + s.stays_in_week_nights) AS fb_rev
    FROM #all_sales s
    LEFT JOIN market_segment ms ON s.market_segment = ms.market_segment
    LEFT JOIN meal_cost mc ON s.meal = mc.meal
)
SELECT 
    market_segment,
    SUM(CASE WHEN is_canceled = 0 THEN gross_rev ELSE 0 END) AS total_gross_revenue,
    SUM(CASE WHEN is_canceled = 0 THEN (gross_rev - commission + fb_rev) ELSE 0 END) AS total_net_revenue,
    -- Коэффициент сохранения выручки (Net / Gross)
    -- Используем DECIMAL(18,6) и NULLIF для защиты от деления на ноль
    CAST(
        SUM(CASE WHEN is_canceled = 0 THEN (gross_rev - commission + fb_rev) ELSE 0 END) AS DECIMAL(18,6)
    ) / NULLIF(
        CAST(SUM(CASE WHEN is_canceled = 0 THEN gross_rev ELSE 0 END) AS DECIMAL(18,6)), 
        0
    ) AS net_retention_ratio
FROM SegmentFinancials
Where market_segment <> 'Complementary'
GROUP BY market_segment
ORDER BY net_retention_ratio DESC;
```
<img width="488" height="167" alt="image" src="https://github.com/user-attachments/assets/4171ef8b-6178-473d-b70a-715ac6cce1b3" />

Выводы: 
1. Основная категория отелей в доле выручки и бронирований - отели в черте города.
2. У отелей в черте города процент отмен выше почти в 2 раза, чем у resort отелей.
3. Средняя стоимость за ночь и среднее количество дней отдыха выше у resort отелей.

### 4. Сезонность по месяцам

```sql
WITH MonthlyStats AS(
    SELECT 
        
        arrival_date_month,
        -- Маппинг месяцев для сортировки
        CASE arrival_date_month
            WHEN 'January' THEN 1 WHEN 'February' THEN 2 WHEN 'March' THEN 3
            WHEN 'April' THEN 4 WHEN 'May' THEN 5 WHEN 'June' THEN 6
            WHEN 'July' THEN 7 WHEN 'August' THEN 8 WHEN 'September' THEN 9
            WHEN 'October' THEN 10 WHEN 'November' THEN 11 WHEN 'December' THEN 12
        END AS month_num,
        COUNT(*) AS bookings,
        SUM(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) * adr ELSE 0 END) AS revenue,
        CAST(SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS DECIMAL(10,2)) / COUNT(*) * 100 AS cancel_rate
    FROM #all_sales
    GROUP BY  arrival_date_month
)
SELECT 
    arrival_date_month,
    bookings,
    revenue,
    cancel_rate
FROM MonthlyStats
ORDER BY month_num;
```
<img width="359" height="248" alt="image" src="https://github.com/user-attachments/assets/932f5b56-d863-4073-b50d-3e6189689a8e" />

Выводы: 
1. Пик бронирований июль-октябрь 
2. Наибольший процент отмен в апреле и июне. 

### 5. Анализ отмен по типу питания

```sql
WITH Meals_year AS(
SELECT arrival_date_year,#all_sales.meal as meal, round(sum(Cost),2) as meal_sum
FROM #all_sales
LEFT JOIN meal_cost ON #all_sales.meal = meal_cost.meal
WHERE #all_sales.meal <> 'Undefined'
GROUP BY arrival_date_year,#all_sales.meal)

SELECT 
    arrival_date_year,
    meal,
    meal_sum,
    sum(meal_sum) over (partition by arrival_date_year, meal order by arrival_date_year)/
    sum(meal_sum) over (partition by arrival_date_year order by arrival_date_year) as meal_percentage
FROM Meals_year
```
<img width="335" height="249" alt="image" src="https://github.com/user-attachments/assets/884ad4a5-4ebd-456f-93ad-8bc20d865589" />

Выводы: 
1. BB (Bed and Breakfast) - занимает наибольшую долю в выручке, однако его процент от общей выручки снизился с 71% до 60%
2. SC (Self catering) - занимал всего 4% в 2018 году, но вырос до 27% в 2020 году.

### 6. Сравнение новых и постоянных клиентов
```sql
SELECT arrival_date_year,
    CASE WHEN is_repeated_guest = 1 THEN 'Repeated Guest' ELSE 'New Guest' END AS guest_type,
    COUNT(*) AS total_bookings,
    CAST(SUM(CASE WHEN is_canceled = 1 THEN 1 ELSE 0 END) AS DECIMAL(10,2)) / COUNT(*) * 100 AS cancellation_rate_pct,
    SUM(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) * adr ELSE 0 END) AS total_revenue,
    AVG(CASE WHEN is_canceled = 0 THEN (stays_in_weekend_nights + stays_in_week_nights) * adr ELSE NULL END) AS avg_revenue_per_booking
FROM #all_sales
GROUP BY arrival_date_year,is_repeated_guest
ORDER BY arrival_date_year;
```
<img width="633" height="145" alt="image" src="https://github.com/user-attachments/assets/0d0c1a31-9272-465f-bd66-60b7d8e37f99" />

Выводы: 
1. Постоянные клиенты стали гораздо реже отменяют бронирование (с 54% до 4%)
2. Постоянные клиенты отменяют бронирование реже, чем новые
3. Доля постоянных клиентов в выручке очень мала

---

## 💡 Инсайты и рекомендации

### 🔴 Проблемы

| Проблема | Метрика | Влияние |
|----------|---------|---------|
| Высокий процент отмен | ~37% | Потеря выручки |
| Высокие комиссии OTA | 30% | Снижение маржи |
| Низкая доля повторных гостей | < 10% | Упущенная LTV |

### 🟢 Возможности

| Возможность | Действие | Ожидаемый эффект |
|-------------|----------|------------------|
| Стимулирование Direct-бронирований | Программа лояльности | +15% Net Revenue |
| Пакеты с питанием (FB/HB) | Скидка 5% при невозвратном тарифе | −10% отмен |
| Динамическое ценообразование | ADR ↑ в пик сезона | +20% выручка |

### 📊 Ключевые выводы

1. **Каналы с высокой комиссией** (Online TA, Offline TA) требуют пересмотра условий контрактов.
2. **Повторные гости** имеют на 40% меньший процент отмен — приоритет для маркетинга.
3. **Питание** снижает вероятность отмены брони — стимулировать продажу пакетов.
4. **Сезонность** выражена — в низкий сезон запускать акции для заполнения номерного фонда.

---

## 📁 Структура репозитория

```
hotel-revenue-analytics/
├── README.md                 # Этот файл
├── queries.sql               # Все SQL-запросы
├── data/
│   ├── hotel_revenue_historical_full.xlsx        # Исходные данные
├── docs/
│   ├── schema.md            # Описание схемы БД
│   └── kpis.md              # Описание метрик
└── reports/
    └── 
```

---

## 📞 Контакты

**Автор:** Оганес Казарян
**Email:**  


**⭐ Если проект был полезен, поставьте звезду!**


