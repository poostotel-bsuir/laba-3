# 🎯 Lab 3: SQL Server high availability and disaster recovery

## 💡 Introduction
The DBMS Semester 2 curriculum involves learning how to administer, maintain and manage data warehouse infrastructure. As part of the curriculum update, it was decided to provide students with the opportunity to perform laboratory work using containerisation tools instead of virtualisation tools. This series of GitHub materials will provide a methodological guide for performing basic tasks from the mandatory labs using Docker containers with MS SQL Server on board.

**The report to the laboratory work should be prepared in Microsoft Word according to ***СТП 2024*** and attached in .pdf format to the repository files.**

All tasks could be done using official documentation or any other resources.  Try to do it using documentation.

> Happy Hunger Games and may the odds be ever in your favor!

## 🔗 Resources

- [СТП 2024](https://www.bsuir.by/m/12_100229_1_185586.pdf)

- [Инструкция по установке Docker Desktop и развёртыванию вашего первого контейнера с MS SQL Server](https://github.com/poostotel-bsuir/laba-1/blob/main/%D0%98%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BA%D1%86%D0%B8%D1%8F_%D0%BF%D0%BE_MS_SQL_%D0%BD%D0%B0_%D0%B1%D0%B0%D0%B7%D0%B5_Docker_Desktop.docx)

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)

- [About log shipping (SQL Server)](https://learn.microsoft.com/en-us/sql/database-engine/log-shipping/about-log-shipping-sql-server?view=sql-server-ver15)

- [Database Mirroring (SQL Server)](https://learn.microsoft.com/en-us/sql/database-engine/database-mirroring/database-mirroring-sql-server?view=sql-server-ver15)

- [Business continuity and database recovery - SQL Server](https://learn.microsoft.com/en-us/sql/database-engine/sql-server-business-continuity-dr?view=sql-server-ver15)

-  [Windows Server Failover Clustering with SQL Server](https://learn.microsoft.com/en-us/sql/sql-server/failover-clusters/windows/windows-server-failover-clustering-wsfc-with-sql-server?view=sql-server-ver15)

## 📚 Objective
Understand in the deep mechanism of high availability in MS SQL Server.

## 📝 Tasks

### 📌 Task 1: Configure database mirroring (in fast mode)
- Mirroring is working.
- You can drop original db (make inactive), and second will have the same data.
- Second DB could be up and running.

### 📌 Task 2: Configure log shipping
- Log shipping is working.
- If you drop first db, second has actual data, and could be up and running.

### 📌 Task 3: Configure replication
- Replication is working, data is distributing to different databases.

### ⚡ Extra task 1: Configure Failover  Cluster of MS SQL
- Failover cluster is running, if one of the instance is failed – you still could connect to endpoint.
- Data is actual and the same between all servers.

## 📙 Manual

### 🟡 Настройка зеркалирования

Для настройки зеркалирования между контейнерами необходимо создать необходимые сертификаты, а также открыть порт для зеркалирования.

На главном сервере выполните следующие команды:
```sql
USE master;
ALTER DATABASE myTest_db SET RECOVERY FULL;
BACKUP DATABASE myTest_db 
TO DISK = '/var/opt/mssql/Backups/myTest_db_Full.bak' 
WITH INIT;
BACKUP LOG myTest_db 
TO DISK = '/var/opt/mssql/Backups/myTest_db_Log.trn' 
WITH INIT;
```

Теперь данные бэкапы необходимо перенести на второй сервер (с помощью ssh также как в предыдущих лабораторных работах) и восстановить с параметром ‘NORECOVERY’:
```sql
USE master;
RESTORE DATABASE myTest_db 
FROM DISK = '/var/opt/mssql/Backups/myTest_db_Full.bak' 
WITH REPLACE, NORECOVERY,
MOVE 'AdventureWorksDW2012' TO '/var/opt/mssql/data/myTest_db.mdf',
MOVE 'AdventureWorksDW2012_log' TO
'/var/opt/mssql/Logs/myTest_db_log.ldf';
RESTORE LOG myTest_db 
FROM DISK = '/var/opt/mssql/Backups/myTest_db_Log.trn' 
WITH NORECOVERY;
```

Далее необходимо создать сертификаты и также перенести их на оба сервера и открыть порт зеркалирования на каждом сервере:
```sql
USE master;
--Создайте мастер ключ, если он ещё не создан, или же откройте существующий командой:
--OPEN MASTER KEY DECRYPTION BY PASSWORD = 'YourStrongPassword123!';
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword123!';
CREATE CERTIFICATE my_vm1_cert WITH SUBJECT = 'my_vm1 Certificate';
CREATE ENDPOINT Mirroring_endpoint
 	   STATE = STARTED
               AS TCP (LISTENER_PORT = 5022)
FOR DATABASE_MIRRORING (ROLE = PARTNER, AUTHENTICATION = CERTIFICATE my_vm1_cert, ENCRYPTION = REQUIRED ALGORITHM AES);
BACKUP CERTIFICATE my_vm1_cert TO FILE = '/var/opt/mssql/Backups/my_vm1_cert.cer';
```

> Если вы вдруг забыли пароль с котором ранее создавали мастер ключ выполните следующие команды, при это ранее созданные сертификаты станут не действительны и их придётся пересоздать снова.
```sql
ALTER SERVICE MASTER KEY FORCE REGENERATE;
ALTER MASTER KEY FORCE REGENERATE WITH ENCRYPTION BY PASSWORD = 'YourStrongPassword123!';
```

Перенесите созданный сертификат (my_vm1_cert.cer) на второй сервер и создайте его из перенесённого бэкапа:
```sql
USE master; 
CREATE LOGIN my_vm1_login WITH PASSWORD = 'YourStrongPassword123!';
CREATE USER my_vm1_user FOR LOGIN my_vm1_login;
CREATE CERTIFICATE my_vm1_cert FROM FILE = '/var/opt/mssql/Backups/my_vm1_cert.cer';
GRANT CONNECT ON ENDPOINT::Mirroring_endpoint TO my_vm1_login;
```

Сделайте аналогично с сертификатом второго сервера и перенесите его на первый аналогичным образом, в итоге в каждом экземпляре сервера будет по два сертификата в нашем примере.

Также на втором контейнере укажите партнёра:
```sql
USE master; ALTER DATABASE myTest_db SET PARTNER = 'TCP://my_vm1:5022';
```

Аналогично на первом выполните:
```sql
USE master; ALTER DATABASE myTest_db SET PARTNER = 'TCP://my_vm2:5022'; 
ALTER DATABASE myTest_db SET SAFETY OFF;
```

В обозревателе объектов SSMS можно увидеть, что базы данных зеркалируются.

Зеркальное отображение базы данных myTest_db

![image](https://github.com/user-attachments/assets/5b2bb192-c889-4226-b7a3-1ca3c7678726)

Теперь для проверки работы зеркалирования на главной реплике (первый контейнер) обновим запись в таблице DimAccount.

Команда обновлния данных записи таблицы DimAccount:
```sql
USE myTest_db;
SELECT AccountCodeAlternateKey FROM DimAccount WHERE AccountKey = 89;
UPDATE DimAccount
SET AccountCodeAlternateKey = 8050
WHERE AccounrKey = 89;
SELECT AccountCodeAlternateKey FROM DimAccount WHERE AccountKey = 89;
```

Обновление записи таблицы DimAccount

![image](https://github.com/user-attachments/assets/f67864fb-d595-4b51-897d-090bc273ee39)

Выполним на обоих серверах команду:
```sql
SELECT mirroring_state_desc, mirroring_role_desc FROM sys.database_mirroring WHERE database_id = DB_ID('myTest_db');
```

И убедимся, что на главном сервере результат:
|mirroring_state_desc|mirroring_role_desc|
|-------------|-------------|
|SYNCHRONIZED|PRINCIPAL|

На реплике:
|mirroring_state_desc|mirroring_role_desc|
|-------------|-------------|
|SYNCHRONIZED|MIRROR|

Теперь спроецируем ситуацию отказа главного сервера командой: 
```powershell
docker stop <название контейнера>
```

Теперь выполним тот же select запрос получим:
|mirroring_state_desc|mirroring_role_desc|
|-------------|-------------|
|DISCONNECTED|MIRROR|

Для того чтобы получить актуальную базу данных на втором сервере необходимо отключить зеркалирование базы данных (SET PARTNER OFF), иначе при попытке восстановить базу данных которая участвует в зеркальном отображении будет получена следующая ошибка:

> The operation cannot be performed on database "XXX" because it is involved in a database mirroring session or an availability group.

Результат восстановления myTest_db

![image](https://github.com/user-attachments/assets/ce18f396-ac6b-4b3c-88a0-f3540f68021f)

В итоге мы получили актуальную версию нашей базы данных и восстановили доступ к данным.

> Для восстановления зеркалирования необходимо заново создать актуальные бэкапы и провести аналогичную настройку зеркального отображения.

### 🟡 Настройка репликации

Репликация позволяет копировать данные (или их часть) с одного сервера (Publisher) на другой (Subscriber), что полезно для распределения нагрузки, отчетов или создания резервных копий данных. Мы настроим Transactional Replication (транзакционную репликацию), так как это один из самых распространенных типов, обеспечивающий почти реальное время передачи изменений.

Мастер создания публикаций

![image](https://github.com/user-attachments/assets/19a51841-f313-4f6a-9cc9-223e674e197d)

Далее выберите базу данных для которой необходимо настроить репликацию, тип публикации “Публикация транзакций”, реплицируемые элементы базы данных (таблицы, представления, пользовательские функции и тд.).

Моментальный снимок

![image](https://github.com/user-attachments/assets/d27e5e7c-8e22-463b-8a24-581f9f711022)

В настройках безопасности агентов (агент моментальных снимков и агент чтения журнала) укажите параметры, в параметрах подключения к издателю укажите логин и пароль сервера на котором настраиваете репликацию, таким образом мастер сервер одновременно будет являться и издателем, и распространителем.

Настройка безопасности аудита

![image](https://github.com/user-attachments/assets/f2512772-f94a-4a31-a733-22dfc0e7a8a9)

Далее укажите имя публикации и нажмите Готово.

Результат создания публикации “TestPub”

![image](https://github.com/user-attachments/assets/16ff9796-9bb5-48b1-9796-a5a8d9829edb)

Теперь создадим подписку для публикации TestPub, для этого выполним следующий скрипт на мастер сервере:
```sql
USE myTest_db;
EXEC sp_addsubscription 
    @publication = 'TestPub', 
    @subscriber = 'ebe0d645f1ac',
    @destination_db = 'myTest_db', 
    @subscription_type = 'push', 
    @sync_type = 'automatic';
```

Для атрибута “@subscriber” укажите наименование сервера подписчика: 
```sql
SELECT @@SERVERNAME;
```

Если ранее была создана ошибочно публикация, то выполните следующую команду:
```sql
EXEC sp_dropsubscription 
    @article = 'all',
    @publication = 'TestPub', 
    @subscriber = 'my_vm2', 
    @destination_db = 'myTest_db';
```

Для проверки успешного подключения к подписчику необходимо запустить монитор репликации. 

Монитор репликации

![image](https://github.com/user-attachments/assets/ae4110bc-3d82-47d6-a46e-6729de63c3d3)

Здесь мы можем просматривать издателей и их подписки, чтобы просмотреть более подробную информацию подписки двойным нажатием выберете нужную подписку. 

Журнал операций от распространителя к подписчику

![image](https://github.com/user-attachments/assets/a445f4c4-dc4c-43d8-bd83-efbcb15516f7)

Можно увидеть, как после успешного подключения к подписчику начался процесс репликации.

Для проверки работы репликации измените произвольные записи на основном сервере и затем выполните select запрос тех же записей на реплике и убедитесь, что данные успешно реплицированы.

> Если в мониторе репликации видна ошибка “Login failed. The login is from an untrusted domain and cannot be used with Integrated authentication. (Источник: MSSQLServer, номер ошибки: 18452)”, то необходимо перейти в свойства подписки и изменить параметры “Соединение с подписчиком".

Свойства подписки

![image](https://github.com/user-attachments/assets/c793b94a-869b-4490-9c2f-129e779f4623)

> Если возникает ситуация, когда попытка синхронизации зависает и не удаётся повторно инициализировать попытку синхронизации с подписчиком, можно вручную остановить работу следующим скриптом:
```sql
USE msdb;
SELECT 
    j.name AS JobName,
    j.job_id AS JobID,
    a.start_execution_date,
    a.stop_execution_date,
    CASE 
        WHEN a.start_execution_date IS NOT NULL AND a.stop_execution_date IS NULL THEN 'Running'
        ELSE 'Not Running'
    END AS JobStatus
FROM sysjobs j
LEFT JOIN sysjobactivity a ON j.job_id = a.job_id
WHERE j.name LIKE '%TestPub%';
```
> Выберете JobName работы со статусом “'Running'” из результата и выполните команду, где “8f39d733d46c-myTest_db-TestPub-MY_VM2-12” это наименование работы:
```sql
EXEC sp_stop_job @job_name = '8f39d733d46c-myTest_db-TestPub-MY_VM2-12';
```

### 🟡 Доставка журналов в MS SQL

Для настройки доставки журналов (log shipping) в MS SQL для начала на мастер сервере необходимо изменить режим восстановления базы данных:
```sql
USE master;
SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'myTest_db';
ALTER DATABASE myTest_db SET RECOVERY FULL;
```

Затем создать бэкапы:
```sql
USE master;
BACKUP DATABASE myTest_db 
TO DISK = '/var/opt/mssql/Backups/myTest_db_Full.bak'
WITH INIT;
BACKUP LOG myTest_db 
TO DISK = '/var/opt/mssql/Backups/myTest_db_Log.trn'
WITH INIT;
```

Перенесём созданные бэкапы на второй сервер с помощью ssh по аналогии с прошлыми лабораторными работами и восстановим их.

Команда восстановления из бэкапа:
```sql
USE master;
RESTORE DATABASE myTest_db 
FROM DISK = '/var/opt/mssql/Backups/myTest_db_Full.bak'
WITH NORECOVERY;
RESTORE LOG myTest_db 
FROM DISK = '/var/opt/mssql/Backups/myTest_db_Log.trn'
WITH NORECOVERY;
```

На втором сервере создадим задание SQL Agent для переноса файла журнала мастер сервера:
```sql
USE msdb;
EXEC sp_add_job 
    @job_name = 'LogShipping_Copy_myTest_db',
    @enabled = 1,
    @description = 'Log Shipping Copy Job for myTest_db using SCP';

EXEC sp_add_jobstep 
    @job_name = 'LogShipping_Copy_myTest_db',
    @step_name = 'Copy Transaction Log Backup via SCP',
    @subsystem = 'CMDEXEC',
    @command = 'sshpass -p "yourRootPassword123!" scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@172.20.0.2:/var/opt/mssql/Backups/myTest_db_Log.trn /var/opt/mssql/Backups/myTest_db_Log.trn',
    @retry_attempts = 5,
    @retry_interval = 5;

EXEC sp_add_jobschedule 
    @job_name = 'LogShipping_Copy_myTest_db',
    @name = 'Every_15_Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15,
    @active_start_date = 20250429,
    @active_start_time = 0;

EXEC sp_add_jobserver 
    @job_name = 'LogShipping_Copy_myTest_db',
    @server_name = @@SERVERNAME;
```

Создадим задание для восстановления файла журнала:
```sql
EXEC sp_add_job 
    @job_name = 'LogShipping_Restore_myTest_db',
    @enabled = 1,
    @description = 'Log Shipping Restore Job for myTest_db';

EXEC sp_add_jobstep 
    @job_name = 'LogShipping_Restore_myTest_db',
    @step_name = 'Restore Transaction Log',
    @subsystem = 'TSQL',
    @command = 'RESTORE LOG myTest_db FROM DISK = ''/var/opt/mssql/Backups/myTest_db_Log.trn'' WITH NORECOVERY;',
    @retry_attempts = 5,
    @retry_interval = 5;

EXEC sp_add_jobschedule 
    @job_name = 'LogShipping_Restore_myTest_db',
    @name = 'Every_15_Minutes',
    @freq_type = 4,
    @freq_interval = 1,
    @freq_subday_type = 4,
    @freq_subday_interval = 15,
    @active_start_date = 20250429,
    @active_start_time = 0;

EXEC sp_add_jobserver 
    @job_name = 'LogShipping_Restore_myTest_db',
    @server_name = @@SERVERNAME;
```

Затем просмотрим файл журнала и увидим, что задание восстановления новых бэкапов успешно выполнено, при возникновении ситуации, когда мастер сервер становится недоступным на втором сервере будет актуальная версия базы данных.

Просмотр файла журнала
![image](https://github.com/user-attachments/assets/d86f2a9b-809b-48cc-99b2-e4fe28ab44d8)


## 🧠 Self-Check Questionnaire

### 🔍 Question 1:
What steps are required to configure database mirroring in high-safety mode? Explain the role of the witness server if used.

### 🔍 Question 2:
How does log shipping ensure data consistency during a primary server failure? Describe the process to promote a secondary server to primary.

### 🔍 Question 3:
What are the key differences between transactional replication and snapshot replication? Provide a use case where transactional replication is preferred.

### 🔍 Question 4:
What prerequisites must be met to configure a Windows Server Failover Cluster (WSFC) for SQL Server? How does automatic failover work in this setup?

### 🔍 Question 5:
How would you validate the integrity of a TDE-encrypted database after restoring it on a secondary server? List the critical steps.
