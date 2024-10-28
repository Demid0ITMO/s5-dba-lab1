## Вариант 367526

### Задание
Используя сведения из <b>системных каталогов</b> получить информацию о любой таблице: 

- Номер по порядку, 
- Имя столбца, 
- Атрибуты (<i>в атрибуты столбца включить тип данных, ограничение типа CHECK</i>).

Программу оформить в виде <b>процедуры</b>

### Решение

```postgresql
create or replace procedure get_table_info(full_table_name text)
    language plpgsql as $$
    declare
        table_oid oid;
        schema_name text;
        table_name text;
        row record;
        attrs text;
        descr record;
        con record;
        splitter text = '+-------+---------------------+-------------------';
    begin
        schema_name := split_part(full_table_name, '.', 1);
        table_name := split_part(full_table_name, '.', 2);
    
        if table_name is null or table_name = '' then
            raise notice 'Схема не указана. Поиск в текущей схеме';
            table_name := schema_name;
            schema_name := "current_schema"();
        end if;

        select pg_class.oid into table_oid 
        from pg_class
        join pg_namespace 
            on pg_class.relnamespace = pg_namespace.oid
        where pg_class.relname=table_name
            and pg_namespace.nspname=schema_name;

        if table_oid is null then
            raise exception 'Таблица "%" не найдена в схеме "%".', table_name, schema_name;
        end if;

        raise notice 'Схема: %, Таблица: %', schema_name, table_name;
        raise notice '%', splitter;
        raise notice '|No.    |    Имя Столбца      |     Атрибуты       ';
        raise notice '%', splitter;

        for row in (
        select att.attnum, att.attname, type.typname, att.attnotnull, att.atttypmod
        from pg_attribute att
        join pg_type type
            on att.atttypid = type.oid
        where att.attrelid=table_oid
            and att.attisdropped=false
            and att.attnum > 0
        ) loop
            
            attrs := 'Type: ' || row.typname;
            if not row.atttypmod = -1 then
                attrs := attrs || '(' || row.atttypmod || ')';
            end if;
            if row.attnotnull then 
                attrs := attrs || ' NOT NULL'; 
            end if;

            raise notice '| % %| % %| %',
                row.attnum, repeat(' ', positive_or_zero(5 - length(cast(row.attnum as text)))),
                row.attname, repeat(' ', positive_or_zero(19 - length(row.attname))),
                attrs;

            for descr in (
            select description
            from pg_description
            where objoid=table_oid
                and objsubid=row.attnum
            ) loop
                attrs := 'Comment: ' || descr.description;
                raise notice '|%|%| %', repeat(' ', 7), repeat(' ', 21), attrs;
            end loop;

            for con in (
            select pg_get_constraintdef(oid) def
            from pg_constraint
            where conrelid=table_oid
                and contype='c'
                and array_position(conkey, row.attnum) is not null
            ) loop
                attrs := 'Constr: ' || con.def;
                raise notice '|%|%| %', repeat(' ', 7), repeat(' ', 21), attrs;
            end loop;

            raise notice '%', splitter;
        end loop;
    exception
        when others then
        raise notice 'Ошибка при выполнении процедуры: %', sqlerrm;
    end;
$$;

create or replace function positive_or_zero(val int) 
    returns int
    language plpgsql as $$
    begin
        if (val > 0) then return val;
        else return 0;
        end if;
    end;
$$;
```

### Тесты
```postgresql
call get_table_info('s367522.object');
call get_table_info('ldaflsldfla.lasdfl');
call get_table_info('human');
call get_table_info('..');
call get_table_info('pg_catalog.pg_attribute');
call get_table_info('departments');
call get_table_info('s367522.employees');
call get_table_info('projects');
```

### Документация

- [pg_attribute](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-attribute)
- [pg_class](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-class)
- [pg_constraint](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-constraint)
- [pg_description](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-description)