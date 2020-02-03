# Настройка master
Для начала поправим postgresql.conf. В Debian он находится по адресу /etc/postgresql/VERSION/CLUSTER/postgresql.conf. 
В нашем примере это /etc/postgresql/9.6/master/postgresql.conf
#master должен быть доступен по сети для slave
#listen_addresses может принимать несколько значений (через запятую)
#можно поставить * - postgres будет доступен на всех сетевых интерфейсах
listen_addresses = '10.0.0.1'
#режим хранения WAL-сегментов. Для репликации – только hot_standby
wal_level = hot_standby
#максимальное количство wal_sender.
#это максимум slave-ов, который сможет подключится к этому серверу
max_wal_senders = 5
#сколько заполненных WAL-сегментов хранить на мастере перед удалением
#число можно подобрать только экспериментально (больше изменений – больше WAL надо хранить)
wal_keep_segments = 32
#папка для архива. Удаленный WAL-сегмент будет скопирован туда
#архивом можно пользоваться для восстановления slave, если slave не успел выкачать WAL с мастера, а мастер его уже удалил
#там должно быть много места – сам postgres не будет чистить свой архив
archive_mode    = on
archive_command = 'cp %p /var/lib/pg-archive/%f'



Теперь нужно разрешить slave-у подключаться к мастеру для репликации. 
Для этого отредактируем pg_hba.conf (лежит там же, где postgresql.conf), и добавим туда специального пользователя:

#TYPE   DB             USER           ADDRESS        #METHOD
host    replication    replication    10.0.0.2/32    md5


Теперь надо создать папку для архива:

mkdir /var/lib/pg-archive/
chown postgres /var/lib/pg-archive/
chmod 700 /var/lib/pg-archive/

Создадим пользователя для репликации. Для этого в консоли самого постгреса выполним команду:

CREATE ROLE replication WITH REPLICATION PASSWORD 'Hw572BbvG7g4cwq5' LOGIN;




# Копируем данные
Для начала остановим postgres на slave и удалим данные из datadir на salve:

slave> service postgresql stop
slave> rm -rf /var/lib/postgresql/9.6/main/*
Теперь скопируем основной каталог данных с мастера (текущее состояние)

master> psql -c "SELECT pg_start_backup('sync', true)"
master> rsync -rahzP /var/lib/postgresql/9.6/main/ 10.0.0.2:/var/lib/postgresql/9.6/main/ --exclude=postmaster.pid
master> psql -c "SELECT pg_stop_backup()"


Настраиваем salve
Если вы хотите читать данные из slave - нужно включить режим hot_standby. Это полезно, если slave используется для распределения нагрузки на чтение. Если slave нужен исключительно как горячая замена мастеру на случай аварии – этот параметр можно не трогать. В конфиге /etc/postgresql/9.6/master/postgresql.conf добавим:

hot_standby = on
В папке с данными (в нашем примере это /var/lib/postgresql/9.6/main/) создадим файл с настройками репликации. Он должен называться recovery.conf

standby_mode = 'on'
primary_conninfo = 'host=10.0.0.1 port=5432 user=replication password=Hw572BbvG7g4cwq5'
trigger_file = '/var/lib/postgresql/9.6/promote_to_master'
restore_command = 'cp /var/lib/pg-archive/%f "%p"'
Создадим на slave папку для архива WAL (так же, как это было сделано на master)

mkdir /var/lib/pg-archive/
chown postgres /var/lib/pg-archive/
chmod 700 /var/lib/pg-archive/
Теперь синхронизируем архив мастера с архивом slave, чтобы гарантировать успешный запуск:

master> rsync -rahzP /var/lib/pg-archive/ 10.0.0.2:/var/lib/pg-archive/
Поднимаем slave:

slave> service postgresql start
В журнале постгреса можно увидеть что сервис стартовал и готов обслуживать соединения:

2018-01-11 02:11:31 MSK [26781-1] LOG:  database system is ready to accept read only connections
2018-01-11 02:11:31 MSK [26788-1] LOG:  started streaming WAL from primary at 10B/33000000 on timeline 1
Чтобы WAL-сегменты не сожрали весь диск мастера подчистую – их надо периодически чистить. Несложный скрипт в crontab поможет:

10 6 * * * /usr/bin/find /var/lib/pg-archive/ -type f -mtime +7 -exec rm {} \;
В этом примере мы чистим архив от сегментов старше 7 дней, задача выполняется в 6:10 утра по времени сервера.

Диагностика и ремонт
Как проверить, что репликация работает? Проще всего - выяснить текущее положение WAL на мастере и slave:

master$ psql -c "SELECT pg_current_xlog_location()"
 pg_current_xlog_location
--------------------------
 0/2000000
(1 row)
slave$ psql -c "select pg_last_xlog_receive_location()"
 pg_last_xlog_receive_location
-------------------------------
 0/2000000
(1 row)
В норме положение мастера и slave должны быть близки или одинаковы (они будут одинаковы, если между выполнением команды на master и на slave на мастере не было изменений). Если на мастере число растет, а на slave – нет – репликация сломалась. Самый простой способ восстановить репликацию:

#остановим slave:
slave> service postgresql stop
#скопируем архив WAL-сегментов с мастера на salve
master> rsync -rahzP /var/lib/pg-archive/ 10.0.0.2:/var/lib/pg-archive/
#запустим slave обратно:
slave> service postgresql start
Это сработает, если синхронизация была потеряна недавно (в конкретно нашем примере – не более 7 дней назад) и WAL-ы из архива еще не удалены. Если синхронизацию сломали давно – придется синхронизироваться с нуля, как описано в разделах “копируем данные” и “настраиваем slave”. То есть – чистить datadir на slave, копировать состояние, копировать архивы и т.д.

Промотирование (перевод slave в master)
Это нужно в тех случаях, когда slave подменяет мастер на случай аварии. Для того, чтобы промотировать slave – нужно создать файл с именем, описанным в секции trigger_file конфига recovery.conf. В нашем примере это /var/lib/postgresql/9.6/promote_to_master

touch /var/lib/postgresql/9.6/promote_to_master

Содержание файла может быть любым.

После этого:

slave перестанет реплицироваться с master
slave станет доступен для операций записи
slave начнет собственный отсчет WAL. Это значит, что даже если master вернется – смигрировать с него данные на slav e автоматически уже не получится
Выводы
Репликация – мощная, удобная и надежная техника. Репликация в postgresql позволяет легко распределить нагрузку и повысить надежность инсталляции. Эта конструкция работает очень надежно и почти никогда не ломается (привет MySQL!). Разумеется, нужно помнить, что:

любая техника требует мониторинга. Проверяйте состояние реплик!
репликация не заменяет backup, а только дополняет его.