# goit-rdb-hw-05 — Olist: CTE-pipeline, recursive CTE, Materialized View, EXPLAIN

Домашнє завдання 5. Notebook виконує такі кроки:

- будує customer-level feature pipeline на Olist як CTE-chain з 8 stages;
- рахує глибину referral-ланцюжка через recursive CTE із захистом від циклів;
- кешує pipeline у materialized view з `REFRESH ... CONCURRENTLY`;
- порівнює повний перерахунок і читання з MV через `EXPLAIN (ANALYZE, BUFFERS)`.

## Джерело даних

- **Набір:** Brazilian E-Commerce Public Dataset by Olist (замовлення 2016–2018).
- **Kaggle:** https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce (`olistbr/brazilian-ecommerce`).
- **Ліцензія:** CC BY-NC-SA 4.0.

Використовуються 5 файлів: `olist_customers_dataset.csv`, `olist_orders_dataset.csv`, `olist_order_items_dataset.csv`, `olist_products_dataset.csv`, `olist_sellers_dataset.csv`. Вибірка не робиться, завантажуються всі рядки.

## Звідки notebook бере дані

Notebook перебирає джерела по черзі й бере перше доступне. Обраний режим виводиться як `DATA_MODE`.

1. **`data/olist/`**: локальні CSV із курсового архіву або вручну завантажені з Kaggle.
2. **`kagglehub`**: `kagglehub.dataset_download('olistbr/brazilian-ecommerce')`. Якщо з'являється помилка 401/403, задайте `KAGGLE_USERNAME` і `KAGGLE_KEY` (Kaggle → Settings → API → Create New Token). У Colab це можна зробити через Secrets.
3. **Stub-схема з умови завдання**: 30 customers, 50 orders, 100 order_items, детермінований seed без `RANDOM()`. Вмикається автоматично, якщо попередні джерела недоступні. Примусово її можна ввімкнути змінною `HW5_FORCE_STUB=1`.

Усі завдання (CTE, recursive CTE, MV, EXPLAIN) виконуються в обох режимах без змін у коді. На stub-схемі плани `EXPLAIN` простіші, а планувальник може обирати `Seq Scan` навіть за наявності індексу, бо таблиці займають одну сторінку.

## Структура

```
goit-rdb-hw-05/
├── README.md
├── hw5_olist.ipynb
├── .gitignore
└── data/
    └── olist/          # необов'язково: CSV з курсового архіву / Kaggle
```

## Запуск

Потрібен **Python 3.10–3.12**, бо `pgserver` не має збірок для Python 3.13+.

```bash
git clone <repo-url> goit-rdb-hw-05
cd goit-rdb-hw-05
python3.12 -m venv .venv && source .venv/bin/activate   # macOS: brew install python@3.12
pip install jupyter
jupyter notebook hw5_olist.ipynb
```

Далі виконайте **Kernel → Restart & Run All**. Перша комірка сама встановлює `pgserver`, `psycopg2-binary`, `sqlalchemy`, `pandas` і `kagglehub`.

**Colab:** runtime має бути з Python ≤ 3.12 (Runtime → Change runtime type → Runtime version). CSV можна покласти в `/content/data/olist/`.

## Порядок виконання

1. Запуск `pgserver` у `/tmp/hw5_pg`, вибір режиму даних, створення typed-схеми (або stub) і smoke-test кількості рядків.
2. CTE-pipeline: `reference_date` → orders total → RFM → cohort → rolling 30-day AOV → cancellation rate → category revenue → preferred category. Кожен stage перевіряється окремо.
3. Створення `hw5_referrals` і recursive CTE з `path_ids` та depth guard. Окремо на таблиці `hw5_referrals_cycle_demo` демонструється цикл: з guard-ами, лише з depth guard і через `CYCLE`-clause.
4. `mv_olist_customer_features` з того самого SQL, що й pipeline; `UNIQUE INDEX` на `customer_id`; `REFRESH ... CONCURRENTLY`.
5. `EXPLAIN (ANALYZE, BUFFERS)` для pipeline, lookup і top-N з MV, зведена таблиця часу виконання, reflection.

## Примітки

- SQLAlchemy підключається через `postgresql+psycopg2://`. DDL виконується напряму через cursor psycopg2 без параметрів, щоб `%` (modulo) у stub-seed не сприймався як placeholder.
- Pipeline, як і в шаблоні завдання, рахується за `customer_id`. В Olist кожен `customer_id` має рівно одне замовлення, тому на реальних даних `frequency` завжди дорівнює 1. Це обмеження розібрано в notebook. Для справжнього RFM групувати треба за `customer_unique_id`.
- Індекс для top-N створено як `(monetary DESC NULLS LAST)`. Шаблонний `(monetary DESC)` має порядок `NULLS FIRST` і не підходить для `ORDER BY monetary DESC NULLS LAST LIMIT 10`. У notebook показано обидва плани.
