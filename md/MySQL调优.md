# 性能监控

## 使用show profile查询剖析工具，可以指定具体的type

此工具默认是禁用的，可以通过服务器变量在绘画级别动态的修改 set profiling=1; 当设置完成之后，在服务器上执行的所有语句，都会测量其耗费的时间和其他一些查询执行状态变更相关的数据。 select \* from emp; 在mysql的命令行模式下只能显示两位小数的时间，可以使用如下命令查看具体的执行时间 show profiles; 执行如下命令可以查看详细的每个步骤的时间： show profile for query 1;

### type

#### all：显示所有性能信息

##### show profile all for query n

#### block io：显示块io操作的次数

##### show profile block io for query n

#### context switches：显示上下文切换次数，被动和主动

##### show profile context switches for query n

#### cpu：显示用户cpu时间、系统cpu时间

##### show profile cpu for query n

#### IPC：显示发送和接受的消息数量

##### show profile ipc for query n

#### Memory：暂未实现

#### page faults：显示页错误数量

##### show profile page faults for query n

#### source：显示源码中的函数名称与位置

##### show profile source for query n

#### swaps：显示swap的次数

##### show profile swaps for query n

### 直接查询Information_schema中的profiling表

set @querd_id = 1;

SELECT STATE, SUM(DURATION) AS Total_R, ROUND( 100 \* SUM(DURATION) / (SELECT SUM(DURATION) FROM INFORMATION_SCHEMA.PROFILING WHERE QUERY_ID = @query_id) , 2) AS Pct_R, COUNT(*) AS Calls, SUM(DURATION) / COUNT(*) AS ‘R/Call’ FROM INFORMATION_SCHEMA.PROFILING WHERE QUERY_ID = @query_id GROUP BY STATE ORDER BY Total_R DESC;

## 使用performance schema来更加容易的监控mysql

### MYSQL performance schema详解.md

## 使用show processlist查看连接的线程个数，来观察是否有大量线程处于不正常的状态或者其他不正常的特征

### 属性说明

#### id表示session id

#### user表示操作的用户

#### host表示操作的主机

#### db表示操作的数据库

#### command表示当前状态

##### sleep：线程正在等待客户端发送新的请求

##### query：线程正在执行查询或正在将结果发送给客户端

##### locked：在mysql的服务层，该线程正在等待表锁

##### analyzing and statistics：线程正在收集存储引擎的统计信息，并生成查询的执行计划

##### Copying to tmp table：线程正在执行查询，并且将其结果集都复制到一个临时表中

##### sorting result：线程正在对结果集进行排序

##### sending data：线程可能在多个状态之间传送数据，或者在生成结果集或者向客户端返回数据

#### info表示详细的sql语句

#### time表示相应命令执行时间

#### state表示命令执行状态

# schema与数据类型优化

## 数据类型的优化

### 更小的通常更好

应该尽量使用可以正确存储数据的最小数据类型，更小的数据类型通常更快，因为它们占用更少的磁盘、内存和CPU缓存，并且处理时需要的CPU周期更少，但是要确保没有低估需要存储的值的范围，如果无法确认哪个数据类型，就选择你认为不会超过范围的最小类型 案例： 设计两张表，设计不同的数据类型，查看表的容量

#### Test.java

### 简单就好

简单数据类型的操作通常需要更少的CPU周期，例如， 1、整型比字符操作代价更低，因为字符集和校对规则是字符比较比整型比较更复杂， 2、使用mysql自建类型而不是字符串来存储日期和时间 3、用整型存储IP地址（使用MySQL函数INET_ATON） 案例： 创建两张相同的表，改变日期的数据类型，查看SQL语句执行的速度

### 尽量避免null

如果查询中包含可为NULL的列，对mysql来说很难优化，因为可为null的列使得索引、索引统计和值比较都更加复杂，坦白来说，通常情况下null的列改为not null带来的性能提升比较小，所有没有必要将所有的表的schema进行修改，但是应该尽量避免设计成可为null的列

### 实际细则

#### 整数类型

可以使用的几种整数类型：TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT分别使用8，16，24，32，64位存储空间。 尽量使用满足需求的最小数据类型

#### 字符和字符串类型

1、char长度固定，即每条数据占用等长字节空间；最大长度是255个字符，适合用在身份证号、手机号等定长字符串 2、varchar可变程度，可以设置最大长度；最大空间是65535个字节，适合用在长度可变的属性 3、text不设置长度，当不知道属性的最大长度时，适合用text 按照查询速度：char\>varchar\>text

##### varchar根据实际内容长度保存数据

###### 1、使用最小的符合需求的长度。

###### 2、varchar(n) n小于等于255使用额外一个字节保存长度，n\>255使用额外两个字节保存长度。

###### 3、varchar(5)与varchar(255)保存同样的内容，硬盘存储空间相同，但内存空间占用不同，是指定的大小 。

###### 4、varchar在mysql5.6之前变更长度，或者从255一下变更到255以上时时，都会导致锁表。

###### 应用场景

####### 1、存储长度波动较大的数据，如：文章，有的会很短有的会很长

####### 2、字符串很少更新的场景，每次更新后都会重算并使用额外存储空间保存长度

####### 3、适合保存多字节字符，如：汉字，特殊字符等

##### char固定长度的字符串

###### 1、最大长度：255

###### 2、会自动删除末尾的空格

###### 3、检索效率、写效率 会比varchar高，以空间换时间

###### 应用场景

####### 1、存储长度波动不大的数据，如：md5摘要

####### 2、存储短字符串、经常更新的字符串

#### BLOB和TEXT类型

MySQL 把每个 BLOB 和 TEXT 值当作一个独立的对象处理。 两者都是为了存储很大数据而设计的字符串类型，分别采用二进制和字符方式存储。

#### datetime和timestamp

1、不要使用字符串类型来存储日期时间数据 2、日期时间类型通常比字符串占用的存储空间小 3、日期时间类型在进行查找过滤时可以利用日期来进行比对 4、日期时间类型还有着丰富的处理函数，可以方便的对时间类型进行日期计算 5、使用int存储日期时间不如使用timestamp类型

##### datetime

###### 占用8个字节

###### 与时区无关，数据库底层时区配置，对datetime无效

###### 可保存到毫秒

###### 可保存时间范围大

####### 1000-9999

###### 不要使用字符串存储日期类型，占用空间大，损失日期类型函数的便捷性

##### timestamp

###### 占用4个字节

###### 时间范围：1970-01-01到2038-01-19

###### 精确到秒

###### 采用整形存储

###### 依赖数据库设置的时区

###### 自动更新timestamp列的值

##### date

###### 占用的字节数比使用字符串、datetime、int存储要少，使用date类型只需要3个字节

###### 使用date类型还可以利用日期时间函数进行日期之间的计算

###### date类型用于保存1000-01-01到9999-12-31之间的日期

#### 使用枚举代替字符串类型

有时可以使用枚举类代替常用的字符串类型，mysql存储枚举类型会非常紧凑，会根据列表值的数据压缩到一个或两个字节中，mysql在内部会将每个值在列表中的位置保存为整数，并且在表的.frm文件中保存“数字-字符串”映射关系的查找表 create table enum_test(e enum(‘fish’,‘apple’,‘dog’) not null);

insert into enum_test(e) values(‘fish’),(‘dog’),(‘apple’);

select e+0 from enum_test;

alter table enum_test modify e enum(‘fish’, ‘dog’, ‘apple’) not null primary key ;

select e+0 from enum_test where e = ‘dog’;

#### 特殊类型数据

人们经常使用varchar(15)来存储ip地址，然而，它的本质是32位无符号整数不是字符串，可以使用INET_ATON()和INET_NTOA函数在这两种表示方法之间转换 案例： select inet_aton(‘1.1.1.1’) select inet_ntoa(16843009)

#### 操作系统读取磁盘时的概念，磁盘预读（4KB）和局部性原理

## 合理使用范式和反范式

### 范式

#### 优点

##### 范式化的更新通常比反范式要快

##### 当数据较好的范式化后，很少或者没有重复的数据

##### 范式化的数据比较小，可以放在内存中，操作比较快

#### 缺点

##### 通常需要进行关联

#### 键

##### 超键

###### 在关系中能唯一标识元组的属性集称为关系模式的超键

##### 候选键

###### 不含有多余属性的超键称为候选键，也称最小超键

####### 主属性

######## 包含在任何一个候选键中的属性

####### 非主属性（非键）

######## 不包含在任何一个候选键中的属性

##### 主键

###### 若候选键多于一个，选定一个候选键作为主键

#### 类型

##### 1NF

###### 列不可分割

##### 2NF

###### 表必须有主键（非主属性都必须完全函数依赖于候选键）

##### 3NF

###### 任何非主属性都直接依赖于主属性，不能传递依赖于主属性。

##### BCNF

###### 决定因素都为键

####### 所有非主属性对每一个键都是完全函数依赖（即非主属性不能对主键子集依赖）

####### 所有主属性对每一个不包含它的键也是完全函数依赖

####### 没有任何属性完全依赖于非主的任何一组属性

###### 非主属性均 既不部分依赖于候选键 也不传递依赖于候选键 且候选键均包含主键

##### 4NF

###### 删除同一表内的多对多关系

####### 函数传递的局限

######## 函数依赖事实上是单值依赖，所以不能表达属性值之间的一对多关系。

####### 多值依赖

######## 属性之间的一对多关系，记为K→→A

####### 平凡的多值依赖

######## 全集U=K+A，一个K可以对应于多个A，即K→→A。

######## 此时整个表就是一组一对多关系。

####### 非平凡的多值依赖

######## 全集U=K+A+B，一个K可以对应于多个A，也可以对应于多个B，A与B互相独立，即K→→A，K→→B。

######## 整个表有多组一对多关系

######### “一” 部分是相同的属性集合

######### “多” 部分是互相独立的属性集合

### 反范式

#### 优点

##### 所有的数据都在同一张表中，可以避免关联

##### 可以设计有效的索引；

#### 缺点

##### 表格内的冗余较多，删除数据时候会造成表有些有用的信息丢失

### 注意

#### 在企业中很好能做到严格意义上的范式或者反范式，一般需要混合使用

##### 在一个网站实例中，这个网站，允许用户发送消息，并且一些用户是付费用户。现在想查看付费用户最近的10条信息。 在user表和message表中都存储用户类型(account_type)而不用完全的反范式化。这避免了完全反范式化的插入和删除问题，因为即使没有消息的时候也绝不会丢失用户的信息。这样也不会把user_message表搞得太大，有利于高效地获取数据。

##### 另一个从父表冗余一些数据到子表的理由是排序的需要。

##### 缓存衍生值也是有用的。如果需要显示每个用户发了多少消息（类似论坛的），可以每次执行一个昂贵的自查询来计算并显示它；也可以在user表中建一个num_messages列，每当用户发新消息时更新这个值。

#### 案例

##### 范式设计

##### 反范式设计

## 主键的选择

### 代理主键

#### 与业务无关的，无意义的数字序列

### 自然主键

#### 事物属性中的自然唯一标识

### 推荐使用代理主键

#### 它们不与业务耦合，因此更容易维护

#### 一个大多数表，最好是全部表，通用的键策略能够减少需要编写的源码数量，减少系统的总体拥有成本

## 字符集的选择

字符集直接决定了数据在MySQL中的存储编码方式，由于同样的内容使用不同字符集表示所占用的空间大小会有较大的差异，所以通过使用合适的字符集，可以帮助我们尽可能减少数据量，进而减少IO操作次数。

### 1.纯拉丁字符能表示的内容，没必要选择 latin1 之外的其他字符编码，因为这会节省大量的存储空间。

### 2.如果我们可以确定不需要存放多种语言，就没必要非得使用UTF8或者其他UNICODE字符类型，这回造成大量的存储空间浪费。

#### 改用utf8mb4

##### MySQL在5.5.3之后增加了这个utf8mb4的编码，是utf8的超集

###### mb4就是most bytes 4的意思，专门用来兼容四字节的unicode

##### 原因

###### 原来mysql支持的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了。

### 3.MySQL的数据类型可以精确到字段，所以当我们需要大型数据库中存放多字节数据的时候，可以通过对不同表不同字段使用不同的数据类型来较大程度减小数据存储量，进而降低 IO 操作次数并提高缓存命中率。

## 存储引擎的选择

### 存储引擎的对比

#### InnoDB支持表级锁和行级锁，可以给索引加锁时，使用行级锁，否则使用表级锁。

#### 存储引擎指，存储文件的组织形式

## 适当的数据冗余

### 1.被频繁引用且只能通过 Join 2张(或者更多)大表的方式才能得到的独立小字段。

### 2.这样的场景由于每次Join仅仅只是为了取得某个小字段的值，Join到的记录又大，会造成大量不必要的 IO，完全可以通过空间换取时间的方式来优化。不过，冗余的同时需要确保数据的一致性不会遭到破坏，确保更新的同时冗余字段也被更新。

## 适当拆分

### 当我们的表中存在类似于 TEXT 或者是很大的 VARCHAR类型的大字段的时候，如果我们大部分访问这张表的时候都不需要这个字段，我们就该义无反顾的将其拆分到另外的独立表中，以减少常用数据所占用的存储空间。这样做的一个明显好处就是每个数据块中可以存储的数据条数可以大大增加，既减少物理 IO 次数，也能大大提高内存中的缓存命中率。

# 执行计划

## mysql执行计划.md

# 通过索引进行优化

想要了解索引的优化方式，必须要对索引的底层原理有所了解

## 索引基本知识

### 索引的优点

#### 1、大大减少了服务器需要扫描的数据量

#### 2、帮助服务器避免排序和临时表

#### 3、将随机io变成顺序io

### 索引的用处

#### 1、快速查找匹配WHERE子句的行

#### 2、从consideration中消除行,如果可以在多个索引之间进行选择，mysql通常会使用找到最少行的索引

#### 3、如果表具有多列索引，则优化器可以使用索引的任何最左前缀来查找行

#### 4、当有表连接的时候，从其他表检索行数据

#### 5、查找特定索引列的min或max值

#### 6、如果排序或分组时在可用索引的最左前缀上完成的，则对表进行排序和分组

#### 7、在某些情况下，可以优化查询以检索值而无需查询数据行

### 索引的分类

#### 主键索引

#### 唯一索引

#### 普通索引

#### 全文索引

#### 组合索引

### 面试技术名词

#### 回表

##### 当使用非主属性作为索引时，叶子结点存储的应该是该记录的主键值。因此查询最终结果还需要使用主键值查询主键的B+树，该行为称为回表。

#### 覆盖索引

##### 如果一个索引包含的数据，可以满足正在查找的所有数据（字段与条件），且因此不需要再回表查询，就称为覆盖索引。

#### 最左匹配

#### 索引下推

##### 谓词下推

##### 低版本时需要先忽略后置字段，先通过前置字段进行查询，因此这个过程需要回表两次

##### 5.6版本添加索引下推后，可以在索引内部同时判断两个字段，只需要一次回表就可以完成查询

#### 索引合并

#### 索引页分裂 页合并

### 索引采用的数据结构

#### 哈希表

#### B+树

### 索引匹配方式

create table staffs( id int primary key auto_increment, name varchar(24) not null default ’’ comment ‘姓名’, age int not null default 0 comment ‘年龄’, pos varchar(20) not null default ’’ comment ‘职位’, add_time timestamp not null default current_timestamp comment ‘入职时间’ ) charset utf8 comment ‘员工记录表’; ———–alter table staffs add index idx_nap(name, age, pos);

#### 全值匹配

##### 全值匹配指的是和索引中的所有列进行匹配

###### explain select \* from staffs where name = ‘July’ and age = ‘23’ and pos = ‘dev’;

#### 匹配最左前缀

##### 只匹配前面的几列

###### explain select \* from staffs where name = ‘July’ and age = ‘23’;

###### explain select \* from staffs where name = ‘July’;

#### 匹配列前缀

##### 可以匹配某一列的值的开头部分

###### explain select \* from staffs where name like ‘J%’;

###### explain select \* from staffs where name like ‘%y’;

#### 匹配范围值

##### 可以查找某一个范围的数据

###### explain select \* from staffs where name \> ‘Mary’;

#### 精确匹配某一列并范围匹配另外一列

##### 可以查询第一列的全部和第二列的部分

###### explain select \* from staffs where name = ‘July’ and age \> 25;

#### 只访问索引的查询

##### 查询的时候只需要访问索引，不需要访问数据行，本质上就是覆盖索引

###### explain select name,age,pos from staffs where name = ‘July’ and age = 25 and pos = ‘dev’;

## 哈希索引

### 基于哈希表的实现，只有精确匹配索引所有列的查询才有效

### 在mysql中，只有memory的存储引擎显式支持哈希索引

### 哈希索引自身只需存储对应的hash值，所以索引的结构十分紧凑，这让哈希索引查找的速度非常快

### 哈希索引的限制

#### 1、哈希索引只包含哈希值和行指针，而不存储字段值，索引不能使用索引中的值来避免读取行

#### 2、哈希索引数据并不是按照索引值顺序存储的，所以无法进行排序

#### 3、哈希索引不支持部分列匹配查找，哈希索引是使用索引列的全部内容来计算哈希值

#### 4、哈希索引支持等值比较查询，也不支持任何范围查询

#### 5、访问哈希索引的数据非常快，除非有很多哈希冲突，当出现哈希冲突的时候，存储引擎必须遍历链表中的所有行指针，逐行进行比较，直到找到所有符合条件的行

#### 6、哈希冲突比较多的话，维护的代价也会很高

### 案例

当需要存储大量的URL，并且根据URL进行搜索查找，如果使用B+树，存储的内容就会很大 select id from url where url=“” 也可以利用将url使用CRC32做哈希，可以使用以下查询方式： select id fom url where url=“” and url_crc=CRC32(““) 此查询性能较高原因是使用体积很小的索引来完成查找

## 组合索引

### 当包含多个列作为索引，需要注意的是正确的顺序依赖于该索引的查询，同时需要考虑如何更好的满足排序和分组的需要

### 案例，建立组合索引a,b,c

#### 不同SQL语句使用索引情况

## 聚簇索引与非聚簇索引

### 聚簇索引

#### 不是单独的索引类型，而是一种数据存储方式，指的是数据行跟相邻的键值紧凑的存储在一起

##### 优点

###### 1、可以把相关数据保存在一起

###### 2、数据访问更快，因为索引和数据保存在同一个树中

###### 3、使用覆盖索引扫描的查询可以直接使用页节点中的主键值

##### 缺点

###### 1、聚簇数据最大限度地提高了IO密集型应用的性能，如果数据全部在内存，那么聚簇索引就没有什么优势

###### 2、插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式

####### 如果不是按照主键顺序加载数据，那么在加载完成后最好使用 OPTIMIZE TABLE 命令重新组织一下表。

####### 也可以先关闭索引的创建，待数据迁移完成后再打开

###### 3、更新聚簇索引列的代价很高，因为会强制InnoDB将每个被更新的行移动到新的位置

###### 4、基于聚簇索引的表在插入新行，或者主键被更新导致需要移动行的时候，可能面临页分裂的问题

####### 当行的主键值要求必须将这一行插入到某个已满的页中时，存储引擎会将该页分裂成两个页面来容纳该行，这就是一次分裂操作。

####### 页分裂会导致表占用更多的磁盘空间。

###### 5、聚簇索引可能导致全表扫描变慢，尤其是行比较稀疏，或者由于页分裂导致数据存储不连续的时候

### 非聚簇索引

#### 数据文件跟索引文件分开存放

## 覆盖索引

### 基本介绍

#### 1、如果一个索引包含所有需要查询的字段的值，我们称之为覆盖索引

#### 2、不是所有类型的索引都可以称为覆盖索引，覆盖索引必须要存储索引列的值

#### 3、不同的存储实现覆盖索引的方式不同，不是所有的引擎都支持覆盖索引，memory不支持覆盖索引

### 优势

#### 1、索引条目通常远小于数据行大小，如果只需要读取索引，那么mysql就会极大的较少数据访问量

#### 2、因为索引是按照列值顺序存储的，所以对于IO密集型的范围查询会比随机从磁盘读取每一行数据的IO要少的多

#### 3、一些存储引擎如MYISAM在内存中只缓存索引，数据则依赖于操作系统来缓存，因此要访问数据需要一次系统调用，这可能会导致严重的性能问题

#### 4、由于INNODB的聚簇索引，覆盖索引对INNODB表特别有用

### 案例演示

#### 覆盖索引.md

## 优化小细节

### 当使用索引列进行查询的时候尽量不要使用表达式，把计算放到业务层而不是数据库层

#### select actor_id from actor where actor_id=4;

#### select actor_id from actor where actor_id+1=5;

### 尽量使用主键查询，而不是其他索引，因为主键查询不会触发回表查询

### 使用前缀索引

#### 前缀索引实例说明.md

##### 基数（Cardinality）

###### 集合中不同元素的个数

##### HyperLogLog

### 使用索引扫描来排序

#### 使用索引扫描来做排序.md

##### Using index：表示直接访问索引就足够获取到所需要的数据，不需要通过索引回表；

##### Using where：表示优化器需要通过索引回表查询数据；

##### Using index condition：索引下推

### union all,in,or都能够使用索引，但是推荐使用in

#### explain select \* from actor where actor_id = 1 union all select \* from actor where actor_id = 2;

#### explain select \* from actor where actor_id in (1,2);

#### explain select \* from actor where actor_id = 1 or actor_id =2;

### 范围列可以用到索引

#### 范围条件是：\<、\<=、\>、\>=、between

#### 范围列可以用到索引，但是范围列后面的列无法用到索引，索引最多用于一个范围列

### 强制类型转换会全表扫描

create table user(id int,name varchar(10),phone varchar(11));

alter table user add index idx_1(phone);

#### explain select \* from user where phone=13800001234;

##### 不会触发索引

#### explain select \* from user where phone=’13800001234’;

##### 触发索引

### 更新十分频繁，数据区分度不高的字段上不宜建立索引

#### 更新会变更B+树，更新频繁的字段建议索引会大大降低数据库性能

#### 类似于性别这类区分不大的属性，建立索引是没有意义的，不能有效的过滤数据，

#### 一般区分度在80%以上的时候就可以建立索引，区分度可以使用 count(distinct(列名))/count(\*) 来计算

### 创建索引的列，不允许为null，可能会得到不符合预期的结果

### 当需要进行表连接的时候，最好不要超过三张表，因为需要join的字段，数据类型必须一致

#### 使用nested-loop（嵌套循环算法）

#### 大表Join大表，使用分区表解决

#### join on 的特殊情况（可以用1 = 1 作为例子查看效果）

### 能使用limit的时候尽量使用limit

### 单表索引建议控制在5个以内（空间大小）

### 单索引字段数不允许超过5个（组合索引）

### 创建索引的时候应该避免以下错误概念

#### 索引越多越好

#### 过早优化，在不了解系统的情况下进行优化

## 索引监控

### show status like ‘Handler_read%’;

### 参数解释

#### Handler_read_first：读取索引第一个条目的次数

#### Handler_read_key：通过index获取数据的次数

#### Handler_read_last：读取索引最后一个条目的次数

#### Handler_read_next：通过索引读取下一条数据的次数

#### Handler_read_prev：通过索引读取上一条数据的次数

#### Handler_read_rnd：从固定位置读取数据的次数

#### Handler_read_rnd_next：从数据节点读取下一条数据的次数

## 简单案例

### 索引优化分析案例.md

# 查询优化

在编写快速的查询之前，需要清楚一点，真正重要的是响应时间，而且要知道在整个SQL语句的执行过程中每个步骤都花费了多长时间，要知道哪些步骤是拖垮执行效率的关键步骤，想要做到这点，必须要知道查询的生命周期，然后进行优化，不同的应用场景有不同的优化方式，不要一概而论，具体情况具体分析，

## 查询慢的原因

### 网络

### CPU

### IO

### 上下文切换

### 系统调用

### 生成统计信息

### 锁等待时间

#### MyISAM（表级锁）

##### 共享读锁

##### 独占写锁

#### InnoDB（索引锁，表现为表级锁和行级锁）

##### 共享锁

##### 排它锁

#### 自增锁

##### 如果表中存在自增字段，MySQL便会自动维护一个自增锁。

#### 间隙锁

##### Innodb在可重复读提交下为了解决幻读问题时引入的锁机制

## 优化数据访问

### 查询性能低下的主要原因是访问的数据太多，某些查询不可避免的需要筛选大量的数据，我们可以通过减少访问数据量的方式进行优化

#### 确认应用程序是否在检索大量超过需要的数据

#### 确认mysql服务器层是否在分析大量超过需要的数据行

### 是否向数据库请求了不需要的数据

#### 查询不需要的记录

我们常常会误以为mysql会只返回需要的数据，实际上mysql却是先返回全部结果再进行计算，在日常的开发习惯中，经常是先用select语句查询大量的结果，然后获取前面的N行后关闭结果集。

优化方式是在查询后面添加limit

#### 多表关联时返回全部列

select \* from actor inner join film_actor using(actor_id) inner join film using(film_id) where film.title=‘Academy Dinosaur’;

select actor.\* from actor…;

#### 总是取出全部列

在公司的企业需求中，禁止使用select \*,虽然这种方式能够简化开发，但是会影响查询的性能，所以尽量不要使用

#### 重复查询相同的数据

如果需要不断的重复执行相同的查询，且每次返回完全相同的数据，因此，基于这样的应用场景，我们可以将这部分数据缓存起来，这样的话能够提高查询效率

## 执行过程的优化

### 查询缓存

在解析一个查询语句之前，如果查询缓存是打开的，那么mysql会优先检查这个查询是否命中查询缓存中的数据，如果查询恰好命中了查询缓存，那么会在返回结果之前会检查用户权限，如果权限没有问题，那么mysql会跳过所有的阶段，就直接从缓存中拿到结果并返回给客户端

### 查询优化处理

mysql查询完缓存之后会经过以下几个步骤：解析SQL、预处理、优化SQL执行计划，这几个步骤出现任何的错误，都可能会终止查询

#### 语法解析器和预处理

mysql通过关键字将SQL语句进行解析，并生成一颗解析树（AST,Abstract Syntax Tree - 抽象语法树），mysql解析器将使用mysql语法规则验证和解析查询，例如验证使用使用了错误的关键字或者顺序是否正确等等，预处理器会进一步检查解析树是否合法，例如表名和列名是否存在，是否有歧义，还会验证权限等等

#### 查询优化器

当语法树没有问题之后，相应的要由优化器将其转成执行计划，一条查询语句可以使用非常多的执行方式，最后都可以得到对应的结果，但是不同的执行方式带来的效率是不同的，优化器的最主要目的就是要选择最有效的执行计划

mysql使用的是基于成本的优化器（CBO,Cost-Based Optimizer），在优化的时候会尝试预测一个查询使用某种查询计划时候的成本，并选择其中成本最小的一个

##### select count(\*) from film_actor; show status like ‘last_query_cost’; 可以看到这条查询语句大概需要做1104个数据页才能找到对应的数据，这是经过一系列的统计信息计算来的

###### 每个表或者索引的页面个数

###### 索引基数

###### 索引和数据行的长度

###### 索引分布情况

##### 在很多情况下mysql会选择错误的执行计划，原因如下：

###### 统计信息不准确

InnoDB因为其mvcc的架构，并不能维护一个数据表的行数的精确统计信息

###### 执行计划的成本估算不等同于实际执行的成本

有时候某个执行计划虽然需要读取很多的页面，但是他的成本却很小： 因为如果这些页面都是顺序读或者这些页面都已经在内存中的话，那么它的访问成本将很小，mysql层面并不知道哪些页面在内存中，哪些在磁盘，所以执行查询过程中到底需要多少次IO是无法得知的

###### mysql的最优可能跟你想的不一样

####### mysql的优化是基于成本模型的优化，但是有可能不是最快的优化

###### mysql不考虑其他并发执行的查询

###### mysql不会考虑不受其控制的操作成本

####### 执行存储过程或者用户自定义函数的成本

##### 优化器的优化策略

###### 静态优化

####### 直接对解析树进行分析，并完成优化

###### 动态优化

####### 动态优化与查询的上下文有关，也可能跟取值、索引对应的行数有关

###### mysql对查询的静态优化只需要一次，但对动态优化在每次执行时都需要重新评估

##### 优化器的优化类型

###### 重新定义关联表的顺序

数据表的关联并不总是按照在查询中指定的顺序进行，决定关联顺序时优化器很重要的功能

###### 将外连接转化成内连接，内连接的效率要高于外连接

####### 执行内连接时，获取的数据量小于外连接

###### 使用等价变换规则，mysql可以使用一些等价变化来简化并规划表达式

###### 优化count(),min(),max()

索引和列是否可以为空，通常可以帮助mysql优化这类表达式：例如，要找到某一列的最小值，只需要查询索引的最左端的记录即可，不需要全文扫描比较

###### 预估并转化为常数表达式，当mysql检测到一个表达式可以转化为常数的时候，就会一直把该表达式作为常数进行处理

explain select film.film_id,film_actor.actor_id from film inner join film_actor using(film_id) where film.film_id = 1

###### 索引覆盖扫描，当索引中的列包含所有查询中需要使用的列的时候，可以使用覆盖索引

###### 子查询优化

mysql在某些情况下可以将子查询转换一种效率更高的形式，从而减少多个查询多次对数据进行访问，例如将经常查询的数据放入到缓存中

###### 等值传播

如果两个列的值通过等式关联，那么mysql能够把其中一个列的where条件传递到另一个上： explain select film.film_id from film inner join film_actor using(film_id) where film.film_id \> 500; 这里使用film_id字段进行等值关联，film_id这个列不仅适用于film表而且适用于film_actor表 explain select film.film_id from film inner join film_actor using(film_id) where film.film_id \> 500 and film_actor.film_id \> 500;

##### 关联查询

mysql的关联查询很重要，但其实关联查询执行的策略比较简单：mysql对任何关联都执行嵌套循环关联操作，即mysql先在一张表中循环取出单条数据，然后再嵌套到下一个表中寻找匹配的行，依次下去，直到找到所有表中匹配的行为止。然后根据各个表匹配的行，返回查询中需要的各个列。mysql会尝试再最后一个关联表中找到所有匹配的行，如果最后一个关联表无法找到更多的行之后，mysql返回到上一层次关联表，看是否能够找到更多的匹配记录，以此类推迭代执行。整体的思路如此，但是要注意实际的执行过程中有多个变种形式：

###### join的实现方式原理

####### Simple Nested-Loop Join

####### Index Nested-Loop Join

####### Block Nested-Loop Join

######## （1）Join Buffer会缓存所有参与查询的列而不是只有Join的列。 （2）可以通过调整join_buffer_size缓存大小 （3）join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G-1，而之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。 （4）使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on，默认为开启。

######## show variables like ‘%optimizer_switch%’

###### 案例演示

查看不同的顺序执行方式对查询性能的影响： explain select film.film_id,film.title,film.release_year,actor.actor_id,actor.first_name,actor.last_name from film inner join film_actor using(film_id) inner join actor using(actor_id); 查看执行的成本： show status like ‘last_query_cost’; 按照自己预想的规定顺序执行： explain select straight_join film.film_id,film.title,film.release_year,actor.actor_id,actor.first_name,actor.last_name from film inner join film_actor using(film_id) inner join actor using(actor_id); 查看执行的成本： show status like ‘last_query_cost’;

##### 排序优化

无论如何排序都是一个成本很高的操作，所以从性能的角度出发，应该尽可能避免排序或者尽可能避免对大量数据进行排序。 推荐使用利用索引进行排序，但是当不能使用索引的时候，mysql就需要自己进行排序，如果数据量小则再内存中进行，如果数据量大就需要使用磁盘，mysql中称之为filesort。 如果需要排序的数据量小于排序缓冲区(show variables like ‘%sort_buffer_size%’;),mysql使用内存进行快速排序操作，如果内存不够排序，那么mysql就会先将树分块，对每个独立的块使用快速排序进行排序，并将各个块的排序结果存放再磁盘上，然后将各个排好序的块进行合并，最后返回排序结果

###### 排序的算法

####### 两次传输排序

第一次数据读取是将需要排序的字段读取出来，然后进行排序，第二次是将排好序的结果按照需要去读取数据行。 这种方式效率比较低，原因是第二次读取数据的时候因为已经排好序，需要去读取所有记录而此时更多的是随机IO，读取数据成本会比较高

两次传输的优势，在排序的时候存储尽可能少的数据，让排序缓冲区可以尽可能多的容纳行数来进行排序操作

####### 单次传输排序

先读取查询所需要的所有列，然后再根据给定列进行排序，最后直接返回排序结果，此方式只需要一次顺序IO读取所有的数据，而无须任何的随机IO，问题在于查询的列特别多的时候，会占用大量的存储空间，无法存储大量的数据

####### 当需要排序的列的总大小超过max_length_for_sort_data定义的字节，mysql会选择双次排序，反之使用单次排序，当然，用户可以设置此参数的值来选择排序的方式

## 优化特定类型的查询

### 优化count()查询

count()是特殊的函数，有两种不同的作用，一种是某个列值的数量，也可以统计行数

#### 总有人认为myisam的count函数比较快，这是有前提条件的，只有没有任何where条件的count(\*)才是比较快的

##### 变量保存表的总行数

#### 使用近似值

在某些应用场景中，不需要完全精确的值，可以参考使用近似值来代替，比如可以使用explain来获取近似的值 其实在很多OLAP的应用中，需要计算某一个列值的基数，有一个计算近似值的算法叫hyperloglog。

#### 更复杂的优化

一般情况下，count()需要扫描大量的行才能获取精确的数据，其实很难优化，在实际操作的时候可以考虑使用索引覆盖扫描，或者增加汇总表，或者增加外部缓存系统。

### 优化关联查询

#### 确保on或者using子句中的列上有索引，在创建索引的时候就要考虑到关联的顺序

当表A和表B使用列C关联的时候，如果优化器的关联顺序是B、A，那么就不需要在B表的对应列上建上索引，没有用到的索引只会带来额外的负担，一般情况下来说，只需要在关联顺序中的第二个表的相应列上创建索引

#### 确保任何的groupby和order by中的表达式只涉及到一个表中的列，这样mysql才有可能使用索引来优化这个过程

### 优化子查询

#### 子查询的优化最重要的优化建议是尽可能使用关联查询代替

### 优化limit分页

在很多应用场景中我们需要将数据进行分页，一般会使用limit加上偏移量的方法实现，同时加上合适的orderby 的子句，如果这种方式有索引的帮助，效率通常不错，否则的化需要进行大量的文件排序操作，还有一种情况，当偏移量非常大的时候，前面的大部分数据都会被抛弃，这样的代价太高。 要优化这种查询的话，要么是在页面中限制分页的数量，要么优化大偏移量的性能

#### 优化此类查询的最简单的办法就是尽可能地使用覆盖索引，而不是查询所有的列

##### select film_id,description from film order by title limit 50,5

##### explain select film.film_id,film.description from film inner join (select film_id from film order by title limit 50,5) as lim using(film_id);

### 优化union查询

mysql总是通过创建并填充临时表的方式来执行union查询，因此很多优化策略在union查询中都没法很好的使用。经常需要手工的将where、limit、order by等子句下推到各个子查询中，以便优化器可以充分利用这些条件进行优化

#### 除非确实需要服务器消除重复的行，否则一定要使用union all，因此没有all关键字，mysql会在查询的时候给临时表加上distinct的关键字，这个操作的代价很高

#### 集合操作

注意事项： 作为运算对象的记录的列数必须相同 集合运算符会除去重复的记录 作为运算对象的记录中列的类型必须一致 可以使用任何select语句，但order by子句只能在最后使用一次

##### union（并集）

##### intersect（交集）

##### except（差集）

### 推荐使用用户自定义变量

用户自定义变量是一个容易被遗忘的mysql特性，但是如果能够用好，在某些场景下可以写出非常高效的查询语句，在查询中混合使用过程化和关系话逻辑的时候，自定义变量会非常有用。 用户自定义变量是一个用来存储内容的临时容器，在连接mysql的整个过程中都存在。

#### 自定义变量的使用

##### set @one :=1

##### set @min_actor :=(select min(actor_id) from actor)

##### set @last_week :=current_date-interval 1 week;

#### 自定义变量的限制

##### 1、无法使用查询缓存

##### 2、用户自定义变量的生命周期是在一个连接中有效，所以不能用它们来做连接间的通信

##### 3、不能在使用常量或者标识符的地方使用自定义变量，例如表名、列名或者limit子句

##### 4、使用未定义变量不会产生任何语法错误

##### 5、值类型：不能显式地声明自定义变量地类型

##### 6、运算优先级：赋值符号 := 的优先级非常低，所以在使用赋值表达式的时候应该明确的使用括号

##### 7、mysql优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按预想地方式运行

#### 自定义变量的使用案例

##### 优化排名语句

###### 1、在给一个变量赋值的同时使用这个变量

####### select actor_id,@rownum:=@rownum+1 as rownum from actor limit 10;

###### 2、查询获取演过最多电影的前10名演员，然后根据出演电影次数做一个排名

####### set @actor_number:=0;

####### select t.\*, @actor_number:=@actor_number+1 as actor_number from (select actor_id,count(\*) as cnt from film_actor group by actor_id) t order by t.cnt desc limit 10;

##### 避免重新查询刚刚更新的数据

###### 当需要高效的更新一条记录的时间戳，同时希望查询当前记录中存放的时间戳是什么

####### update t1 set lastUpdated=now() where id =1; select lastUpdated from t1 where id =1;

####### update t1 set lastupdated = now() where id = 1 and @now:=now(); select @now;

##### 确定取值的顺序

###### 在赋值和读取变量的时候可能是在查询的不同阶段

####### set @rownum:=0; select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum\<=1; 因为where和select在查询的不同阶段执行，所以看到查询到两条记录，这不符合预期

######## 先筛选，在计算值

####### set @rownum:=0; select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum\<=1 order by first_name 当引入了order by之后，发现打印出了全部结果，这是因为order by引入了文件排序，而where条件是在文件排序操作之前取值的

######## 先取所有值排序，再筛选，最后计算值

####### 解决这个问题的关键在于让变量的赋值和取值发生在执行查询的同一阶段： set @rownum:=0; select actor_id,@rownum as cnt from actor where (@rownum:=@rownum+1)\<=1;

######## 先筛选，在计算值

# 分区表

对于用户而言，分区表是一个独立的逻辑表，但是底层是由多个物理子表组成。分区表对于用户而言是一个完全封装底层实现的黑盒子，对用户而言是透明的，从文件系统中可以看到多个使用#分隔命名的表文件。 mysql在创建表时使用partition by子句定义每个分区存放的数据，在执行查询的时候，优化器会根据分区定义过滤那些没有我们需要数据的分区，这样查询就无须扫描所有分区。 分区的主要目的是将数据安好一个较粗的力度分在不同的表中，这样可以将相关的数据存放在一起。

## 分区表的应用场景

### 表非常大以至于无法全部都放在内存中，或者只在表的最后部分有热点数据，其他均是历史数据

### 分区表的数据更容易维护

#### 删除：批量删除大量数据可以使用清除整个分区的方式

#### 维护：对一个独立分区进行优化、检查、修复等操作

### 分区表的数据可以分布在不同的物理设备上，从而高效地利用多个硬件设备

### 可以使用分区表来避免某些特殊的瓶颈

#### innodb的单个索引的互斥访问

#### ext3文件系统的inode锁竞争

### 可以备份和恢复独立的分区

## 分区表的限制

### 分区数量：一个表最多只能有1024个分区，在5.7版本的时候可以支持8196个分区

#### 分区文件过多，会影响IO效率

#### Linux本身的限制

##### ulimit -a 查看 open files，指

##### ulimit -n 查看当前shell能打开的文件描述符数量

##### ulimit -u 查看用户最大可用的进程数

##### 1G内存最多可以打开10w文件

### 分区表达式：在早期的mysql中，分区表达式必须是整数或者是返回整数的表达式，在mysql5.5中，某些场景可以直接使用列来进行分区

### 索引：如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含进来

### 外键：分区表无法使用外键约束

## 分区表的原理

### 分区表的底层原理.md

## 分区表的类型

### 范围分区

#### 根据列值在给定范围内将行分配给分区

##### 范围分区.md

### 列表分区

#### 类似于按range分区，区别在于list分区是基于列值匹配一个离散值集合中的某个值来进行选择

CREATE TABLE employees ( id INT NOT NULL, fname VARCHAR(30), lname VARCHAR(30), hired DATE NOT NULL DEFAULT ‘1970-01-01’, separated DATE NOT NULL DEFAULT ‘9999-12-31’, job_code INT, store_id INT ) PARTITION BY LIST(store_id) ( PARTITION pNorth VALUES IN (3,5,6,9,17), PARTITION pEast VALUES IN (1,2,10,11,19,20), PARTITION pWest VALUES IN (4,12,13,14,18), PARTITION pCentral VALUES IN (7,8,15,16) );

### 列分区

#### mysql从5.5开始支持column分区，可以认为是range和list的升级版，在5.5之后，可以使用column分区替代range和list，但是column分区只接受普通列不接受表达式

CREATE TABLE `list_c` ( `c1` int(11) DEFAULT NULL, `c2` int(11) DEFAULT NULL ) ENGINE=InnoDB DEFAULT CHARSET=latin1 /*!50500 PARTITION BY RANGE COLUMNS(c1) (PARTITION p0 VALUES LESS THAN (5) ENGINE = InnoDB, PARTITION p1 VALUES LESS THAN (10) ENGINE = InnoDB) */

CREATE TABLE `list_c` ( `c1` int(11) DEFAULT NULL, `c2` int(11) DEFAULT NULL, `c3` char(20) DEFAULT NULL ) ENGINE=InnoDB DEFAULT CHARSET=latin1 /*!50500 PARTITION BY RANGE COLUMNS(c1,c3) (PARTITION p0 VALUES LESS THAN (5,‘aaa’) ENGINE = InnoDB, PARTITION p1 VALUES LESS THAN (10,‘bbb’) ENGINE = InnoDB) */

CREATE TABLE `list_c` ( `c1` int(11) DEFAULT NULL, `c2` int(11) DEFAULT NULL, `c3` char(20) DEFAULT NULL ) ENGINE=InnoDB DEFAULT CHARSET=latin1 /*!50500 PARTITION BY LIST COLUMNS(c3) (PARTITION p0 VALUES IN (‘aaa’) ENGINE = InnoDB, PARTITION p1 VALUES IN (‘bbb’) ENGINE = InnoDB) */

### hash分区

#### 基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含mysql中有效的、产生非负整数值的任何表达式

CREATE TABLE employees ( id INT NOT NULL, fname VARCHAR(30), lname VARCHAR(30), hired DATE NOT NULL DEFAULT ‘1970-01-01’, separated DATE NOT NULL DEFAULT ‘9999-12-31’, job_code INT, store_id INT ) PARTITION BY HASH(store_id) PARTITIONS 4;

CREATE TABLE employees ( id INT NOT NULL, fname VARCHAR(30), lname VARCHAR(30), hired DATE NOT NULL DEFAULT ‘1970-01-01’, separated DATE NOT NULL DEFAULT ‘9999-12-31’, job_code INT, store_id INT ) PARTITION BY LINEAR HASH(YEAR(hired)) PARTITIONS 4;

### key分区（密钥分区）

#### 类似于hash分区，区别在于key分区只支持一列或多列，且mysql服务器提供其自身的哈希函数，必须有一列或多列包含整数值

CREATE TABLE tk ( col1 INT NOT NULL, col2 CHAR(5), col3 DATE ) PARTITION BY LINEAR KEY (col1) PARTITIONS 3;

### 子分区

#### 在分区的基础之上，再进行分区后存储

CREATE TABLE `t_partition_by_subpart` ( `id` INT AUTO_INCREMENT, `sName` VARCHAR(10) NOT NULL, `sAge` INT(2) UNSIGNED ZEROFILL NOT NULL, `sAddr` VARCHAR(20) DEFAULT NULL, `sGrade` INT(2) NOT NULL, `sStuId` INT(8) DEFAULT NULL, `sSex` INT(1) UNSIGNED DEFAULT NULL, PRIMARY KEY (`id`, `sGrade`) ) ENGINE = INNODB PARTITION BY RANGE(id) SUBPARTITION BY HASH(sGrade) SUBPARTITIONS 2 ( PARTITION p0 VALUES LESS THAN(5), PARTITION p1 VALUES LESS THAN(10), PARTITION p2 VALUES LESS THAN(15) );

## 如何使用分区表

如果需要从非常大的表中查询出某一段时间的记录，而这张表中包含很多年的历史数据，数据是按照时间排序的，此时应该如何查询数据呢？ 因为数据量巨大，肯定不能在每次查询的时候都扫描全表。考虑到索引在空间和维护上的消耗，也不希望使用索引，即使使用索引，会发现会产生大量的碎片，还会产生大量的随机IO，但是当数据量超大的时候，索引也就无法起作用了，此时可以考虑使用分区来进行解决

### 全量扫描数据，不要任何索引

使用简单的分区方式存放表，不要任何索引，根据分区规则大致定位需要的数据为止，通过使用where条件将需要的数据限制在少数分区中，这种策略适用于以正常的方式访问大量数据

### 索引数据，并分离热点

如果数据有明显的热点，而且除了这部分数据，其他数据很少被访问到，那么可以将这部分热点数据单独放在一个分区中，让这个分区的数据能够有机会都缓存在内存中，这样查询就可以只访问一个很小的分区表，能够使用索引，也能够有效的使用缓存

## 在使用分区表的时候需要注意的问题

### null值会使分区过滤无效

### 分区列和索引列不匹配，会导致查询无法进行分区过滤

### 选择分区的成本可能很高

### 打开并锁住所有底层表的成本可能很高

### 维护分区的成本可能很高

# 服务器参数设置

## general

### datadir=/var/lib/mysql

#### 数据文件存放的目录

### socket=/var/lib/mysql/mysql.sock

#### mysql.socket表示server和client在同一台服务器，并且使用localhost进行连接，就会使用socket进行连接

### pid_file=/var/lib/mysql/mysql.pid

#### 存储mysql的pid

### port=3306

#### mysql服务的端口号

### default_storage_engine=InnoDB

#### mysql存储引擎

### skip-grant-tables

#### 当忘记mysql的用户名密码的时候，可以在mysql配置文件中配置该参数，跳过权限表验证，不需要密码即可登录mysql

## character

### character_set_client

#### 客户端数据的字符集

### character_set_connection

#### mysql处理客户端发来的信息时，会把这些数据转换成连接的字符集格式

### character_set_results

#### mysql发送给客户端的结果集所用的字符集

### character_set_database

#### 数据库默认的字符集

### character_set_server

#### mysql server的默认字符集

## connection

### max_connections

#### mysql的最大连接数，如果数据库的并发连接请求比较大，应该调高该值

### max_user_connections

#### 限制每个用户的连接个数

### back_log

#### mysql能够暂存的连接数量，当mysql的线程在一个很短时间内得到非常多的连接请求时，就会起作用，如果mysql的连接数量达到max_connections时，新的请求会被存储在堆栈中，以等待某一个连接释放资源，如果等待连接的数量超过back_log,则不再接受连接资源

### wait_timeout

#### mysql在关闭一个非交互的连接之前需要等待的时长

##### JDBC

### interactive_timeout

#### 关闭一个交互连接之前需要等待的秒数

##### 长连接

## log

### log_error

#### 指定错误日志文件名称，用于记录当mysqld启动和停止时，以及服务器在运行中发生任何严重错误时的相关信息

### log_bin

#### 指定二进制日志文件名称，用于记录对数据造成更改的所有查询语句

### binlog_do_db

#### 指定将更新记录到二进制日志的数据库，其他所有没有显式指定的数据库更新将忽略，不记录在日志中

### binlog_ignore_db

#### 指定不将更新记录到二进制日志的数据库

### MySQL日志

#### Redo Log（前滚日志）

#### Undo Log（回滚日志）

#### Bin Log（服务端的日志）

Binlog 是 server 层的日志，主要做 MySQL 功能层面的事情 与 Redo Log的区别 Redo Log 是 Innodb 独有的， Binlog 是所有引擎都可以使用的 Redo Log 是物理日志，记录的是在某个数据页上做了什么修改， Binlog 是逻辑日志，记录的是这个语句的原始逻辑 Redo Log 是循环写的（随机写），空间会用完， Binlog 是可以追加写的（顺序写），不会覆盖之前的日志信息

Binlog 中会记录所有的逻辑，并且采用追加写的方式 一般在企业中数据库会有备份系统，可以定期执行备份，备份的周期可以自己设置 恢复数据的过程： 找到最近一次的全量备份数据 从备份的时间点开始，将备份的 binlog 取出来，重放到要恢复的那个时刻

### sync_binlog

#### 指定多少次写日志后同步磁盘

### general_log

#### 是否开启查询日志记录

### general_log_file

#### 指定查询日志文件名，用于记录所有的查询语句

### slow_query_log

#### 是否开启慢查询日志记录

### slow_query_log_file

#### 指定慢查询日志文件名称，用于记录耗时比较长的查询语句

### long_query_time

#### 设置慢查询的时间，超过这个时间的查询语句才会记录日志

### log_slow_admin_statements

#### 是否将管理语句写入慢查询日志

## cache

### key_buffer_size

#### 索引缓存区的大小（只对myisam表起作用）

### query_cache

#### query_cache_size

##### 查询缓存的大小，未来版本被删除

###### show status like ‘%Qcache%’

####### 查看缓存的相关属性

###### Qcache_free_blocks

####### 缓存中相邻内存块的个数，如果值比较大，那么查询缓存中碎片比较多

###### Qcache_free_memory

####### 查询缓存中剩余的内存大小

###### Qcache_hits

####### 表示有多少次命中缓存

###### Qcache_inserts

####### 表示多少次未命中而插入

###### Qcache_lowmen_prunes

####### 多少条query因为内存不足而被移除cache

###### Qcache_queries_in_cache

####### 当前cache中缓存的query数量

###### Qcache_total_blocks

####### 当前cache中block的数量

#### query_cache_limit

##### 超出此大小的查询将不被缓存

#### query_cache_min_res_unit

##### 缓存块最小大小（缓存页）

#### query_cache_type

##### 缓存类型，决定缓存什么样的查询

###### 0

####### 禁用

###### 1

####### 将缓存所有结果，除非sql语句中使用sql_no_cache禁用查询缓存

###### 2

####### 表示只缓存select语句中通过sql_cache指定需要缓存的查询

### sort_buffer_size

#### 每个需要排序的线程分派该大小的缓冲区

##### innodb_sort_buffer_size

##### myisam_sort_buffer_size

### max_allowed_packet=32M

#### 限制server接受的数据包大小

### join_buffer_size=2M

#### 表示关联缓存的大小

### thread_cache_size

服务器线程缓存，这个值表示可以重新利用的，保存在缓存中的线程数量 当断开连接时，客户端的线程将被放到缓存中，以响应下一个客户端，而不是销毁 如果线程重新被请求，将从缓存中读取线程 如果缓存中是空的或者是新的请求，线程将被重新请求，即重新创建线程 如果有很多新的线程，增加这个值即可

#### Threads_cached

##### 代表当前此时此刻线程缓存中有多少空闲线程

#### Threads_connected

##### 代表当前已建立连接的数量

#### Threads_created

##### 代表最近一次服务启动，已创建现成的数量，如果该值比较大，那么服务器会一直再创建线程

#### Threads_running

##### 代表当前激活的线程数

## INNODB

### innodb_buffer_pool_size=

#### 该参数指定大小的内存来缓冲数据和索引，最大可以设置为物理内存的80%

### innodb_flush_log_at_trx_commit

#### 主要控制innodb将log buffer中的数据写入日志文件并flush磁盘的时间点，值分别为0，1，2

### innodb_thread_concurrency

#### 设置innodb线程的并发数，默认为0表示不受限制，如果要设置建议跟服务器的cpu核心数一致或者是cpu核心数的两倍

### innodb_log_buffer_size

#### 此参数确定日志文件所用的内存大小，以M为单位

### innodb_log_file_size

#### 此参数确定数据日志文件的大小，以M为单位

### innodb_log_files_in_group

#### 以循环方式将日志文件写到多个文件中

### read_buffer_size

#### mysql读入缓冲区大小，对表进行顺序扫描的请求将分配到一个读入缓冲区

### read_rnd_buffer_size

#### mysql随机读的缓冲区大小

### innodb_file_per_table

#### 此参数确定为每张表分配一个新的文件

# mysql集群（后续更新）

## 主从复制

### 解决服务器之间的传输延迟问题

#### 5.7之后的MTS（并行复制原理）

## 读写分离

### MySQL-proxy

### MyCat

## 分库分表
