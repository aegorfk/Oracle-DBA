Oracle-DBA
==========
Задача:  мигрировать БД между платформами HPUX и LINUX,  версии БД  11.2.0.2.0.
Для начала нужно узнать endian платформ между которыми мигрируем, так как от порядка байтов зависит какие преобразования с БД нам нужно будет сделать. Данную информацию можно взять из представлений:
•	v$transportable_platform- отображает все платформы, которые поддерживают кросс-платформенное преобразование табличных пространств. В частности, в нем перечислены все платформы, поддерживаемые командой RMAN CONVERT TABLESPACE, вместе с значениями их endian.
•	v$db_transportable_platform- отображает все платформы, в которые текущая база данных может преобразоваться с помощью команды RMAN CONVERT DATABASE. Преобразование БД с помощью этой команды поддерживается только для платформ с одинаковыми endian. Поэтому это представление отображает меньше строк, чем V$TRANSPORTABLE_PLATFORM.
                     Хост HPUX
oracle$export ORACLE_SID=OLD_DB
SQL> SELECT * FROM v$transportable_platform ORDER BY 2;
 
SQL> SELECT * FROM v$db_transportable_platform ORDER BY 2;
 
В нашем случае endian разные (PLATFORM_ID 4 и 13), поэтому дальше воспользуемся нотой по состветствующему переносу с металинка: 10g+: Transportable Tablespaces Across Different Platforms (Doc ID 243304.1). 
!Если endian платформ одинаковые, то потребуется нота: Cross-Platform Database Migration (across same endian) using RMAN Transportable Database (Doc ID 1401921.1).
                                             Выполним шаги для нашего случая:
1.	Определим объекты БД, которые требуется пересоздаться вручную после переноса БД. Это внешние таблицы, директории или BFILE. Посмотрим что из этого есть у нас:
 
SQL> col directory_name format a25
SQL> col directory_path format a50
SQL> SELECT directory_name, directory_path FROM DBA_DIRECTORIES;
! Ограничения: Нельзя транспортировать tbs SYSTEM и SYSAUX (и вообще объекты принадлежащие пользователю SYS). UNDO и TEMP тоже.
Соответственно на новом хосте нужно создать новую базу.
Кодировка и совместимость на базах, между которыми происходит транспортировка tbs дожны совпадать.  Проверить это можно так: select name, value from v$parameter where name in ('db_block_size','compatible');  и  select * from nls_database_parameters;
2. Подготовка
Посмотреть все платформы, для которых поддерживается транспортировка табличных пространств и их endian format:   select * from V$TRANSPORTABLE_PLATFORM order by endian_format;
Определить название и endian format своей платформы: SELECT d.PLATFORM_NAME, ENDIAN_FORMAT FROM V$TRANSPORTABLE_PLATFORM tp, V$DATABASE d WHERE tp.PLATFORM_NAME = d.PLATFORM_NAME;
Проверить, являются ли предназначенные к транспортировке tbs изолированными (self-contained):   EXEC DBMS_TTS.TRANSPORT_SET_CHECK('tbs1,tbs2....', TRUE);
Если являются, данный запрос не должен выдать ничего:
SELECT * FROM TRANSPORT_SET_VIOLATIONS;.
3. Транспортировка
-   Перевести предназначенные к транспортировке tbs в режим "только для чтения":alter tablespace tbs1 read only; и т.д.
-  Экспортировать tbs с помощью data pump:
create directory dump_dir as '/oradata/dump';
grant read,write on directory dump_dir to public;
expdp TRANSPORT_TABLESPACES=tbs1,tbs2,tbs3 DIRECTORY=dump_dir DUMPFILE=tr_tbs.dmp logfile=tr_tbs.log
-	Скопировать файлы данных и дампы транспортируемых tbs на хост с целевой базой (напр. с помощью scp) во временный каталог (напр. /oradata/tmp).
! Операцию RMAN CONVERT можно проводить как на исходном сервере, так и на конечном:
!1.Если конвертация будет на исходном хосте, то воспользуемся командой: RMAN CONVERT TABLESPACE
!2. Если конвертация будет на конечном, то используем команду: RMAN CONVERT DATAFILE
 Мы будем проводить конвертацию на конечном. Для этого нужно скопировать файлы БД на новый сервер, или подключить массив с файлами к новому серверу. 
Для примера ниже скрипт создания команд копирования:
SQL> SELECT distinct 'scp -p oracle@sourceserver:'||substr(file_name, 1, instr(file_name,'TST')+3)||'* /app/oradata/TST/' "Copy files" FROM DBA_DATA_FILES;  2
Copy files
--------------------------------------------------------------------------------
scp -p oracle@sourceserver:/app/oradata/TST/* /app/oradata/TST/
-------------Действия на Linux хосте-------------
1.Предварительно создаём базу 
2.Копируем pfile (create pfile='/tmp/initNEW_DB.ora' from spfile;) с исходного сервера, отредактируем если нужно и положим в каталог $ORACLE_HOME/dbs новой базы 
3. На исходном хосте сгенерируем скрипт создания контрольного файла: ALTER DATABASE BACKUP CONTROLFILE TO TRACE AS '/tmp/ctrlNEW_DB.ora' RESETLOGS; Отредактируем под новый сервер и скопируем в соответствующий каталог. 
4. Создадим файл паролей и запустим экземпляр с полученным файлом параметров, предварительно создав все каталоги указанные в этом файле:
export ORACLE_SID=NEW_DB
orapwd file=${ORACLE_HOME}/dbs/orapw${ORACLE_SID} password=tst
sqlplus / as sysdba
SQL> startup nomount pfile='/oracle/product/11.2.0.2/dbs/initNEW_DB.ora';
5.Конвертация с помощью RMAN:
RMAN TARGET /
convert datafile '/oradata/tmp/tbs1.dbf' from platform 'HP-UX IA (64-bit)' db_file_name_convert '/oradata/tmp','/oradata/new_db/data/';  
{
source_dir="/oratemp/data"
temp_dir="/oradata/wsrmdb1/tempdata"
target_dir="/oradata/wsrmdb1/data"
cd ${temp_dir}
for f in `ls -1 *`; do
  echo "rman target / <<EOF"
  echo "CONVERT DATAFILE '${temp_dir}/${f}' TO PLATFORM=\"Linux x86 64-bit\" FROM PLATFORM=\"HP-UX IA (64-bit)\" DB_FILE_NAME_CONVERT=\"${temp_dir}/\", \"${target_dir}/\";"
  echo "EOF"
  #echo "rm -f ${temp_dir}/${f}"
  echo ""
done
}

и т.д. для каждого файла данных.
6. Импорт дампа
Открываем базу  и импортируем дамп tbs:  
impdb  dumpfile=tr_tbs.dmp  directory=dump_dir transport_datafiles='/oradata/new_db/data/tbs1.dbf','/oradata/new_db/data/tbs2.dbf','/oradata/new_db/data/tbs3.dbf'   logfile=tr_tbs.log
7.  Готово.

Oracle Database Administration































/oradata/loan/data/sysaux01.dbf


alter database datafile '/oradata/loan/data/undotbs01.dbf' resize 100M;
alter database datafile '/oradata/loan/data/users00.dbf' resize 4000M;
alter database datafile '/oradata/loan/data/users01.dbf' resize 20000M;
alter database datafile '/oradata/loan/data/users02.dbf' resize 1200M;
alter database datafile '/oradata/loan/data/users03.dbf' resize 900M;
alter database datafile '/oradata/loan/data/users04.dbf' resize 900M;
alter database datafile '/oradata/loan/data/users05.dbf' resize 900M;
alter database datafile '/oradata/loan/data/users06.dbf' resize 1600M;
alter database datafile '/oradata/loan/data/users07.dbf' resize 1300M;
alter database datafile '/oradata/loan/data/users08.dbf' resize 1000M;



alter database tempfile '/oradata7/slxdb/temp_files/slx_temp02.dbf' resize 5M;


ORA-03297: файл содержит используемые данные вне запрошенного для RESIZE значения

select 'alter table '|| table_name ||' move online tablespace users2;' from dba_segments where tablespace name in ('users'); 




вывести свободное место в таблеспейсах:

SELECT tablespace_name,
       size_mb,
       free_mb,
       max_size_mb,
       max_free_mb,
       TRUNC((max_free_mb/max_size_mb) * 100) AS free_pct,
       RPAD(' '|| RPAD('X',ROUND((max_size_mb-max_free_mb)/max_size_mb*10,0), 'X'),11,'-') AS used_pct
FROM   (
        SELECT a.tablespace_name,
               b.size_mb,
               a.free_mb,
               b.max_size_mb,
               a.free_mb + (b.max_size_mb - b.size_mb) AS max_free_mb
        FROM   (SELECT tablespace_name,
                       TRUNC(SUM(bytes)/1024/1024) AS free_mb
                FROM   dba_free_space
                GROUP BY tablespace_name) a,
               (SELECT tablespace_name,
                       TRUNC(SUM(bytes)/1024/1024) AS size_mb,
                       TRUNC(SUM(GREATEST(bytes,maxbytes))/1024/1024) AS max_size_mb
                FROM   dba_data_files
                GROUP BY tablespace_name) b
        WHERE  a.tablespace_name = b.tablespace_name
       )
ORDER BY tablespace_name;



create tablespace USERS2 DATAFILE '/oradata/loan/data/users00_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M; //создали таблеспейс
ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users01_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users02_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users03_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users04_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users05_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users06_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users07_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;
//ALTER TABLESPACE USERS2 ADD DATAFILE '/oradata/loan/data/users08_1.dbf' SIZE 100M REUSE AUTOEXTEND ON NEXT 1M;


Следует сначала посмотреть таблицы, потом индексы, потом ЛОБы и IOT_TOP
ПОСМОТРЕЛИ таблицы из табличного пространства:

ТАБЛИЦЫ:
select owner, table_name from all_tables where tablespace_name='USERS';
alter table <ИМЯ_СХЕМЫ>.<имя_ТАБЛИЦЫ> move tablespace "<НОВОЕ_ТАБЛИЧНОЕ_ПРОСТРАНСТВо>"; // перенесли из таблицы в tablespace, индесы остаются валидными;

ИНДЕКСЫ:
select owner, index_name, index_type from all_indexes where tablespace_name='USERS';
alter index <ИМЯ_СХЕМЫ>.<ИМЯ_ИНДЕКСА> rebuild tablespace "<НОВОЕ_ТАБЛИЧНОЕ_ПРОСТРАНСТВо>"; \\на всякий перестроили индексы

ПАРТИЦИИ:
select table_owner, table_name, partition_name from all_tab_partitions where tablespace_name='USERS';
alter table <ИМЯ_СХЕМЫ>.<имя_ТАБЛИЦЫ> move partition <ИМЯ_ПАРТИЦИИ> tablespace "<НОВОЕ_ТАБЛИЧНОЕ_ПРОСТРАНСТВо>";

ИНДЕКСЫ ПАРТИЦИЙ:
select table_owner, table_name, partition_name from all_index_partitions where tablespace_name='USERS';
ALTER INDEX <ИМЯ_ИНДЕКСА> REBUILD PARTITION <ИМЯ_ПАРТИЦИИ> TABLESPACE "<НОВОЕ_ТАБЛИЧНОЕ_ПРОСТРАНСТВо>";

ЛОБЫ:
select owner, table_name, column_name, segment_name from dba_lobs where tablespace_name='USERS';
alter table <ИМЯ_СХЕМЫ>.<имя_ТАБЛИЦЫ> move lob (<ИМЯ_КОЛОНКИ>) store as (<НОВОЕ_ТАБЛИЧНОЕ_ПРОСТРАНСТВо>);


ИНДЕКСЫ_ПАРТИЦИЙ:
select owner, index_name, index_type FROM all_ind_partitions where tablespace_name='USERS';
alter index ..... rebuild partition .... tablespace USERS2; 



ИНДЕКСЫ ЛОБОВ:

alter lobindex <ИМЯ_ИНДЕКСА> rebuild tablespace USERS2; \\перенесли лобы

Перемещение IoT:

alter index    PRPC.SYS_IOT_TOP_88791 rebuild tablespace "USERS2";
Error at line 442
ORA-28650: Первичный индекс для IOT не может быть восстановлен

alter index PRPC.SYS_IOT_TOP_88786 rebuild tablespace "USERS2";
Error at line 443
ORA-28650: Первичный индекс для IOT не может быть восстановлен

Делаем:
ALTER TABLE PRPC.DR$AB_DATA_DICTCOMPNME_IDX$K MOVE TABLESPACE USERS2;
ALTER TABLE PRPC.DR$AB_DATA_DICTCOMPNME_IDX$N MOVE TABLESPACE USERS2;


ПРОСМОТРЕЛИ ЛОБЫ:


ПРОСМОТРЕТЬ ПАРТИЦИИ:

select table_owner, table_name, partition_name from all_tab_partitions where tablespace_name='USERS';







CREATE TABLESPACE USERS DATAFILE 
  '/oradata/loan/data/users08.dbf' SIZE 7360M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users07.dbf' SIZE 16865M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users06.dbf' SIZE 28544M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users05.dbf' SIZE 32704M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users04.dbf' SIZE 32704M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users03.dbf' SIZE 32704M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users02.dbf' SIZE 32704M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users00.dbf' SIZE 32704M AUTOEXTEND ON NEXT 16M MAXSIZE UNLIMITED,
  '/oradata/loan/data/users01.dbf' SIZE 33554416K AUTOEXTEND ON NEXT 100M MAXSIZE UNLIMITED
LOGGING
ONLINE
PERMANENT
EXTENT MANAGEMENT LOCAL AUTOALLOCATE
BLOCKSIZE 8K
SEGMENT SPACE MANAGEMENT AUTO
FLASHBACK ON;
