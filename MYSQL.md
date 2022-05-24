# MYSQL线上服务器部署 #

## 一.基本配置 ##
1. 更换默认3306线上开放端口更换为不常见的**端口号**。
2. 更换默认**用户名**
3. 更换默认**密码**
4. 开启**binlog日志**文件，方便数据恢复跟数据变更查询

## 二.SSL认证 ##
线上版本5.7以上

**PS.**虽然SSL方式使得安全性提高了，但是相对地使得QPS也降低23%左右

**1.查看是否开启了ssl**

    show global variables like '%ssl%';
    //“have_ssl” 为YES的时候,数据库是开启加密连接方式的。

**2.查看数据库基本信息**

    select version(); //查看数据库版本
    show variables like 'port'; //查看数据库端口
    show variables like 'datadir'; //查看数据库存放路径

**3.配置证书**

通过openssl制作生成证书

①.生成一个CA私钥

    openssl genrsa 2048 > ca-key.pem

②.通过CA私钥生成数字证书

    openssl req -new -x509 -nodes -days 99999 -key ca-key.pem -out ca.pem
	
③.创建mysql服务器私钥和请求证书

    openssl req -newkey rsa:2048 -days 99999 -nodes -keyout server-key.pem -out server-req.pem

④.将私钥转换为RSA私钥文件格式

    openssl rsa -in server-key.pem -out server-key.pem

⑤.用CA证书生成一个服务器的数字证书

    openssl x509 -req -in server-req.pem -days 99999 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem

⑥.创建客户端的RSA私钥和数字证书

    openssl req -newkey rsa:2048 -days 99999 -nodes -keyout client-key.pem -out client-req.pem

**PS**：Common Name字段需要填写应用服务器的ip或域名，也就是指连接服务器的ip（新服务器ip）

⑦.将生成的私钥转换为RAS私钥文件格式

    openssl rsa -in client-key.pem -out client-key.pem

⑧.用CA证书来生成一个客户端的数字证书

    openssl x509 -req -in client-req.pem -days 99999 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out client-cert.pem

**PS**：将生成后的客户端证书拷贝到应用服务器上，client-*

**4.数据库配置ssl证书**

①.将CA证书和服务端ssl文件至mysql数据目录

    cp ca.pem server-*.pem /www/server/data
    # /www/server/data是数据库的路径

②.修改msql数据库目录的CA证书和服务端ssl文件所属用户和组

    chown -v mysql.mysql  /www/server/data{ca,server*}.pem

③.修改mysql配置文件，添加ssl调用配置

    vi /etc/my.cnf
    在mysqld下添加
    [mysqld]
    ssl-ca=/www/server/data/ca.pem
    ssl-cert=/www/server/data/server-cert.pem
    ssl-key=/www/server/data/server-key.pem

④.重启mysql服务，检查数据库ssl是否开启状态,have_openssl 与 have_ssl 值都为YES表示ssl开启成功

    service mysqld restart
    show variables like 'have%ssl%';

⑤.测试ssl可用性

    grant all on *.* to 'test'@'127.0.0.1' identified by 'test' require SSL;
**PS** 需要将127.0.0.1更换为应用服务器的ip

⑥.密码连接测试

    mysql -utest -ptest -h 127.0.0.1 
    此时会报错：
    ERROR 1045 (28000): Access denied for user 'test1'@'124.222.67.220' (using password: YES)
    YES代表需要密码，但没有通过ssl验证

**PS** 如果MySQL端口不是3306，需要在后面加参数(-P 端口号)

⑦.通过客户端密钥与证书ssl+密码连接测试，并查看属性

    mysql -utest -ptest -h 127.0.0.1 --ssl-cert=client-cert.pem --ssl-key=client-key.pem

    进入数据库后，\s查看属性
    加密前：
    SSL: Not in use
    加密后：
    SSL: Cipher in use is DHE-RSA-AES256-GCM-SHA384

**PS**启动时，需要在client-cert.pem和client-key.pem证书目录下启动，或者在启动时更改证书的路径

    例:mysql -utest -ptest -h 127.0.0.1 --ssl-cert=/root/client-cert.pem --ssl-key=/root/client-key.pem