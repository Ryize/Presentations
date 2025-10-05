# Roadmap: RLS в финансовом домене (PostgreSQL)

---

## Что делаем
- **Задача:** изоляция данных по арендаторам (tenant) и доступам к счетам с RLS.
- **База:** PostgreSQL 14+ (лучше 15+).
- **Роли:** `app` — приложение (живёт под RLS), `app_admin` — админ (создаёт и рулит).
- **Контекст сессии:** `app.tenant_id`, `app.user_id`, `app.role` — задаёт приложение через `set_config`.

Схема простая: пользователи → имеют доступ к счетам → по счетам есть транзакции. RLS фильтрует строки по `tenant_id` и по доступам к конкретным счетам.

---

## Структура репозитория

```
.
├─ docker/
│  └─ docker-compose.yml
├─ sql/
│  ├─ 01_roles_and_schema.sql
│  ├─ 02_tables.sql
│  ├─ 03_rls_enable.sql
│  ├─ 04_policies.sql
│  ├─ 05_grants.sql
│  ├─ 06_seed.sql
│  └─ 07_scenarios.sql
├─ tests/             # опционально: pgTAP
│  └─ rls_tests.sql
├─ migrations/        # опционально: Flyway/Liquibase
│  └─ V1__init.sql
└─ README.md          # этот файл
```

---

## Быстрый старт (Docker)

`docker/docker-compose.yml`:

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: fin
    ports:
      - "5432:5432"
    volumes:
      - ./init:/docker-entrypoint-initdb.d:ro
```

Самый простой путь: положи все SQL из раздела ниже в папку `init/` (с префиксами `01_*.sql`, `02_*.sql`, ...). При первом старте Postgres сам всё применит.

Старт:
```bash
docker compose -f docker/docker-compose.yml up -d
PGPASSWORD=admin psql -h localhost -U admin -d fin -c "select version();"
```

---

## Пошаговая установка

> Все команды ниже можно просто выполнить подряд в `psql` под владельцем БД/суперпользователем.

### 1) Роли и схема

```sql
-- sql/01_roles_and_schema.sql
CREATE ROLE app LOGIN PASSWORD 'app_password';
CREATE ROLE app_admin LOGIN PASSWORD 'admin_password';

CREATE SCHEMA fin AUTHORIZATION app_admin;
GRANT USAGE ON SCHEMA fin TO app;
SET search_path TO fin, public;
```

### 2) Таблицы и индексы

```sql
-- sql/02_tables.sql
SET search_path TO fin, public;

CREATE TABLE fin.tenants (
  id        int PRIMARY KEY,
  name      text NOT NULL
);

CREATE TABLE fin.app_users (
  id        int PRIMARY KEY,
  login     text UNIQUE NOT NULL,
  full_name text NOT NULL,
  tenant_id int NOT NULL REFERENCES fin.tenants(id)
);

CREATE TABLE fin.accounts (
  id         bigserial PRIMARY KEY,
  tenant_id  int NOT NULL REFERENCES fin.tenants(id),
  account_no text UNIQUE NOT NULL,
  currency   text NOT NULL CHECK (char_length(currency)=3)
);

CREATE TABLE fin.user_account_access (
  user_id    int    NOT NULL REFERENCES fin.app_users(id),
  account_id bigint NOT NULL REFERENCES fin.accounts(id),
  can_trade  boolean NOT NULL DEFAULT false,
  PRIMARY KEY (user_id, account_id)
);

CREATE TABLE fin.transactions (
  id         bigserial PRIMARY KEY,
  tenant_id  int NOT NULL REFERENCES fin.tenants(id),
  account_id bigint NOT NULL REFERENCES fin.accounts(id),
  tx_time    timestamptz NOT NULL DEFAULT now(),
  kind       text NOT NULL CHECK (kind IN ('DEPOSIT','WITHDRAWAL','FEE','DIVIDEND','TRADE')),
  amount     numeric(18,2) NOT NULL,
  created_by int NOT NULL REFERENCES fin.app_users(id)
);

-- Индексы под RLS-предикаты и джоины
CREATE INDEX ON fin.accounts(tenant_id);
CREATE INDEX ON fin.transactions(tenant_id);
CREATE INDEX ON fin.transactions(account_id);
CREATE INDEX ON fin.user_account_access(user_id, account_id);
```

### 3) Включаем RLS

```sql
-- sql/03_rls_enable.sql
ALTER TABLE fin.accounts     ENABLE ROW LEVEL SECURITY;
ALTER TABLE fin.transactions ENABLE ROW LEVEL SECURITY;

-- Принудительно применяем RLS всегда
ALTER TABLE fin.accounts     FORCE ROW LEVEL SECURITY;
ALTER TABLE fin.transactions FORCE ROW LEVEL SECURITY;
```

### 4) Полезные функции контекста + политики

```sql
-- sql/04_policies.sql
CREATE OR REPLACE FUNCTION fin.ctx_tenant() RETURNS int LANGUAGE sql IMMUTABLE AS
$$ SELECT COALESCE(current_setting('app.tenant_id', true), '-1')::int; $$;

CREATE OR REPLACE FUNCTION fin.ctx_user() RETURNS int LANGUAGE sql IMMUTABLE AS
$$ SELECT COALESCE(current_setting('app.user_id',  true), '-1')::int; $$;

-- 4.1 Изоляция по tenant
CREATE POLICY rls_accounts_by_tenant ON fin.accounts
USING (tenant_id = fin.ctx_tenant());

CREATE POLICY rls_tx_by_tenant ON fin.transactions
USING (tenant_id = fin.ctx_tenant());

-- 4.2 Видимость счетов только при наличии membership
CREATE POLICY rls_accounts_by_membership ON fin.accounts
USING (
  EXISTS (
    SELECT 1
    FROM fin.user_account_access uaa
    WHERE uaa.account_id = accounts.id
      AND uaa.user_id = fin.ctx_user()
  )
);

-- 4.3 Транзакции видны по доступным счетам
CREATE POLICY rls_tx_by_account_access ON fin.transactions
USING (
  EXISTS (
    SELECT 1
    FROM fin.user_account_access uaa
    WHERE uaa.account_id = transactions.account_id
      AND uaa.user_id = fin.ctx_user()
  )
);

-- 4.4 Правила записи для трейдеров (INSERT/UPDATE)
CREATE POLICY rls_tx_insert_trader ON fin.transactions
AS PERMISSIVE
FOR INSERT
WITH CHECK (
  tenant_id = fin.ctx_tenant()
  AND created_by = fin.ctx_user()
  AND EXISTS (
    SELECT 1 FROM fin.user_account_access uaa
    WHERE uaa.account_id = transactions.account_id
      AND uaa.user_id = fin.ctx_user()
      AND uaa.can_trade = true
  )
);

CREATE POLICY rls_tx_update_trader ON fin.transactions
AS PERMISSIVE
FOR UPDATE
USING (
  created_by = fin.ctx_user()
  AND EXISTS (
    SELECT 1 FROM fin.user_account_access uaa
    WHERE uaa.account_id = transactions.account_id
      AND uaa.user_id = fin.ctx_user()
      AND uaa.can_trade = true
  )
)
WITH CHECK (
  tenant_id = fin.ctx_tenant()
  AND created_by = fin.ctx_user()
);
```

### 5) Права (минимум, без лишнего)

```sql
-- sql/05_grants.sql
GRANT SELECT ON fin.accounts, fin.transactions TO app;
GRANT INSERT, UPDATE ON fin.transactions TO app;
GRANT SELECT ON fin.user_account_access, fin.tenants, fin.app_users TO app;

REVOKE INSERT, UPDATE, DELETE ON fin.accounts FROM app;  -- записи в счета — только админ/ETL
```

### 6) Демоданные (под админом)

```sql
-- sql/06_seed.sql
SET ROLE app_admin;
SET search_path TO fin, public;
SET row_security = off;

INSERT INTO fin.tenants(id, name) VALUES (1001,'Acme Securities'), (1002,'Globex Capital');

INSERT INTO fin.app_users(id, login, full_name, tenant_id) VALUES
  (501,'trader.alex','Алексей Трейдер',1001),
  (502,'analyst.irina','Ирина Аналитик',1001),
  (503,'auditor.oleg','Олег Аудитор',1001),
  (601,'trader.bob','Боб Трейдер',1002);

INSERT INTO fin.accounts(tenant_id, account_no, currency) VALUES
  (1001,'ACC-ACME-001','USD'),
  (1001,'ACC-ACME-002','EUR'),
  (1002,'ACC-GLOB-001','USD');

INSERT INTO fin.user_account_access(user_id, account_id, can_trade)
SELECT 501, id, true FROM fin.accounts WHERE tenant_id=1001;

INSERT INTO fin.user_account_access(user_id, account_id, can_trade)
SELECT 502, id, false FROM fin.accounts WHERE tenant_id=1001;

INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1001, a.id, 'DEPOSIT', 10000.00, 501 FROM fin.accounts a WHERE a.account_no='ACC-ACME-001';

INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1001, a.id, 'FEE', -25.00, 501 FROM fin.accounts a WHERE a.account_no='ACC-ACME-001';

INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1002, a.id, 'DEPOSIT', 5000.00, 601 FROM fin.accounts a WHERE a.account_no='ACC-GLOB-001';

RESET row_security;
RESET ROLE;
```

### 7) Как проверить (живая демонстрация)

```sql
-- sql/07_scenarios.sql

-- Алексей (trader, tenant=1001)
SET ROLE app;
SELECT set_config('app.tenant_id','1001',false);
SELECT set_config('app.user_id','501',false);
SELECT set_config('app.role','trader',false);

SELECT 'Alex: Accounts' AS section, * FROM fin.accounts;
SELECT 'Alex: Transactions' AS section, * FROM fin.transactions;

-- Вставка по доступному счёту — ок
INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1001, a.id, 'TRADE', -1234.56, 501
FROM fin.accounts a WHERE a.account_no='ACC-ACME-002';

-- Попытка записать в чужой tenant — должно упасть по RLS
INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1002, a.id, 'FEE', -10.00, 501
FROM fin.accounts a WHERE a.account_no='ACC-GLOB-001';

-- Ирина (analyst, read-only по membership)
SELECT set_config('app.tenant_id','1001',false);
SELECT set_config('app.user_id','502',false);
SELECT set_config('app.role','analyst',false);

SELECT 'Irina: Accounts' AS section, * FROM fin.accounts;
SELECT 'Irina: Transactions' AS section, * FROM fin.transactions;

-- Попытка вставки — нет прав (ни политики, ни GRANT)
INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
SELECT 1001, a.id, 'FEE', -1.00, 502 FROM fin.accounts a LIMIT 1;

-- Олег (auditor, tenant-wide read-only)
SELECT set_config('app.tenant_id','1001',false);
SELECT set_config('app.user_id','503',false);
SELECT set_config('app.role','auditor',false);

SELECT 'Oleg: Accounts' AS section, * FROM fin.accounts;
SELECT 'Oleg: Transactions' AS section, * FROM fin.transactions;

-- Попытка UPDATE — нет прав
UPDATE fin.transactions SET amount=amount+1 WHERE id=1;
```

---

## Перформанс и безопасность, коротко
- Индексы уже есть на полях из предикатов (`tenant_id`, `account_id`, `user_id`).
- Проверяй планы: `EXPLAIN (ANALYZE, BUFFERS)` на ключевых отчётах.
- Принцип *default deny*: не раздавай лишние GRANT и не пиши лишних write‑политик.
- Массовые загрузки/ETL — под `app_admin` с `SET row_security = off;` или ролью с `BYPASSRLS`.

---

## Тесты (pgTAP), если хочется
```sql
-- tests/rls_tests.sql (пример)
CREATE EXTENSION IF NOT EXISTS pgtap;
SELECT plan(3);

SELECT set_config('app.tenant_id','1001',false);
SELECT set_config('app.user_id','501',false);
SELECT set_config('app.role','trader',false);

SELECT ok( (SELECT COUNT(*) FROM fin.accounts) > 0, 'trader sees some accounts');
SELECT ok( (SELECT COUNT(*) FROM fin.transactions) > 0, 'trader sees some transactions');

-- аналитик не должен вставлять
BEGIN;
DO $$
BEGIN
  BEGIN
    INSERT INTO fin.transactions(tenant_id, account_id, kind, amount, created_by)
    SELECT 1001, a.id, 'FEE', -1.00, 502 FROM fin.accounts a LIMIT 1;
    RAISE EXCEPTION 'should fail';
  EXCEPTION WHEN others THEN
    -- норм, поймали ошибку
    NULL;
  END;
END;
$$;
ROLLBACK;
SELECT pass('analyst cannot insert');

SELECT finish();
```
Запуск:
```bash
PGPASSWORD=admin psql -h localhost -U admin -d fin -f tests/rls_tests.sql
```

---

## Миграции (по желанию)
Сложи всё в `migrations/V1__init.sql` и прогоняй Flyway/Liquibase. Главное — порядок: роли/схема → таблицы → включаем RLS → политики → гранты → сиды.

---

## Если что-то пошло не так
- Временный обход для админ-операций:
  ```sql
  SET ROLE app_admin;
  SET row_security = off;
  -- делаешь свои изменения
  RESET row_security;
  RESET ROLE;
  ```
- Можно временно отключить RLS на таблице (лучше не в проде):
  ```sql
  ALTER TABLE fin.transactions DISABLE ROW LEVEL SECURITY;
  ALTER TABLE fin.transactions ENABLE ROW LEVEL SECURITY;
  ```

---

### Готово
Копируй этот файл в `README.md`, SQL — в папку `sql/`, добавь `docker-compose.yml`, и всё повторится из коробки. Если хочешь, позже добьём раздел под SQL Server или интеграцию со своим бэкендом (Supabase/Hasura/Spring/SQLAlchemy).
