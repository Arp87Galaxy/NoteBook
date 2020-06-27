# mysql面试

## 1事务

### 1.1ACID

原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）

### 1.2隔离级别 （默认不可重复读）

| 事务隔离级别                 | 脏读 | 不可重复读 | 幻读 |
| ---------------------------- | ---- | ---------- | ---- |
| 读未提交（read-uncommitted） | 是   | 是         | 是   |
| 不可重复读（read-committed） | 否   | 是         | 是   |
| 可重复读（repeatable-read）  | 否   | 否         | 是   |
| 串行化（serializable）       | 否   | 否         | 否   |

脏读：A事务还未提交，B个事务就读取到该数据。如果A事务回滚，B就读到的是脏数据

不可重复读（A事务执行期间，其他事务进行修改）： A事务多次执行同一语句查询，B事务在期间进行了**修改**，并提交事务，A事务就会读到不同数据。

幻读（A事务执行期间，其他事务进行插入删除）：A事务修改所有数据，还没有提交，B事务**插入**，B事务提交。A提交就会发现没有全部修改，幻读。



## force index()和ignore index

## 覆盖索引与回表、联合索引、索引下推区别

1.回表查询 ：普通索引会先遍历找到主键值（innodb主键值，MyISAM的普通索引存储的是记录指针）。再遍历聚集索引，再找到行数据块

![img](https://upload-images.jianshu.io/upload_images/4459024-a75e767d0198a6a4?imageMogr2/auto-orient/strip|imageView2/2/w/421/format/webp)

2.覆盖索引（不用回表）： 查询的字段就在索引里：

​						如 有一个 name 普通索引  

​						sql为   select id,name from user where name='shenjian';

​						因为id，name 都在name索引里能找到，所以不用回表就能找到

​						反之   select id,name**,sex** from user where name='shenjian';  sex不在name索引里

3、索引下推：首先有模糊查询时减少回表次数MySQL5.6版本后（没用到索引下推则  ：先查到数据返回结果，再从结果筛选）

   设置语句 SET optimizer_switch = 'index_condition_pushdown=off';

like 'hello%’and age >10 检索，MySQL5.6版本之前，会对匹配的数据进行回表查询。5.6版本后，会先过滤掉age<10的数据，再进行回表查询，减少回表率，提升检索速度

## 索引失效

在解释索引命中规则的前提下, 先了解一下如下原则:

最左匹配原则:

1. 最左前缀匹配原则, mysql会一只向右匹配直到遇到范围查询(>, <, between, like)就停止匹配, 比如a=1 and b=2 and c>3 and d=4 如果建立了(a,b,c,d)顺序的索引, d是用不到索引的, 如果建立(a,b,d,c)的索引, 则都可以使用到, a,b,d的顺序可以任意调整.
2. = 和 in 可以乱序, 比如 a=1 and b=2 and c=3 建立(a,b,c)索引可以任意顺序, mysql 的查询优化器会帮你优化成索引可以识别的形式.

### 1.联合索引

使用情况:
对于查询语句"select e.* from e where e.e1=1 and e.e3=2"涉及到两列, 这个时候我们一般采用一个联合索引(e1, e3); 而不用两个单列索引, 这是因为一条查询语句往往应为mysql优化器的关系只用了一个索引, 就算你有两个索引, 他也只会用到一个(`Mysql的最左匹配原则`) ; 在只用到一个的基础上, 联合索引是会比单列索引要快的;

命中规则:
示例: create table e(e1 int, e2 varchat(9), e3 int, primary key(e1, e3));
这样就建立了一个联合索引: e1, e3

- 使用联合索引的全部索引键, 可触发索引的使用.
  如: select e.* from e where e.e1=1 and e.e3=2
- 查询条件中包含索引的前缀部分, 也就是 e1, 可以触发索引的使用
  如: select e.* from e where e.e1=1
- 使用部分索引键, 但不包含索引的前缀部分，不可触发索引的使用。
  如: select e.* from e where e.e3=1
- 使用联合索引的全部索引键, 但不是AND操作, 不可以触发索引的使用
  如: select e.* from e where e.e3=2 or e.e1=1

### 2.普通索引

就是最基本的索引, 查他就能命中

### 3. 唯一索引

和普通索引类似, 不同的就是索引的列必须是唯一存在的, 可以为空

### 4. 全文索引

只支持老版本的MySql 也就是引擎为MyISAM的数据表.

## 聚集索引 普通索引

1.聚集索引：叶子节点存的是整行数据，直接通过这个聚集索引的键值找到某行     比如主键索引

​		（1）如果表定义了PK，则PK就是聚集索引；

​		（2）如果表没有定义PK，则第一个not NULL unique列是聚集索引；

​		（3）否则，InnoDB会创建一个隐藏的row-id作为聚集索引；

2.普通索引：叶子结点指向聚集索引

## 最左前缀 匹配到范围查询条件为止，后面的不会用到索引

1.最左前缀: 索引（a,b,c） select * from table where a = '1' and b = ‘2’ and c='3'   abc索引

​												select * from table where c='3' 无索引

​												select * from table where a = '1' and c='3'  a索引无c索引

​												select * from table where a = '1' and b = ‘2’ and c='3'

（会用到索引，但是会扫描整个索引树）如果有范围查询则后面不会用到索引，所以范围查询最好用到最后面

总结 index（a,b,c）只会走（a）（a,b） (a,b,c) 三中索引

​		三个字段会出现如下where情况		(a)(a,b)(a,b,c)(a,c)    (b)(bc)（c）；但是根据最左前缀倒数第四种情况只会走a索引 后三种不走



## 1.limit

limit 偏移量大的时候会出现效率下降

limit查询过程 先根据二级索引树找到主键，再根据主键树找到数据块，再根据offset的值丢弃前面的数据块。

解决方案：

​		用where 索引字段 > offset限制索引操作的回表数据。

## 2. 索引优化

#### explain

---------------------------------------寻找查询表语句-------------------------------------------------

id  ：执行顺序 ，先大后小，相同先上，

select_type: 查询类型，主查询，子查询，（子查询）衍生表，联合查询（union后面的select），union result ，						最后执行

table：表名

-----------------------------------------------查询条件索引情况----------------------------------------

type ： 表连接类型

1. system : const 的一个特列，只有一行数据
2. const（唯一值连接一个常量）:  条件列最多有一行匹配（主键索引和unique（唯一索引）），条件值在查询开始就被读取（常量值）。
3. eq_ref（唯一值连接一组数据，该组数据是唯一的。）:  索引条件列最多有一行匹配（主键索引和unique（唯一索引）），条件值是一组数据，但是该列也是唯一的，一找到就停止不用再继续扫描。（条件值与条件列一一对应）
4. ref（条件值与条件列一一对应）：1.索引条件列不唯一，一旦找到，还要再小范围查找直到不同值，因为索引是排序的；2.条件列使用了部分列。
5. range：索引条件列  +范围操作符+ 常量
6. index ：先索引树找到匹配列，再去找数据，
7. all：没用到索引，直接全表扫描

possible_keys   :可能用到的索引

key：实际用到的索引（可见只能用一个索引）

key_len ：索引里面的哪几个列

ref ： type 是左边的条件，ref是右边的条件，[const，数据库名.表名.索引列]

rows :查找了多少行找到数据。

extra：额外信息（主要显示select后面的列，从哪获得的值）

1. using where ：从where语句后面条件获取的数据，没有用索引
2. using index ：从索引获取值，没有回表，order by 时表示索引排序，没有文件排序
3. using  filesort：得到的结果使用到了文件排序，需要优化，order by 命中索引
4. Using index condition ：命中索引，需要回表。
5. Using temporary：需要建立中间表来暂存数据，group by，和order by 同时存在 ，作用于不同的字段，需要优化



**索引会被查询和排序用到**

范围查询索引会失效

多值索引中：最左前缀法则（带头大哥不能少，中间楼梯不能断）

不要在索引列上做任何操作（计算，函数，（自动or手动）类型转换），会导致索引失效而转向全表扫描type=all（索引列上少操作）

范围条件右边的列不会使用到索引，范围列的索引列也会用到，但是type=range （不能检索条件，而是范围查询）

尽量覆盖索引查询，减少select *（就算\*表示了所有列，在extar中也会少个Using Index（值从索引拿，Using where 值从where条件查询后的列拿）。

`name` is null ,is not null 也不能使用索引

like查询 %写在字符串右边，如果%非要写左边，覆盖索引列select不会失效

`少用`or， 会导致索引失效



子查询时，小表驱动大表

1. 主查询表小于子查询表时用exists（exists 子查询返回true或false也可以是1也可以时常量字符串）
2. 主查询表大于子查询表时用in

### order by

排序：order by 尽量使用index排序减少使用filesort排序

​			order by 满足最左前缀规则 using  index

​			排序不一样（升序，降序） 不能使用索引，列不是索引一部分，不能使用索引

​			对于 order by来说，in（。。。）也是范围，索引失效

​			where   + order by  满足最左 using index  using filesort

​			少写select *    当查询的字段大小小于**max_length_for_sort_data** 时而且字段不是Text和BLOB时，会采用优化的单路排序，否则多路排序，

​					两种算法的数据都可能超过**sort_buffer**，会产生tmp文件，多次I/O，单路风险会大一些，

​			**提高上面两个参数都会提高两种算法的效率**

### group by  先排序，后分组，能where 就不要having





### 开启慢查询日志

show  variables like '%slow_query_log%';

set global slow_query_log=1;//重启失效



SHOW VARIABLES LIKE 'long_query_time%'; 查询多少秒是慢查询

set global long_query_time =5;



show global status  like '%Slow_queries%'; 看看有几条慢查询，健康检查





### 锁

show open tables；查看被锁表

show status like ’table%‘

​		table_lock_immediate   立即获取锁的次数；

​		table_lock_waited;  有操作等待，就加1；表示锁冲突的严重性；

#### MyISAM（偏向读）支持表级锁

当一个锁释放，等待的队列中会优先给写操作，长时间的读也会，导致其他线程等待，因此要尽量使用中间表，分担长时间运行的查询

1. 表锁 ，表锁期间，当前session 不能操作其他表，直到unlock。

   读锁（共享锁）： 当前session    lock table `表名`  read； 只能读不能写（报错）；其他session能读，但写会阻塞等待解锁（不报错）

   写锁（独占锁）：当前session    lock table `表名`  write；能写能读。其他session读写都会阻塞

#### InnoDB（，偏向写）支持表锁，行锁，支持事务默认行锁



**行锁（对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁；对于普通SELECT语句，InnoDB不会加任何锁）**：

1.  同一行提交之前，自己的修改自己能读，其他session不能读到还没提交的修改，只能快照读。并且清除快照也要commit。

2. 修改操作索引失效,没有索引会导致 行锁升级为表锁，原因是：没有索引造成了全表扫描。

3. 显示的加锁：

   共享锁：select * from tableName where … + lock in share more

   排他锁：select * from tableName where … + for update

**间隙锁（GAP**）： 当范围修改时，innodb会锁住范围内的索引键值，即使并不存在，当你插入一个在范围内但不存在的数据行时，会被锁住，等待锁释放再执行。