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
        database_name text;
        schema_name text;
        table_name text;
        row record;
        attrs text;
        descr record;
        con record;
        splitter text = '+-------+---------------------+-------------------';
        query text;
        query_result record;
    begin
        database_name := split_part(full_table_name, '.', 1);
        schema_name := split_part(full_table_name, '.', 2);
        table_name := split_part(full_table_name, '.', 3);
    
        if table_name='' or table_name is null then
            if schema_name = '' or schema_name is null then
                raise notice 'Схема не указана';
                table_name := database_name;
                schema_name := null;
            else
                table_name := schema_name;
                schema_name := database_name;
            end if;
            database_name := current_database();
        end if;

        if database_name != current_database() then
            raise exception 'Нельзя получать информацию о таблице из другой БД.';
        end if;
    
        query := 'select pg_class.oid oid, pg_namespace.nspname schema_name
                  from pg_class
                  join pg_namespace
                      on pg_class.relnamespace = pg_namespace.oid
                  where pg_class.relname= $1';
    
        if schema_name is not null then
            query := query || ' and pg_namespace.nspname= $2;';
            execute query into query_result using table_name, schema_name;
        else
            query := query || ';';
            execute query into query_result using table_name;
        end if;

        table_oid := query_result.oid;
        schema_name := query_result.schema_name;

        if table_oid is null then
            raise exception 'Таблица "%" не найдена', full_table_name;
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
call get_table_info('..');
call get_table_info('studs.s367522.employees');
call get_table_info('ucheb.s367522.employees');
call get_table_info('s367522.employees');
call get_table_info('employees');
```

### Документация

- [pg_attribute](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-attribute)
- [pg_class](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-class)
- [pg_constraint](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-constraint)
- [pg_description](https://postgrespro.ru/docs/postgrespro/10/catalog-pg-description)