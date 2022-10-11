# 第二章 MySQL目录结构

## 2.1 主要目录结构

```bash
# linux 下查看指令
find / -name mysql
```

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220911181547.png)

- /usr/bin 相关命令目录

### 数据库文件的存放路径

```mysql
show variables like 'datadir';
```

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220911181442.png)

## 2.2 数据库和文件系统

### 默认数据库

```mysql
# 查看当前所有数据库
SHOW DATABASES;
```

有4个数据库是属于MySQL自带的系统数据库：

- `mysql`：MySQL 系统自带的核心数据库，它存储了MySQL的用户账户和权限信息，一些存储过程、事件的定义信息，一些运行过程中产生的日志信息，一些帮助信息以及时区信息等
- `information_schema`：这个数据库保存着MySQL服务器维护的所有其他数据库的信息，比如有 哪些表、哪些视图、哪些触发器、哪些列、哪些索引
- `performance_schema`：这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，可以用来监控MySQL服务的各类性能指标
- `sys`：这个数据库主要是通过 视图 的形式把 `information_schema` 和 `performance_schema` 结合起来，帮助系统管理员和开发人员监控 MySQL 的技术性能

### 系统表空间OR独立表空间

1. 系统表空间（system tablespace）：

   ```nginx
   就是我们常说的共享表空间，系统表空间（在操作系统上体现就是ibdata文件）是我们在初始化mysql实例时生成的（在初始化mysql实例时会读取my.cnf中的innodb_data_file_path参数，然后初始出相应的文件ibdata1、ibdata2 …，至于文件多大，有多少个，看你my.cnf中的参数是怎样设置的）
   ```

2. 独立表空间（file-per-table tablespaces）：

   ```nginx
   从mysql 5.6.6版本开始，独立表空间（file-per-table tablespaces）默认是开启的（也就是innodb_file_per_table参数不设置时，它默认等于1），在开启的情况下，你创建一个innodb引擎的表，那么表有自己独立的一些数据文件
   ```

参考文档：

- [(52条消息) MySQL系统表空间和独立表空间_阿 亮的博客-CSDN博客_mysql 系统表空间](https://blog.csdn.net/weixin_43733154/article/details/106913924)

#### 查询操作

```mysql
# 查询使用的是 系统表空间 还是 独立表空间
show variables like 'innodb_file_per_table';
```

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220911182818.png)

ON : 开启 ，使用的是独立表空间

