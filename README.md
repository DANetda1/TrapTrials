# Решение задач по хранимым процедурам и функциям

## Задания

### Задание 1. Создание хранимой процедуры.

**Цель:** процедура добавляет новую должность в таблицу `JOBS`. Максимальная зарплата для новой должности устанавливается как удвоенная минимальная зарплата.

a. Создайте хранимую процедуру с именем `NEW_JOB`, которая принимает три параметра:
- Идентификатор должности (`job_id`),
- Название должности (`job_title`),
- Минимальную зарплату (`min_salary`).

Ответ:
```sql
create or replace procedure new_job(
    p_job_id     varchar(10),
    p_job_title  varchar(35),
    p_min_salary integer
)
language plpgsql
as $$
begin
    insert into jobs (job_id, job_title, min_salary, max_salary)
    values (
        p_job_id,
        p_job_title,
        p_min_salary,
        p_min_salary * 2
    );
end;
$$;
```

b. Выполните процедуру, добавив должность со следующими параметрами:
- `job_id`: `'SY_ANAL'`
- `job_title`: `'System Analyst'`
- `min_salary`: `6000`

Ответ:
```sql
call new_job('SY_ANAL', 'System Analyst', 6000);
```

---

### Задание 2. Создание хранимой процедуры.

**Цель:** Добавить запись в историю изменения должности сотрудника и обновить данные сотрудника.

a. Создайте процедуру с именем `ADD_JOB_HIST`, которая принимает два параметра:
- Идентификатор сотрудника (`employee_id`),
- Новый идентификатор должности (`job_id`).

Процедура должна:
- Добавить новую запись в `JOB_HISTORY` с текущей датой найма сотрудника в качестве даты начала и сегодняшней датой в качестве даты окончания.
- Обновить дату найма сотрудника в таблице `EMPLOYEES` на сегодняшнюю дату.
- Изменить должность сотрудника на новую и установить его зарплату как минимальная зарплата этой должности плюс 500.
- Добавить обработку исключений на случай, если сотрудник не существует.

Ответ:
```sql
create or replace procedure add_job_hist(
    p_employee_id integer,
    p_job_id      varchar(10)
)
language plpgsql
as $$
declare
    v_hire_date     date;
    v_old_job_id    varchar(10);
    v_department_id integer;
    v_min_salary    integer;
begin
    select hire_date, job_id, department_id
    into v_hire_date, v_old_job_id, v_department_id
    from employees
    where employee_id = p_employee_id;

    if not found then
        raise exception 'Сотрудник с id % не существует', p_employee_id;
    end if;

    insert into job_history(employee_id, start_date, end_date, job_id, department_id)
    values (p_employee_id, v_hire_date, current_date, v_old_job_id, v_department_id);

    select min_salary
    into v_min_salary
    from jobs
    where job_id = p_job_id;

    update employees
    set job_id    = p_job_id,
        hire_date = current_date,
        salary    = v_min_salary + 500
    where employee_id = p_employee_id;
end;
$$;
```

b. Отключите триггеры на таблицах `EMPLOYEES`, `JOBS`, `JOB_HISTORY`.

Ответ:
```sql
alter table employees   disable trigger all;
alter table jobs        disable trigger all;
alter table job_history disable trigger all;
```

c. Выполните процедуру с параметрами:
- `employee_id`: `106`
- `job_id`: `'SY_ANAL'`

Ответ:
```sql
call add_job_hist(106, 'SY_ANAL');
```

Вставьте скриншот результата выполнения.
![Результат выполнения процедуры](./images/sem08_01.png)

d. Выполните запросы для проверки изменений в таблицах `JOB_HISTORY` и `EMPLOYEES`.

Ответ:
```sql
select *
from job_history
where employee_id = 106;

select *
from employees
where employee_id = 106;
```

Вставьте скриншоты результатов.
![Результат выполнения запроса](./images/sem08_02.png)

e. Зафиксируйте изменения (commit).

f. Включите триггеры обратно.

Ответ:
```sql
alter table employees   enable trigger all;
alter table jobs        enable trigger all;
alter table job_history enable trigger all;
```

---

### Задание 3. Создание хранимой процедуры.

**Цель:** Обновить диапазон зарплат для указанной должности с обработкой исключений.

a. Создайте процедуру `UPD_JOBSAL`, принимающую три параметра:
- Идентификатор должности (`job_id`),
- Новую минимальную зарплату (`min_salary`),
- Новую максимальную зарплату (`max_salary`).

Добавьте обработку исключений:
- Если указан несуществующий идентификатор должности;
- Если максимальная зарплата меньше минимальной;
- Если строка заблокирована (используйте FOR UPDATE NOWAIT).

Ответ:
```sql
create or replace procedure upd_jobsal(
    p_job_id      varchar(10),
    p_min_salary  integer,
    p_max_salary  integer
)
language plpgsql
as $$
declare
    v_dummy integer;
begin
    if p_max_salary < p_min_salary then
        raise exception 'Максимальная зарплата % меньше, чем минимальная %', p_max_salary, p_min_salary;
    end if;

    begin
        select 1
        into v_dummy
        from jobs
        where job_id = p_job_id
        for update nowait;
    exception
        when no_data_found then
            raise exception 'Работа с id % не существует', p_job_id;
        when lock_not_available then
            raise exception 'Строка для job id % заблокирована', p_job_id;
    end;

    update jobs
    set min_salary = p_min_salary,
        max_salary = p_max_salary
    where job_id = p_job_id;
end;
$$;
```

b. Выполните процедуру с параметрами: `job_id`='SY_ANAL', `min_salary`=7000, `max_salary`=140 (ожидается ошибка).

Вставьте скриншот ошибки.
![Результат выполнения процедуры](./images/sem08_03.png)

c. Отключите триггеры на таблицах `EMPLOYEES`, `JOBS`.

Ответ:
```sql
alter table employees disable trigger all;
alter table jobs      disable trigger all;
```

d. Повторно выполните процедуру с корректными параметрами: `min_salary`=7000, `max_salary`=14000.

Ответ:
```sql
call upd_jobsal('SY_ANAL', 7000, 14000);
```

Вставьте скриншот результата выполнения.
![Результат выполнения процедуры](./images/sem08_04.png)

e. Проверьте изменения в таблице `JOBS`.

Вставьте скриншот изменений.
![Результат выполнения запроса](./images/sem08_05.png)

f. Зафиксируйте изменения и включите триггеры обратно.

Ответ:
```sql
commit;

alter table employees enable trigger all;
alter table jobs      enable trigger all;
```

---

### Задание 4. Создание хранимой функции.

**Цель:** Рассчитать стаж сотрудника.

a. Создайте функцию `GET_YEARS_SERVICE`, принимающую идентификатор сотрудника и возвращающую его стаж работы (в годах). Добавьте обработку исключений на случай несуществующего сотрудника.

Ответ:
```sql
create or replace function get_years_service(
    p_employee_id integer
)
returns integer
language plpgsql
as $$
declare
    v_hire_date date;
    v_years     integer;
begin
    select hire_date
    into v_hire_date
    from employees
    where employee_id = p_employee_id;

    if not found then
        raise exception 'сотрудник с id % не существует', p_employee_id;
    end if;

    v_years := extract(year from age(current_date, v_hire_date));

    return v_years;
end;
$$;
```

b. Вызовите функцию для сотрудника с ID 999, используя RAISE NOTICE (ожидается ошибка).

Ответ:
```sql
do $$
begin
    raise notice 'Years of service for 999: %', get_years_service(999);
end;
$$;
```

Вставьте скриншот результата выполнения.
![Результат выполнения функции](./images/sem08_06.png)

c. Вызовите функцию для сотрудника с ID 105.

Ответ:
```sql
select get_years_service(105);
```

Вставьте скриншот результата выполнения.
![Результат выполнения функции](./images/sem08_07.png)

d. Проверьте корректность данных запросом из таблиц `JOB_HISTORY` и `EMPLOYEES`.

Вставьте скриншот результата.
![Результат выполнения запроса](./images/sem08_08.png)

---

### Задание 5. Создание хранимой функции.

**Цель:** Получить количество уникальных должностей сотрудника.

a. Создайте функцию `GET_JOB_COUNT`, возвращающую количество уникальных должностей, на которых работал сотрудник (включая текущую). Используйте UNION и DISTINCT. Добавьте обработку исключений для несуществующего сотрудника.

Ответ:
```sql
create or replace function get_job_count(
    p_employee_id integer
)
returns integer
language plpgsql
as $$
declare
    v_cnt integer;
begin
    perform 1
    from employees
    where employee_id = p_employee_id;

    if not found then
        raise exception 'Сотрудник с id % не существует', p_employee_id;
    end if;

    select count(distinct job_id)
    into v_cnt
    from (
        select job_id
        from employees
        where employee_id = p_employee_id

        union

        select job_id
        from job_history
        where employee_id = p_employee_id
    ) j;

    return v_cnt;
end;
$$;
```

b. Вызовите функцию для сотрудника с ID 176.

Ответ:
```sql
select get_job_count(176);
```

Вставьте скриншот результата выполнения.
![Результат выполнения функции](./images/sem08_09.png)

---

### Задание 6. Создание триггера.

**Цель:** Проверить, что изменение зарплат должности не выводит текущие зарплаты сотрудников за новые пределы.

a. Создайте триггер `CHECK_SAL_RANGE`, срабатывающий перед обновлением столбцов `MIN_SALARY` и `MAX_SALARY` таблицы `JOBS`. Триггер должен проверять текущие зарплаты сотрудников и выдавать исключение, если новая зарплата выходит за пределы заданного диапазона.

Ответ:
```sql
create or replace function check_sal_range_fn()
returns trigger
language plpgsql
as $$
declare
    v_dummy integer;
begin
    select 1
    into v_dummy
    from employees e
    where e.job_id = new.job_id
      and (
            (new.min_salary is not null and e.salary < new.min_salary)
         or (new.max_salary is not null and e.salary > new.max_salary)
      )
    limit 1;

    if found then
        raise exception
            'Новый диапазон зарплат [% - %] не включает текущие зарплаты сотрудников для должности %',
            new.min_salary, new.max_salary, new.job_id;
    end if;

    return new;
end;
$$;

create trigger check_sal_range
before update of min_salary, max_salary
on jobs
for each row
execute function check_sal_range_fn();
```

b. Протестируйте триггер с диапазоном от 10000 до 20000 для должности AC_ACCOUNT (ожидается ошибка).

Ответ:
```sql
update jobs
set min_salary = 10000,
    max_salary = 20000
where job_id = 'AC_ACCOUNT';
```

Вставьте скриншот ошибки.
![Результат выполнения триггера](./images/sem08_10.png)

c. Затем установите диапазон от 8000 до 15000 и объясните результат.

Вставьте скриншот результата и напишите объяснение.
![Результат выполнения триггера](./images/sem08_11.png)

Объяснение:
```markdown
-- Ваш ответ здесь
```
Если посмотреть таблицу employees, то можно увидеть, что у должности AC_ACCOUNT есть только один сотрудник с зарплатой 8300. Эта зарплата находится внутри нового диапазона от 8000 до 15000, поэтому триггер не выдал ошибку и обновление прошло успешно.
---

