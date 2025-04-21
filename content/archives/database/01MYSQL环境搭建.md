## MYSQL 环境搭建（ZIP）

下载地址：https://dev.mysql.com/downloads/mysql/

教程：https://www.jb51.net/article/167678.htm

在zip解压目录下，创建一个配置文件为 my.ini，编辑 my.ini 配置以下基本信息：

```ini
[mysqld]

# 设置3306端口

port=3306

# 设置mysql的安装目录，这里根据自己的解压目录来写

basedir=D:\Program Files (x86)\mysql-8.0.17-winx64

# 设置mysql数据库的数据的存放目录，解压后的文件里有个Data文件夹

datadir=D:\Program Files (x86)\mysql-8.0.17-winx64\Data

# 允许最大连接数

max_connections=200

# 允许连接失败的次数。

max_connect_errors=10

# 服务端使用的字符集默认为UTF8

character-set-server=utf8

# 创建新表时将使用的默认存储引擎

default-storage-engine=INNODB

# 默认使用“mysql_native_password”插件认证

#mysql_native_password

default_authentication_plugin=mysql_native_password

[mysql]

# 设置mysql客户端默认字符集

default-character-set=utf8

[client]

# 设置mysql客户端连接服务端时默认使用的端口

port=3306

default-character-set=utf8
 
```

CMD,在BIN目录下输入：

`
mysqld install
`

然后初始化，输入

`mysqld ``--initialize-insecure --user=mysql`

**这时有个临时密码要注意，在登录时候有用**

打开服务

`net start mysql`

登录MYSQL

`mysql -u root -p`

输入**临时密码**

修改临时密码

------

#### 查询用户密码

###### 查询用户密码命令：

```mysql
`mysql> ``select` `host,user,authentication_string from mysql.user;`
```

host： 允许用户登录的ip；

user：当前数据库的用户名；

authentication_string： 用户密码；

如果没密码， root 这一行应该是空的。

#### 正确修改root密码的步骤为：

注意：在MySQL 5.7.9以后废弃了password字段和password()函数

一定不要采取如下形式设置密码：

[?](https://www.jb51.net/article/167678.htm#)

```mysql
`use mysql; ``update user ``set` `authentication_string=``"newpassword"` `where user=``"root"``;`
```

这样会给user表中root用户的authentication_string字段下设置了newpassword值；

步骤1.如果当前root用户authentication_string字段下有内容，先将其设置为空，没有就跳到步骤 2。

```mysql
`use mysql; ``update` `user` `set` `authentication_string=``''` `where` `user``=``'root'`
```

步骤2.使用ALTER修改root用户密码,方法为：

```mysql
`use mysql；``ALTER user ``'root'``@``'localhost'` `IDENTIFIED BY ``'新密码'``;``FLUSH PRIVILEGES;`
```

-----

