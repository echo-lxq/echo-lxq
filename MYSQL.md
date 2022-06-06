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


## MYSQL储存用户密码 ##

1.**直接明文保存**，比如用户设置的密码是“123456”，直接将“123456”保存在数据库中，这种是最简单的保存方式，也是最不安全的方式。但实际上不少互联网公司，都可能采取的是这种方式。

2.**使用对称加密算法来保存**，比如3DES、AES等算法，使用这种方式加密是可以通过解密来还原出原始密码的，当然前提条件是需要获取到密钥。不过既然大量的用户信息已经泄露了，密钥很可能也会泄露，当然可以将一般数据和密钥分开存储、分开管理，但要完全保护好密钥也是一件非常复杂的事情，所以这种方式并不是很好的方式。

3.**使用MD5、SHA1等单向HASH算法保护密码**，使用这些算法后，无法通过计算还原出原始密码，而且实现比较简单，因此很多互联网公司都采用这种方式保存用户密码，曾经这种方式也是比较安全的方式，但随着彩虹表技术的兴起，可以建立彩虹表进行查表破解，目前这种方式已经很不安全了。

**PS.彩虹表技术**

彩虹表的出现，针对性的解决了 R 函数导致的链重复问题：它在各步的运算中，并不使用统一的 R 函数，而是分别使用 R1…Rk 一共 k 个不同的 R 函数。这样生成的哈希链集即被称为彩虹表。

4.**特殊的单向HASH算法，**由于单向HASH算法在保护密码方面不再安全，于是有些公司在单向HASH算法基础上进行了加盐、多次HASH等扩展，这些方式可以在一定程度上增加破解难度，对于加了“固定盐”的HASH算法，需要保护“盐”不能泄露，这就会遇到“保护对称密钥”一样的问题，一旦“盐”泄露，根据“盐”重新建立彩虹表可以进行破解，对于多次HASH，也只是增加了破解的时间，并没有本质上的提升。

5.**PBKDF2算法，**该算法原理大致相当于在HASH算法基础上增加随机盐，并进行多次HASH运算，随机盐使得彩虹表的建表难度大幅增加，而多次HASH也使得建表和破解的难度都大幅增加。

使用PBKDF2算法时，HASH算法一般选用sha1或者sha256，随机盐的长度一般不能少于8字节，HASH次数至少也要1000次，这样安全性才足够高。一次密码验证过程进行1000次HASH运算，对服务器来说可能只需要1ms，但对于破解者来说计算成本增加了1000倍，而至少8字节随机盐，更是把建表难度提升了N个数量级，使得大批量的破解密码几乎不可行，该算法也是美国国家标准与技术研究院推荐使用的算法。

**随机盐**+名文密码 -> 多次hash -> 密文密码

**函数定义**

    DK = PBKDF2(PRF, Password, Salt, c, dkLen)
其中：

- DK是PBKDF2算法产生的密钥

- PRF是一个伪随机函数，例如HASH_HMAC函数，它会输出长度为hLen的结果
- Password 是用来生成密钥的原文密码
- Salt 是一系列用于生成密钥加密的盐值
- c是迭代运算的次数
- dkLen 是期望得到的密钥的长度

## mysql,mysqli和PDO的区别 ##
PHP-MySQL 是 PHP 操作 MySQL 资料库最原始的 Extension 

PHP-MySQLi 的 i 代表 Improvement ，提更了相对进阶的功能，就 Extension 而言，本身也增加了安全性。

PDO (PHP Data Object) 则是提供了一个 Abstraction Layer 来操作数据库。只需要使用**PDO接口**中的方法就可以对数据库进行操作

PDO配置 
PHP.ini中,去掉"extension=php_pdo.dll"前面的";"号,若要连接数据库，还需要去掉与PDO相关的数据库扩展前面的";"号，然后重启Apache服务器即可。
 
    extension=php_pdo.dll 
    extension=php_pdo_mysql.dll 
    extension=php_pdo_pgsql.dll 
    extension=php_pdo_sqlite.dll 
    extension=php_pdo_mssql.dll 
    extension=php_pdo_odbc.dll 
    extension=php_pdo_firebird.dll 

# mysql优化 #

**MySQL优化方向：**

在设计上：字段类型，存储引擎，范式

在功能上：索引，缓存，分库分表

在架构上：集群，主从复制，负载均衡，读写分离

## MySQL索引 ##
**1.索引是什么？**

- 索引是帮助MySQL高效获取数据的**数据结构**。
 
- 索引往往存储在**磁盘**上的文件中
 
- 索引中包括:**聚集索引**，**覆盖索引**，**组合索引**，**前缀索引**，**唯一索引** 等，默认都是使用**B+树结构组织索引**

**ps**.主键索引跟唯一索引的区别

1. 主键是一种约束，唯一索引是一种索引；
1. 主键创建后一定包含一个唯一性索引，唯一性索引不一定是主键；
1. 唯一性索引列允许空值， 主键不允许；
1. 主键可被其他表引为外键，唯一索引不能；
1. 一个表只能创建一个主键，但可创建多个唯一索引。
1. 主键更适合那些不容易改变的唯一标识，如自动递增列，身份证号等。
1. 在**RBO 模式**下，主键的执行计划优先级高于唯一索引。两者可以提高查询的速度。--oracle



**2.索引的优劣势？**

	优势： 
		检索:可以提高数据检索的效率，降低数据库的IO成本
		排序:通过索引列对数据进行排序，降低了CPU的消耗
	劣势: 
		占磁盘空间
		降低更新表的效率

**3.索引分类**

1. 单列索引
	- 普通索引：没有任何限制。add index
	- 唯一索引:索引列中的值必须唯一，允许空值。add unique index
	- 主键索引:特殊的唯一索引，不允许空值。PK
1. 组合索引
	- 在表中的对个字段组合上创建的索引 add index(col1, col2……)
	- 遵循最左前缀原则(最左匹配原则)
1. 全文索引(MyISAM,InnoDB5.6以后)
	- 只能在CHAR,VARCHAR,TEXT类型字段上使用全文索引。fulltext
	- 优先级最高，先执行
	- 存储索引，决定执行一个索引
1. 空间索引

**PS.**在 MySQL 中, 索引是在存储引擎层实现的, 所以并没有统一的索引标准, 由于 InnoDB 存储引擎在 MySQL数据库中使用最为广泛, 下面以 InnoDB 为例来分析一下其中的索引模型.在 InnoDB 中, 表都是根据主键顺序以索引的形式存放的, InnoDB 使用了 B+ 树索引模型，所以数据都是存储在 B+ 树中的，如下图所示

![](./images/B+tree.png)

从图中可以看出, 根据叶子节点内容不同,索引类型分为**主键索引**和**非主键索引**.
**主键索引也被称为聚簇索引**,叶子节点存放的是整行数据; 而**非主键索引** 被称为二级索引,**叶子节点存放的是主键的值**.
如果根据主键查询, 只需要搜索ID这颗B+树
而如果通过非主键索引查询, 需要先搜索k索引树, 找到对应的主键, 然后再到ID索引树搜索一次, 这个过程叫做回表.
**总结, 非主键索引的查询需要多扫描一颗索引树, 效率相对更低.**

**关于B+树**
数据结构...

**4.索引使用**

1.索引相关语句

主键索引不需要创建，系统会自动生成

①.单列索引之普通索引

    create index index_name on table(coloumn(length));
	alter table table_name add index index_name(column(length));

②.单列索引之唯一索引

	create unique index index_name on table(column(length));
	alter table table_name add unique index index_name(column);

③.单列索引之全文索引

	create fulltext index index_name on table(column(length));
	alter table table_name add fulltext index_name(column);

④.组合索引

	alter table table_name add index index_name(time(50),title(50)...);

⑤.删除索引

	drop index index_name on table;

⑥.查看索引
	
	show index from table_name;

**5.索引原理分析**

5.1 索引的存储结构

- 索引在存储引擎中实现(不同的引擎会只用不同的索引)
- MyISAM和InnoDB存储引擎:只支持B+tree索引
- MEMORY/HEAP存储引擎:支持HSAH和BTREE索引

	`MyISAM采用的是非聚簇索引，InnoDB采用的是聚簇索引`

①.B树

- B树的高度一般在2-4，树的高度直接影响IO读写的次数
- 三层树结构----支撑的数据可以达到20G,如果是四层树结构----支撑的数据可以达到几十T

②.B树和B+树的区别

- B树和B+树最大区别在于非叶子节点是否存储数据的问题。

    `由于B树的其他子节点也存储有数据data，所以在每页中占用了相当一部分内存，而B+树只有主键索引，没有数据data域，每页((4kB,8KB,16KB)存储的主键索引相对来说是比较多的。`


5.2 聚集(簇)索引(InnoDb)

- 主键索引(聚集索引)的叶子结点会存储数据行，也就是说数据和索引在一起
- 辅助索引只会存储主键值

5.3 非聚集(簇)索引(MyISAM)

- B+树叶子结点只会存储数据行（数据文件）的指针，简单来说就是数据和索引不在一起
- 非聚集索引包含 主键索引 和 辅助索引 到会存储指针的值

5.4 主键索引Primary key

- InnoDB要求表必须有主键(MyISAM可以没有)，如果没有，MySQL系统会自动选择一个唯一标识数据记录的列作为主键
- MyISAM的索引文件(mdi）仅仅保存数据记录的地址
- MyISAM的数据文件(ibd)中记录对应的记录

5.5 辅助索引Secondary key(次要索引)

- 结构和主键搜索引没有任何区别
- 同样用B+Tree,data域存储相应记录主键的值而不是地址

聚集索引通过主键搜索十分高效，但是辅助索引搜索需要检索两边索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

	TODO:https://blog.csdn.net/qq_44129924/article/details/115333658

## SQL优化 ##

**1.插入优化**

- 大量数据采用批量插入形式

- 事务设置手动提交，MySQL默认是自动提交，意味着每写一个SQL事务就自动提交，可能会频繁的涉及事务开始和提交，所以建议手动提交

**2.order dy优化**

- Using filesort：通过表的索引或者全表扫描，读取到满足条件的数据行，然后在排序缓冲区 sort buffer 中完成排序，所以返回的数据不是通过索引直接返回的，这样的排序形式就叫filesort

- Using index：通过有序索引顺序扫描直接返回有序数据，这种情况不需要额外的排序，所以效率比较高

- 根据多字段排序时，遵循左前缀原则

**3.group by优化**

- group by进行分组

- 在分组操作时，可以通过索引来提高效率，索引使用也要满足左前缀原则

