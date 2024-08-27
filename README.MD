# DB SHarding with ProxySQL

## Initial Configuration
reference proxysql [documentation](https://proxysql.com/documentation/proxysql-configuration/)
1. login to proxysql admin
    ```bash
    mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt 'ProxySQL Admin> '
    ```
2. Verify that the configuration is empty
    ```bash
    ProxySQL Admin> SELECT * FROM mysql_servers;
    Empty set (0.00 sec)

    ProxySQL Admin> SELECT * from mysql_replication_hostgroups;
    Empty set (0.00 sec)

    ProxySQL Admin> SELECT * from mysql_query_rules;
    Empty set (0.00 sec)
    ```
3. Add backends, the configuration from the official are different for a little bit. Because we're running with 2 mariadb database in local docker, here the setup
    ```bash
    ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'mariadb1',3306);
    Query OK, 1 row affected (0.01 sec)

    ProxySQL Admin> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'mariadb2',3306);
    Query OK, 1 row affected (0.01 sec)
    ```
4. check the host in mysql_servers
    ```bash
    ProxySQL Admin> SELECT * FROM mysql_servers;
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    | hostgroup_id | hostname | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    | 1            | mariadb1 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
    | 2            | mariadb2 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    2 rows in set (0.000 sec)
    ```
5. Configure monitoring. create user monitor for both mariadb server **mariadb1** and **mariadb2**
    ```bash
    MariaDB> CREATE USER 'monitor'@'%' IDENTIFIED BY 'monitor';
    Query OK, 1 row affected (0.00 sec)
    MariaDB> GRANT USAGE, REPLICATION CLIENT ON *.* TO 'monitor'@'%';
    Query OK, 0 rows affected (0.00 sec)
    ```
6. Then add the credentials of the monitor user to ProxySQL
    ```bash
    ProxySQL Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_username';
    Query OK, 1 row affected (0.00 sec)

    ProxySQL Admin> UPDATE global_variables SET variable_value='monitor' WHERE variable_name='mysql-monitor_password';
    Query OK, 1 row affected (0.00 sec)
    ```
7. Then configure the various monitoring intervals
    ```bash
    ProxySQL Admin> UPDATE global_variables SET variable_value='2000' WHERE variable_name IN ('mysql-monitor_connect_interval','mysql-monitor_ping_interval','mysql-monitor_read_only_interval');
    Query OK, 3 rows affected (0.00 sec)
    ProxySQL Admin> SELECT * FROM global_variables WHERE variable_name LIKE 'mysql-monitor_%';
    +----------------------------------------------------------------------+----------------+
    | variable_name                                                        | variable_value |
    +----------------------------------------------------------------------+----------------+
    | mysql-monitor_enabled                                                | true           |
    | mysql-monitor_connect_timeout                                        | 600            |
    | mysql-monitor_ping_max_failures                                      | 3              |
    | mysql-monitor_ping_timeout                                           | 1000           |
    | mysql-monitor_aws_rds_topology_discovery_interval                    | 1000           |
    | mysql-monitor_read_only_max_timeout_count                            | 3              |
    | mysql-monitor_replication_lag_group_by_host                          | false          |
    | mysql-monitor_replication_lag_interval                               | 10000          |
    | mysql-monitor_replication_lag_timeout                                | 1000           |
    | mysql-monitor_replication_lag_count                                  | 1              |
    | mysql-monitor_groupreplication_healthcheck_interval                  | 5000           |
    | mysql-monitor_groupreplication_healthcheck_timeout                   | 800            |
    | mysql-monitor_groupreplication_healthcheck_max_timeout_count         | 3              |
    | mysql-monitor_groupreplication_max_transactions_behind_count         | 3              |
    | mysql-monitor_groupreplication_max_transactions_behind_for_read_only | 1              |
    | mysql-monitor_galera_healthcheck_interval                            | 5000           |
    | mysql-monitor_galera_healthcheck_timeout                             | 800            |
    | mysql-monitor_galera_healthcheck_max_timeout_count                   | 3              |
    | mysql-monitor_replication_lag_use_percona_heartbeat                  |                |
    | mysql-monitor_query_interval                                         | 60000          |
    | mysql-monitor_query_timeout                                          | 100            |
    | mysql-monitor_slave_lag_when_null                                    | 60             |
    | mysql-monitor_threads_min                                            | 8              |
    | mysql-monitor_threads_max                                            | 128            |
    | mysql-monitor_threads_queue_maxsize                                  | 128            |
    | mysql-monitor_local_dns_cache_ttl                                    | 300000         |
    | mysql-monitor_local_dns_cache_refresh_interval                       | 60000          |
    | mysql-monitor_local_dns_resolver_queue_maxsize                       | 128            |
    | mysql-monitor_wait_timeout                                           | true           |
    | mysql-monitor_writer_is_also_reader                                  | true           |
    | mysql-monitor_username                                               | monitor        |
    | mysql-monitor_password                                               | monitor        |
    | mysql-monitor_history                                                | 600000         |
    | mysql-monitor_connect_interval                                       | 2000           |
    | mysql-monitor_ping_interval                                          | 2000           |
    | mysql-monitor_read_only_interval                                     | 2000           |
    | mysql-monitor_read_only_timeout                                      | 500            |
    +----------------------------------------------------------------------+----------------+
    37 rows in set (0.002 sec)
    ```
8. Changes made to the MySQL Monitor in table global_variables will be applied after executing the LOAD MYSQL VARIABLES TO RUNTIME statement. To persist the configuration changes across restarts the SAVE MYSQL VARIABLES TO DISK must also be executed.
    ```bash
    ProxySQL Admin> LOAD MYSQL VARIABLES TO RUNTIME;
    Query OK, 0 rows affected (0.00 sec)

    ProxySQL Admin> SAVE MYSQL VARIABLES TO DISK;
    Query OK, 54 rows affected (0.02 sec)
    ```
9. Backend’s health check
    ```bash
    ProxySQL Admin> LOAD MYSQL SERVERS TO RUNTIME;
    Query OK, 0 rows affected (0.00 sec)

    ProxySQL Admin> SELECT * FROM mysql_servers;
    ProxySQL Admin> SELECT * FROM mysql_servers;
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    | hostgroup_id | hostname | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    | 1            | mariadb1 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
    | 2            | mariadb2 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
    +--------------+----------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
    2 rows in set (0.000 sec)
    ```
10. Once the configuration is active, it’s possible to verify the status of the MySQL backends in the monitor database tables in ProxySQL Admin:
    ```bash
    ProxySQL Admin> SHOW TABLES FROM monitor;
    +--------------------------------------+
    | tables                               |
    +--------------------------------------+
    | mysql_server_aws_aurora_check_status |
    | mysql_server_aws_aurora_failovers    |
    | mysql_server_aws_aurora_log          |
    | mysql_server_connect_log             |
    | mysql_server_galera_log              |
    | mysql_server_group_replication_log   |
    | mysql_server_ping_log                |
    | mysql_server_read_only_log           |
    | mysql_server_replication_lag_log     |
    +--------------------------------------+
    9 rows in set (0.001 sec)
    ```
11. Each check type has a dedicated logging table, each should be checked individually
    ```bash
    ProxySQL Admin> SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 4;
    +----------+------+------------------+-------------------------+---------------+
    | hostname | port | time_start_us    | connect_success_time_us | connect_error |
    +----------+------+------------------+-------------------------+---------------+
    | mariadb1 | 3306 | 1724647557276414 | 1251                    | NULL          |
    | mariadb2 | 3306 | 1724647557247867 | 1309                    | NULL          |
    | mariadb1 | 3306 | 1724647555279554 | 735                     | NULL          |
    | mariadb2 | 3306 | 1724647555247224 | 784                     | NULL          |
    +----------+------+------------------+-------------------------+---------------+
    4 rows in set (0.000 sec)
    ProxySQL Admin> SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 5;
    +----------+------+------------------+----------------------+------------+
    | hostname | port | time_start_us    | ping_success_time_us | ping_error |
    +----------+------+------------------+----------------------+------------+
    | mariadb2 | 3306 | 1724647621263805 | 381                  | NULL       |
    | mariadb1 | 3306 | 1724647621263592 | 538                  | NULL       |
    | mariadb2 | 3306 | 1724647619263040 | 278                  | NULL       |
    | mariadb1 | 3306 | 1724647619262887 | 380                  | NULL       |
    | mariadb1 | 3306 | 1724647617262481 | 152                  | NULL       |
    +----------+------+------------------+----------------------+------------+
    5 rows in set (0.001 sec)
    ```
Initial configuration ends with connect and ping test to database **mariadb1** and **mariadb2** with **ProxySQL**

## MariaDB Configuration
1. login to database container **mariadb1** and **mariadb2**
    ```bash
    mysql -u root -ppassword
    ```
2. create new `root` user with host `%` and grant all privileges
    ```sql
    create user 'root'@'%' identified by 'password';
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';
    flush privileges;
    ```
3. create new table `user` for `db_home`
    ```sql
    CREATE TABLE user (
        id INT AUTO_INCREMENT PRIMARY KEY,
        nama VARCHAR(255) NOT NULL,
        password VARCHAR(255) NOT NULL,
        location VARCHAR(255) NOT NULL
    );
    ```
## Setup Rules
you need create rules to make your data spread to database
1. login to proxysql
    ```bash
    mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt 'ProxySQL Admin> '
    ```
2. add new rules
    ```sql
    ProxySQL Admin> INSERT INTO mysql_query_rules (match_pattern, destination_hostgroup, apply, active) VALUES 
    ( "shard_sby", 1, 1, 1);
    ProxySQL Admin> INSERT INTO mysql_query_rules (match_pattern, destination_hostgroup, apply, active) VALUES 
    ( "shard_jkt", 2, 1, 1);
    ```
3. check rules
    ```sql
    ProxySQL Admin> select match_digest, destination_hostgroup, apply from mysql_query_rules;
    ```
3. Reload rules and apply changes
    ```sql
    ProxySQL Admin> LOAD MYSQL QUERY RULES TO RUNTIME;
    ProxySQL Admin> SAVE MYSQL QUERY RULES TO DISK;
    ```
## Add User
add user from mariadb to proxysql to enable sending query through proxysql
```sql
ProxySQL Admin> INSERT INTO mysql_users (username, password, default_hostgroup) VALUES ('root', 'password', 1);
Query OK, 1 row affected (0.000 sec)

ProxySQL Admin> LOAD MYSQL USERS TO RUNTIME;
Query OK, 0 rows affected (0.001 sec)

ProxySQL Admin> SAVE MYSQL USERS TO DISK;
Query OK, 0 rows affected (0.031 sec)
```
## Test
1. try connecting to proxysql, before insert data
    ```bash
    mysql -c -h 127.0.0.1 -P 6033 -u root -ppassword
    ```
2. CRUD
    1. insert data. Run following command in **proxysql** container
        ```sql
        mysql -c -h 127.0.0.1 -P 6033 -u root -ppassword -e "INSERT INTO /* shard_sby */ db_home.user (nama, password, location) VALUES ('User1', 'password', 'surabaya')";
        mysql -c -h 127.0.0.1 -P 6033 -u root -ppassword -e "INSERT INTO /* shard_jkt */  db_home.user (nama, password, location) VALUES ('User2', 'password', 'jakarta')";
        ```