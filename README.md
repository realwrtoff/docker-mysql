### 利用rancher和docker镜像搭建mysql主从

1. rancher & 两台物理机 A 和 B， A作为master， B作为slave

   mysql磁盘路径为/data/mysql， 下面有子目录data和mysql配置文件

2. 在rancher应用商店里查找mysql镜像，使用mysql 5.7.18版本，启动两个一模一样的镜像，命名分别为mysql-master, mysql-slave;

3. 升级mysql-master

​	 mysql 添加端口映射 3306:3306 ，

​         mysql 调度规则 只能在主机A上启动；

​	mysql 添加卷 /data/mysql/master.conf:/etc/mysql/conf.d/master.cnf

​	 mysql-data 调度规则 只能在主机A上启动；

​	 mysql-data 添加卷 /data/mysql/data:/var/lib/mysql

4. 升级mysql-slave

   mysql 添加端口映射 3306:3306 ，

   mysql 调度规则 只能在主机B上启动；

   mysql 添加卷 /data/mysql/slave.conf:/etc/mysql/conf.d/master.cnf

   mysql-data 调度规则 只能在主机B上启动；

   mysql-data 添加卷 /data/mysql/data:/var/lib/mysql

5. 连接A上的mysql， 创建备份用户，并查看当前状态

   ```sql
   CREATE USER 'repl'@'%' IDENTIFIED BY '123456';
   GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';
   ```

   ```sql
   mysql> show master status;
   +--------------+---------+------------+----------------+
   |File          | Position|Binlog_Do_DB|Binlog_Ignore_DB|
   +--------------+---------+------------+----------------+
   |mysql-bin.000001| 154   |            | mysql          |
   +--------------+---------+------------+----------------+
   1 row in set (0.00 sec)
   ```

6. 连接B上的mysql， 建立主从关系

   ```sql
   change master to master_host='172.17.0.2', master_user='repl', master_password='123456', master_port=3306, master_log_file='mysql-bin.000001', master_log_pos= 0, master_connect_retry=30;
   ```

   备注：master_host 本来应该填写master容器的ip，考虑到重启后会发生变动，我们做了端口映射，所以通过物理主机内网ip可以访问到master，所以这里填写物理主机内网ip。

   查看主从同步状态， 开启复制过程

   ```
   show slave status \G;
   start slave;
   show slave status \G;
   ```

7. 测试主从复制

   在master 创建数据库， 在slave上查询是否存在该数据库。

   ​

