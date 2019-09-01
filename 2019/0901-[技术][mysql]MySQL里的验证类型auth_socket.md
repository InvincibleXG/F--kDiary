# 萌新把在win中开发的老旧SSH项目部署在linux上踩坑实记
----
> 作为一个经常做 Windows端J2EE 的菜鸟程序猿来说，极少的 linux部署经验 实在令人汗颜！

是这样的，有个任务：把一个 05年的中大型J2EE项目 用 `SVN check out` 下来后，要部署到一个服务器上调试。然后经过本地(win10)调试后，已经将这个 SSH 项目的启动相关的代码摸透了，就等着服务器去部署了，结果技术总监给了个 linux服务器 。。。没事，正好让我练练手。但是环境搞好以后，发现项目启动报错了 *Access Denied*，这让我百思不得其解，因为我安装完 MySQL 以后还特地把密码改了一遍！

当然缺乏经验且想当然的我认为大概率是 MySQL 的版本问题，因为我本机是 MySQL8，而项目本身是 MySQL5.7 被我改到了 8，可是 linux 上的 MySQL 是用 `apt-get install` 来安装的，没有指定版本，也是 5.7。更换版本相关的配置和依赖后，无效，我才开始正视这个 `访问被拒绝的权限问题`。

首先，我用 mysql -uroot -p 再输入密码登录，`select * from mysql.user;`对输出的数据表进行检查，root 确实是在 localhost 上，但是对应的 plugin 列显示的是 "auth_socket"，而其他行都是什么 "native password"，虽然不知道啥意思，但是我觉得这里面有蹊跷。退出来后，我故意输错密码，竟然还能成功进入 mysql命令行——原来如此，我设置的密码根本就无效！localhost 主机不需要密码就可以登录，所以导致 Hibernate 配置了密码后，无法正常访问数据库。

> 安装5.7并且没有为root用户提供密码，它将使用auth_socket插件。该插件不关心，也不需要密码。它只检查用户是否使用UNIX套接字进行连接，然后比较用户名。
> 如果我们要配置密码，我们需要在同一命令中同时更改插件并设置密码。首先更改插件然后设置密码将不起作用，它将再次回退到auth_socket。所以，运行：`ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '密码';`

于是我的解决方法是，删除这行记录，用 `grant语句` 重新对 任意主机的root用户 设定了密码，权限问题搞定。
```sql
grant all on *.* to 'root'@'%' identified by '密码';
flush privileges;
```

然后还有数据表大小写区分的问题，需要到 MySQL 的配置文件[mysqld]中加入让 表名全部小写的配置项 `lower_case_table_names=1`。
