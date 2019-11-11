# Py连MySQL踩坑记录

简化起因 `ImportError: No module named MySQLdb` 这个邪恶的错误！ 系统是 OSX Mojave，自带 `Python 2.7.10` 和 `Ruby 2.3.7` 但是万恶地不自带 `brew` 包管理器。好了，噩梦开始了～

缺少 `MySQLdb` 但是 `pip` 找不到同名依赖包，网上学习后得知

>对于不同的系统和程序有如下的解决方法：
>easy_install mysql-python (mix os)
>pip install mysql-python (mix os)
>apt-get install python-mysqldb (Linux Ubuntu)
>cd/usr/ports/databases/py-MySQLdb && make install clean (FreeBSD)
>yum install MySQL-python (linux Fedora, CentOS)
>pip install mysqlclient (Windows)

这些方法，无论怎么走，都会必然遇到一个更加骚气的东西 `brew` 这是一个包管理工具，但是可笑的是他竟然需要用 `ruby` 去编译。。。等等，咱们不是在搞 `Python` 吗？？？

没关系，继续干这个 `brew` ，我们会发现这玩意儿虽然脚本在 github 上，但是速度非常非常慢，是那种完全下不了的存在！一直卡住阻塞的原因，其实是我们的 mac 上没有安装 `command line tools` ， 到 [这边](https://developer.apple.com/download/more/) 完成开发者认证后即可下载（龟速预警)

之后终于可以正常下载安装 `brew` 了吗？ 当然不是，因为源是境外的，所以你懂的。。。

解决方案自然就是换源，我们可以发现这是用 `ruby` 解释/编译的，因此源必定在代码中。具体可以参考一位热心网友提出的 [解决方案](https://my.oschina.net/Rayn/blog/2876725) ，先将脚本代码拉下来，修改后直接运行 `ruby code_file` 即可。(亲测这样brew可以安装，而brew/core是brew启动时Tap安装的。

因此安装好brew再去HomeBrew/Library/里替换core的源？「尚未确定」)



----



其实，有一个绕过这些的方法～

安装pymysql：

> pip install pymysql

使用 PyMySQL 辅助进行数据库连接，将连接字符串改为 `mysql+pymysql://username:password@server/db`

竟然就特么可以了……