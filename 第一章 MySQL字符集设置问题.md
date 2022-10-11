# 第一章 MYSQL字符集设置

## 1.1 查看默认使用的字符集

```mysql
show variables like 'character%';
```

执行结果：

![MySQL8.0执行结果](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220911174943.png)

- `character_set_server`服务器级别的字符集
- `character_set_database`数据库级别的字符集
- `character_set_client`服务器解码请求时使用的字符集
- `character_set_results`服务器向客户端返回数据时使用的字符集

### 常见字符集

`utf8`：表示一个字符需要使用1-4个字节

`utf8mb3`：阉割过的utf8字符集，使用1-3个字节表示字符

`utf8mb4`：正宗的utf8字符集，使用1-4个字节表示字符

#### 字符集比较规则

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220621080353.png)

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220621080419.png)

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220621080444.png)

#### 字符集比较规则解释

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220621080002.png)

## 1.2 字符集在Linux和Windows下的区别

`关键字和函数名`是不用区分字母大小写的，比如 SELECT、WHERE、ORDER、GROUP BY 等关 键字，以及 ABS、MOD、ROUND、MAX 等函数名

- win系统默认大小写不敏感
- linux系统大小写敏感

### 命令查看

```mysql
show variables like '%lower_case_table_names%';
```

Linux操作系统下执行的结果：

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220911180123.png)

1. 默认为0，大小写敏感 
2. 设置1，大小写不敏感。创建的表，数据库都是以小写形式存放在磁盘上，对于sql语句都是转 换为小写对表和数据库进行查找
3. 设置2，创建的表和数据库依据语句上格式存放，凡是查找都是转换为小写进行

## 1.3 语法校验规则的设置 sql_mode

### 查看当前的sql_mode

```mysql
# 会话级别的sql_mode
select @@session.sql_mode;

# 全局的sql_mode
select @@global.sql_mode;

# 或者
show variables like 'sql_mode';
```

### 常见sql_mode解释

- ONLY_FULL_GROUP_BY：对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中
- NO_AUTO_VALUE_ON_ZERO：该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自增长值。如果用户希望插入的值为0，而该列又是自增长的，那么这个选项就有用了。
- STRICT_TRANS_TABLES：如果一个值不能插入到一个事务中，则中断当前的操作，对非事务表不做限制
- NO_ZERO_IN_DATE：不允许日期和月份为零
- NO_ZERO_DATE：mysql数据库不允许插入零日期，插入零日期会抛出错误而不是警告
- ERROR_FOR_DIVISION_BY_ZERO：在insert或update过程中，如果数据被零除，则产生错误而非警告。如果未给出该模式，那么数据被零除时Mysql返回NULL
- NO_AUTO_CREATE_USER：禁止GRANT创建密码为空的用户
- NO_ENGINE_SUBSTITUTION：如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常
- PIPES_AS_CONCAT：将"||"视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样是，也和字符串的拼接函数Concat想类似
- ANSI_QUOTES：不能用双引号来引用字符串，因为它被解释为识别符

### 设置sql_mode

临时设置：

```mysql
# 设置严格模式 --> 临时设置
# 会话级别
set session sql_mode = 'STRICT_TRANS_TABLES';
# 全局设置
set global sql_mode = 'STRICT_TRANS_TABLES';

```

永久设置：在配置文件中 linux下是 my.cnf文件（win下是 my.ini 文件）直接配置sql_mode

> 线上的生产环境应该采用`临时设置+永久设置的方式`来解决线上禁止重启mysql的问题

