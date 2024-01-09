# 使用存储过程自动化分区管理 Zabbix MySQL(8) 数据库中的大表;
# auto Partitioning tables in the Zabbix MySQL (8) database using stored procedures;


初始化操作一次之后,之后的分区操作都是自动化的**定期每天凌晨4点操作**，迁移的时候记得把存储过程和事件都迁移走!

After the initial setup, all subsequent partition operations are automated. Remember to migrate the stored procedures and events as well when migrating!


这篇文章会是一行中文对应一行翻译后的英文，就和看电影一样

This article will have one line of Chinese corresponding to one line of translated English, just like watching a movie in China.

```text
Chinese
English
```

你是否为zabbix的几个大表头疼

Are you feeling a headache for some of Zabbix's big tables

```sql
history     
history_log 
history_str 
history_text
history_uint
trends     
trends_uint 
```
## 如何使用 (HOW TO USE)
> **注意:** 下面的操作是在您安装好zabbix服务器和mysql后并且在官方的指导文件中已经把库表创建好了。

> **Notice:** The following operation is to create the library table in the official guide file after you have installed the Zabbix server and MySQL.


```shell
git clone https://github.com/hanzhongzi/zabbix_pratition_tables.git
cd zabbix_pratition_tables
sed -i 's/zabbix_server/{YOUR_ZABBIX_DATABASE_NAME}/g' zabbix_alter_tables.sql
mysql -u{USERNAME} -h{MYSQLHOST} -p{MYSQLPASSWORD}  {YOUR_ZABBIX_DATABASE_NAME} < zabbix_alter_tables.sql
sed -i 's/zabbix_server/{YOUR_ZABBIX_DATABASE_NAME}/g' zabbix_pratition_tables.sql
mysql -u{USERNAME} -h{MYSQLHOST} -p{MYSQLPASSWORD}  {YOUR_ZABBIX_DATABASE_NAME} < zabbix_pratition_tables.sql
```

## 针对两个SQL文件的解释:
```text
一.在您按照官方文档导入数据表和基础数据后,首先在您命名的数据库中执行zabbix_alter_tables.sql中的SQL。
目的是基于 clock 字段的值，将history, history_log, history_str, history_text, history_uint, trends, trends_uint表都被分成多个分区,每个分区都有一个特定的值范围。
分区的值范围是通过 UNIX_TIMESTAMP(DATE_FORMAT(DATE_SUB(CURDATE(), INTERVAL 1 YEAR), '%Y-%m-%d')) 计算得到的。这表示每个分区的上限是当前日期一年前的某一天的 Unix 时间戳。
定一年前的某一天是为了不影响当前的数据，和待会儿测试删除分区。


二.之后在您命名的数据库中执行zabbix_pratition_tables.sql中的SQL。
这段 SQL 脚本是为上面说的那些大表分区管理设计的。让我们逐步解析这个脚本的关键部分：
1.创建 manage_partitions 表：
此表创建用于跟踪不同表的分区管理方式。它包括字段，如表名、分区周期（每天或每月）、保留分区的时长（以天或月计）、最后一次更新分区的时间，以及额外的评论。
2.存储过程：
脚本定义了几个用于创建和删除分区的存储过程。
create_next_partitions：此过程遍历 manage_partitions 表，并为列出的每个表创建下一组分区。它根据 manage_partitions 表的 period 列确定是创建每日分区还是每月分区。
create_partition_by_day 和 create_partition_by_month：这些过程被 create_next_partitions 调用，负责按日或按月创建单个分区。
drop_partitions：此过程检查基于 manage_partitions 表中的 keep_history 值需要删除的旧分区。
drop_old_partition：由 drop_partitions 调用，此过程实际上从表中删除指定的分区。
3.创建计划事件（e_part_manage）：
此事件被安排在每天上午 4:00 运行。它调用 zabbix_server 架构的 drop_partitions 和 create_next_partitions 过程，确保根据 manage_partitions 中定义的规则定期管理分区。
4.为 manage_partitions 插入语句：
这些语句将初始配置数据插入 manage_partitions 表，适用于各种表，如 history、history_uint、history_str、history_text、history_log、trends 和 trends_uint。
每个条目指定了保留该表分区的时长（例如，每日分区 60 天，每月分区 12 个月）。
总之，这个脚本自动化了 Zabbix 服务器数据库的未来大表分区管理过程。它建立了一个系统，根据定义的标准定期创建和删除指定表的分区，帮助管理数据增长和优化数据库性能，这种方法特别适用!
```
## Explanation of two SQL files:
```text
1.zabbix_alter_tables.sql:
After importing the data tables and base data following the official documentation, you should first execute the SQL in zabbix_alter_tables.sql within the database you named. The purpose is to partition the tables history, history_log, history_str, history_text, history_uint, trends, and trends_uint based on the values of the clock field. Each partition will have a specific value range.

The value range for each partition is calculated using UNIX_TIMESTAMP(DATE_FORMAT(DATE_SUB(CURDATE(), INTERVAL 1 YEAR), '%Y-%m-%d')). This means the upper limit of each partition is the Unix timestamp of a specific day one year before the current date.

Setting the date to one year before is done to avoid affecting current data and to prepare for the upcoming test of partition deletion.


2.zabbix_pratition_tables.sql:
Creation of manage_partitions Table:

This table is created to keep track of how partitions should be managed for different tables. It includes fields like table name, partitioning period (daily or monthly), how long to keep the partitions (in days or months), when the partition was last updated, and additional comments.
Stored Procedures:

The script defines several stored procedures for creating and dropping partitions.
create_next_partitions: This procedure iterates through the manage_partitions table and creates the next set of partitions for each table listed. It decides whether to create daily or monthly partitions based on the period column of the manage_partitions table.
create_partition_by_day and create_partition_by_month: These procedures are called by create_next_partitions and are responsible for creating individual partitions on a daily or monthly basis.
drop_partitions: This procedure checks for old partitions that need to be dropped based on the keep_history value from the manage_partitions table.
drop_old_partition: Called by drop_partitions, this procedure actually drops the specified partition from a table.
Creation of a Scheduled Event (e_part_manage):

This event is scheduled to run daily at 4:00 AM. It calls the drop_partitions and create_next_partitions procedures for the zabbix_server schema, ensuring that partitions are regularly managed according to the rules defined in manage_partitions.
Insert Statements for manage_partitions:

These statements insert initial configuration data into the manage_partitions table for various tables like history, history_uint, history_str, history_text, history_log, trends, and trends_uint.
Each entry specifies how long to keep partitions for that table (e.g., 60 days for daily partitions, 12 months for monthly partitions).
In summary, this script automates the process of partition management for a Zabbix server database. It sets up a system to regularly create and drop partitions for specified tables based on defined criteria, helping to manage data growth and optimize database performance. This approach is particularly useful in scenarios where tables grow large over time, such as in monitoring or logging systems like Zabbix.
```

## 如何测试
``` sql
mysql> show events \G;
mysql> SHOW CREATE EVENT zabbix_server.e_part_manage\G;
-- 你会看到这个事件要执行什么;You will see what this event is going to execute."
-- CREATE DEFINER=`root`@`localhost` EVENT `e_part_manage` ON SCHEDULE EVERY 1 DAY STARTS '2021-02-19 04:00:00' ON COMPLETION PRESERVE ENABLE COMMENT 'Creating and dropping partitions' DO BEGIN
-- CALL zabbix_server.drop_partitions('zabbix_server');
-- CALL zabbix_server.create_next_partitions('zabbix_server');
-- END
--
CALL zabbix_server.create_next_partitions('zabbix_server');
-- #去执行 CALL zabbix_server.create_next_partitions('zabbix_server'); 会在我们上面说的未来会很大很头疼的表中新建一个分区~，日期是今天正常滴可以有数据写入进来。
-- #去执行 CALL zabbix_server.drop_partitions('zabbix_server');  会删除之前咱们在 zabbix_alter_tables.sql 中增加的表分区。
CALL zabbix_server.drop_partitions('zabbix_server');
-- # To execute CALL zabbix_server.create_next_partitions('zabbix_server'); which will create a new partition in the tables we discussed earlier that are going to be large and troublesome in the future~. The date is today and data can normally be written into it.
-- # To execute CALL zabbix_server.drop_partitions('zabbix_server'); will delete the table partitions we added earlier in zabbix_alter_tables.sql.
-- #如果再次执行 CALL zabbix_server.create_next_partitions('zabbix_server'); 会报错各个表的分区已经新建了。
-- # If CALL zabbix_server.create_next_partitions('zabbix_server') is executed again, it will throw an error stating that the partitions for each table have already been created.
```

# 如果有任何问题欢迎您提issue，如果对您有用请给我点个star。这将鼓励我去做出更多帮助大家的项目
# If you have any questions, feel free to open an issue. If you find it useful, please give me a star. This will encourage me to create more projects that help everyone.

