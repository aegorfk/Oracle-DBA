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
