# 使用存储过程分区 Zabbix MySQL(8) 数据库中的表;
# Partitioning tables in the Zabbix MySQL (8) database using stored procedures;


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

## 针对两个SQL文件的解释：
## Explanation of two SQL files:
