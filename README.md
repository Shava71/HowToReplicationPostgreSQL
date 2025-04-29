# Инструкция по репликации в Postgre

### Перед прочтением инструкции советую ознамоиться со [статьёй](https://habr.com/ru/articles/514500/) с целью понимания, что такое репликация

<details><summary>Логическая репликация [WAL]</summary>
Шаги установки:</br>
    
1. Установка логического декодирования. Переходим по пути (для собственного удобства опишу полный путь для MacOS): ```/Library/PostgreSQL/17/data```. Открываем файл ```postgresql.conf``` и вписываем:
    ```
    wal_level = logical
    max_replication_slots = 5
    max_wal_senders = 10
    ```
  * Установка wal_level в значение logical позволяет WAL записывать информацию, необходимую для логического декодирования.
  * Убедитесь, что значение параметра max_replication_slots равно или больше количества коннекторов PostgreSQL, использующих WAL, и прибавьте к этому количество других слотов репликации, используемых вашей базой данных.
  * Убедитесь, что параметр max_wal_senders, определяющий максимальное количество одновременных соединений с WAL, как минимум вдвое превышает количество логических слотов репликации. Например, если ваша база данных использует в общей сложности 5 слотов репликации, значение параметра max_wal_senders должно быть 10 или больше.

2. Перезапустим сервис postgresql
   Команды для перезапуска postgresql для маководов, если она была установлена через ```.dmg``` с официального сайта:  
   ```sudo launchctl list | grep postgres``` - просмотр сервисов postgres  
   ```sudo launchctl start/stop postgresql-17``` - старт/остановка сервиса
  
3. Настроить логическую репликацию с помощью подключаемого модуля ```test_decoding```:<br/>
   Создайте слот логической репликации для базы данных, которую вы хотите синхронизировать, выполнив следующую команду:<br/>
    
   ``` SELECT pg_create_logical_replication_slot('replication_slot', 'test_decoding'); ```<br/>
   
   Чтобы убедиться в том, что слот был успешно создан, выполните следующую команду:<br/>
   
   ```SELECT slot_name, plugin, slot_type, database, active, restart_lsn, confirmed_flush_lsn FROM pg_replication_slots;```

5. Cоздайте публикацию для всех ваших таблиц или только для тех, что вам нужны. Если вы зададите конкретные таблицы, вы сможете добавить или удалить их из публикации позже.
  Все таблицы:  
   ```CREATE PUBLICATION pub FOR ALL TABLES;```
   Для определённых таблиц:
   ```CREATE PUBLICATION pub FOR TABLE table1, table2, table3;```
   По желанию вы можете выбрать, какие операции включить в публикацию. Например, следующая публикация включает для первой таблицы (table1) только операции INSERT и UPDATE.
   ```CREATE PUBLICATION insert_update_only_pub FOR TABLE table1 WITH (publish = 'INSERT, UPDATE');```

6. Убедитесь, что выбранные вами таблицы есть в публикации.
   ```SELECT * FROM pg_publication_tables WHERE pubname='pub';```
   OUTPUT:
   ```
   pubname | schemaname | tablename
    ---------+------------+-----------
    pub     | public     | table1
    pub     | public     | table2
    pub     | public     | table3
    (3 rows)
    ```
   С этого момента наша публикация ```pub``` будет отслеживать изменения всех таблиц в базе данных ```psql-stream```.

7. Заполним одну из таблиц некоторыми данными. <br/>
   Проверим содержимое в WAL при помощи запроса: <br/>
   ```SELECT * FROM pg_logical_slot_get_changes('replication_slot', NULL, NULL);```
   OUTPUT:
   ```
      lsn    | xid  |                          data                          
    -----------+------+--------------------------------------------------------
     0/19EA2C0 | 1045 | BEGIN 1045
     0/19EA2C0 | 1045 | table public.t: INSERT: id[integer]:1 name[text]:51459cbc211647e7b31c8720
     0/19EA300 | 1045 | table public.t: INSERT: id[integer]:2 name[text]:51459cbc211647e7b31c8720
     0/19EA340 | 1045 | table public.t: INSERT: id[integer]:3 name[text]:51459cbc211647e7b31c8720
     0/19EA380 | 1045 | table public.t: INSERT: id[integer]:4 name[text]:51459cbc211647e7b31c8720
     0/19EA3C0 | 1045 | table public.t: INSERT: id[integer]:5 name[text]:51459cbc211647e7b31c8720
     0/19EA400 | 1045 | table public.t: INSERT: id[integer]:6 name[text]:51459cbc211647e7b31c8720
     0/19EA440 | 1045 | table public.t: INSERT: id[integer]:7 name[text]:51459cbc211647e7b31c8720
     0/19EA480 | 1045 | table public.t: INSERT: id[integer]:8 name[text]:51459cbc211647e7b31c8720
     0/19EA4C0 | 1045 | table public.t: INSERT: id[integer]:9 name[text]:51459cbc211647e7b31c8720
     0/19EA500 | 1045 | table public.t: INSERT: id[integer]:10 name[text]:51459cbc211647e7b31c8720
     0/19EA5B0 | 1045 | COMMIT 1045
   ```
   * ```pg_logical_slot_peek_changes``` — это ещё одна команда PostgreSQL для просмотра изменений из записей WAL без их поглощения. Поэтому многократный вызов команды ```pg_logical_slot_peek_changes``` будет возвращать один и тот же результат.
   * В свою очередь, ```pg_logical_slot_get_changes``` возвращает результаты только при первом вызове. Последующие вызовы ```pg_logical_slot_get_changes``` возвращают пустые наборы результатов. Это означает, что при выполнении команды ```get``` результаты обрабатываются и удаляются, что значительно расширяет наши возможности по написанию логики использования этих событий для создания реплики таблицы.
  
8. Не забудьте избавиться от слота, который вам больше не нужен, чтобы остановить его поглощение. <br/>
    ```SELECT pg_drop_replication_slot('replication_slot');```

9. Создайте пользователя ```replication```, чтобы через него дополнительный сервер мог подключаться к основному: <br/>
    ```CREATE ROLE replication WITH REPLICATION PASSWORD '<superpassrowd>' LOGIN;```
   * Необходимо так же выделить права данной роли на чтение таблиц, на которые была зарегестрирована подписка
    ```GRANT SELECT ON [(ALL TABLES)|table_name] IN SCHEMA public TO replication;```

10. В файле ```pg_hba.conf``` разрешаем подключение этому пользователю (путь к нему можно найти при помощи запроса ```SHOW hba_file;```: <br/>
    ```
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    host    replication     replication     192.168.233.0/24         md5
    ```

11. Перезапускаем postgresql
12. В БД, на основе котороый будет происходить репликация, создаём её:
    ```
    CREATE PUBLICATION name_pub
    FOR TABLE public.table_name
    WITH (publish = 'insert, update, delete, truncate'); -- операции, которые будут разрешены
    ```
13. Переходим в БД, которая будет подключаться к публикации
14. Создаём подписку: <br/>
    ```
    CREATE SUBSCRIPTION name_sub
    CONNECTION 'host=localhost port=**** dbname=DB user=replication password=...'
    PUBLICATION name_pub
    WITH (copy_data = true);
    ```
</details>

<details><summary>Физическая репликация на базе потоковой [WAL]</summary>
    
1. На мастер-сервере изменим параметры ```postgresql.conf``` изменим следующим образом:

    ```
    # Определяет как много информации записывать в WAL. 
    # Со значением replica в журнал записываются данные для поддержки архивирования WAL и репликации.
    # В т.ч. для запросов только на чтение.
    # wal_level = hot_standby - для версий до 9.6
    # https://www.postgresql.org/docs/9.6/runtime-config-wal.html
    wal_level = replica
    
    # Число одновременных подключений для резервных серверов. Жалательно установить на 1 подключение
    # больше, чем фактическое количество резервных серверов, т.к. в случае неожиданного отключения
    # старое соединение будет некоторое время использоваться.
    # https://www.postgresql.org/docs/9.4/runtime-config-replication.html
    max_wal_senders = 10
    
    # Задает минимальный размер в мегабайтах сегментов файлов журнала, хранящихся в каталоге pg_wal, на случай,
    # если резервному серверу потребуется извлечь их для потоковой репликации.
    # В ранних версиях параметр назывался wal_keep_segments и указывал количество файлов, а не их размер.
    # https://www.postgresql.org/docs/13/runtime-config-replication.html
    wal_keep_size = 1024
    ```

2. Создаём пользователя ```replication``` и прописываем его в ```pg_hba.conf``` как в пунктах 9-10 из логической репликации
3. Перезапускаем сервис
4. Переходим к серверу, на котором будет храниться копия исходной БД. Очищаем в его кластере все данные из ```data```:</br>
    ```rm -Rf /var/lib/pgsql/data/*```
5. Копируем текущее состояние с основного сервера на дополнительный:

    ```
    pg_basebackup -h localhost -p 5433 -U replication -D /Users/mac/postgres_data/cluster3 -Fp -Xs -P -R
    # -Fp: обычный формат
    # -Xs: вместе с WAL
    # -R: сразу создаёт standby.signal и postgresql.auto.conf с подключением
    ```
6. Запускаем кластер, заменив в нём порт на другое значение.
</details>


   


