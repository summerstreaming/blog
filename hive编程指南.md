# 《hive编程指南》读书笔记
简介：本书是一本hive的编程指南，旨在介绍如何使用hive的sql方法进行汇总、查询和分析存储在hadoop分布式文件系统上的大数据集合；总体上讲本书的阅读难度不高，主要是科普在client下的hive操作和hive的基本概念；  

## 第一章 基础知识  
介绍了Hadoop和MapReduce生态，给了一个经典的word count的例子；重点表达的是如果使用传统的MR来写需要较多代码实现，但是使用**hive可以提高开发效率**，开发者可以**聚焦业务功能**实现上；  

## 第二章 基础操作  
主要介绍了hive相关的环境搭建；不仅包括hive，也连带的介绍了hadoop的环境安装；      

## 第三章 数据类型和文件格式  
* 基本数据类型：  
tinyint 1b  
smallint 2b  
int 4b  
bigint 8b  
boolean true,false  
float  
double  
string  
timestamp  
binary 字节数组  
* 集合数据类型：  
struct：类似c中的struct  
map：普通的key-value集合  
array：普通的数组集合  
  
行内字段之间的分隔符是可以指定的(hive自己一般指定不可见字符):  
```sql
row format delimited
fields terminated by ','
collection terminated by '\t'
map keys terminated by ':'
lines terminated by '\n'  -- 目前行分隔符只支持回车
stored as textfile;
```
传统数据库都是**写时模式**；hive是**读时模式**，但是load data的时候会对数据的格式进行检查，比如定义的textfile格式，就不允许挂接sequencefile的文件；  
读时模式：在查询时对表模式进行验证；  
写时模式：在数据写入的时候就对表模式进行验证；  

## 第四章 hiveQL：数据定义  
本章主要讲了表结构的各种管理操作；
hive的表从表数据文件管理上区分为管理表(内表)和外表；  
外表的主要特征：方便在hive外共享数据，删除表不会影响表数据文件(内表会同时删除文件),

* 创建表：
```sql
create table if not exists management_table( -- 建的是管理表
    id bigint comment 'just id',
    name string comment 'full name for id'
) comment 'table comment'
tblproperties('A'='a','B'='b') -- 表属性，hive会自动添加2个表属性，last_modify_by和last_modify_time,最后修改人和最后修改时间
location '/user/hive/warehouse/mydb.db/management_table';


create table if not exists management_table2 like management_table; -- copy表结构但是不会copy数据
create external table if not exists management_table2 like management_table
location 'hdfs://user/mydata/ext_table/management_table2'; -- copy表结构生成一个外表，但是不会copy数据

create external table if not exists management_table( -- 建的是外表，用关键字 external 来标识
    id bigint comment 'just id',
    name string comment 'full name for id'
) comment 'table comment'
partitioned by ('dt','hr') -- 分区表的分区字段，hive会根据分区字段建文件夹，数据文件中本身不需要有这个字段
row format delimited fields terminated by '\t'
lines terminated by '\n'
stored as textfile
location 'hdfs://user/mydata/management_table';
```

* 查看表结构或者分区信息:
```sql
describe management_table; -- 查看表结构信息
describe extended management_table; -- 查看详细的表结构信息，区别在包括了表属性信息

show partitions management_table; -- 查看表的所有分区
show partitions management_table partition(dt='20180618'); -- 只查看一部分分区的信息，此处相当于查看所有dt相同的hr分区信息，也可以加上hr的分区限制，只查看一个分区的信息，用逗号分割；

```

* 查询相关:  
```sql
set hive.mapred.mode=strict; --查询hive的sql中使用如下语句可以强制必须指定分区才能进行查询；
-- 报错信息：Error in semantic analysis: No partition predicate found for table "table_name"

set hive.mapred.mode=nonstrict; -- 取消强制指定分区查询

```

* load数据相关:
```sql
load data local inpath '/usr/mydata/data_dir' -- local定义的是本地数据文件目录，从本地load到hive中，
into table table_name
partition(dt='',hr=''); -- 确定数据load到一个分区中

-- 外部分区表load数据
alter table table_name add if not exists partition(dt='',hr='')
location 'hdfs://master_server/mydata/data_dir';

```

* 修改表结构、分区：
```sql
-- 外部分区表load数据
alter table table_name 
add if not exists partition(dt='20180618',hr='09') location 'hdfs://master_server/mydata/data_dir1'
add if not exists partition(dt='20180618',hr='10') location 'hdfs://master_server/mydata/data_dir2';

-- 外部分区表修改分区数据路径
alter table table_name partition(dt='',hr='')
set location 'hdfs://master_server/mydata/data_dir';

alter table table_name drop if exists partition(dt='',hr='');

-- 修改表名
alter table management_table rename to management_table2; -- 修改表名

-- 修改列信息
alter table table_name 
change column name new_name string comment '' after id;

-- 增加列信息, 新增加的字段会在已有字段之后，分区字段之前
alter table table_name add columns(
birth int comment 'my birth year',
high int comment 'my length'
);

-- 替换新列，删除之前的所有列，替换成这些新的列，不包括分区字段
alter table table_name replace columns(
id int comment 'my id',
name string comment 'my name',
birth int comment 'my birth year',
high int comment 'my length'
);

--修改表结构，让数据分桶存储
alter table table_name 
clustered by (id) -- 可以有多个字段，逗号分割 
sorted by(id) -- 可选子句
into 8 buckets;
```

## 第五章 HiveQL:数据操作
本章主要讲了hive表的相关数据导入和导出操作，比如load数据到表中和从表中抽取数据到文件系统中；

* load数据到hive表中  
```sql
-- 通过load data来添加数据
-- hive要求表文件在同一个集群，如果有多个hdfs，跨hdfs系统的时候只能用local方式；
load data local inpath '/usr/mydata/data_dir/table_name' -- local表示数据是在本地文件系统，数据会拷贝到目标路径，如果没有local则是转移集群上的文件到目标位置
overwrite into table table_name -- overwrite表示把原来的文件删除掉，加载新的数据，如果没有overwrite则是增量添加文件，重名文件会改名
partition(dt='',hr='');

-- 通过查询来添加数据
insert overwrite table table_name
partition(dt='',hr='')
select id,name from table_ohter
where id < 1000000;

-- 一次扫描写入表的多个分区数据
from table_other
insert overwrite table table_name partition(dt='20180618',hr='11')
  select id,name where table_other.id < 1000000
insert overwrite table table_name partition(dt='20180618',hr='12')
  select id,name where table_other.id between 1000000 and 2000000
insert overwrite table table_name partition(dt='20180618',hr='13')
  select id,name where table_other.id between 2000000 and 3000000;

-- 动态分区插入(比上面的方法写起来更简洁)
insert overwrite table table_name
partition(dt,hr)
select id,name,dt,hr from table_other where id<1000000; -- select 中的dt和hr不一定需要和partition中的字段名一样，是根据select中最后的2列来确定分区字段的，而不是根据字段命名来匹配的；

-- 混合使用动态和静态分区
insert overwrite table table_name
partition(dt='20180618',hr) -- 静态分区必须出现在动态分区之前
select id,name,dt,hr from table_other where dt='20180618' and id<1000000;

-- 动态分区默认没有开启，需要设置开启;同时还有一些其他设置，不再列举
set hive.exec.dynamic.partition=true;

-- 查询语句中创建表并加载数据(只能用于管理表，不适用于外表)
create table table_name 
as select id,name from table_other where id<1000000;
```

* 导出数据  
对于需要导出的数据，可以直接copy文件来用，前提是文件格式符合需要；
```sql
-- sql导出数据到本地路径
insert overwrite local directory '/usr/mydata/data_dir'
    select id, name from table_name where dt='20180618';

-- sql导出数据到本地多个路径，和多分区导入一样
from table_name
insert overwrite directory '/usr/mydata/data_dir_14'
    select id,name where dt='20180618' and hr='13'
insert overwrite directory '/usr/mydata/data_dir_14'
    select id,name where dt='20180618' and hr='14';
```

## 第六章 HiveQL:查询
本章主要关注与hiveql和普通sql的差异，包括语法和特性，以及对性能造成的影响；  

* 集合数据查询出来的是java的json格式的数据，例如：["apple","orange","peach"];  
* map数据查询出来的是java的json格式的数据，例如：{"apple":"red", "orange":"yellow", "peach":"green"};  
* struct结构的数据和map查询出来的数据一样;  

展开各种数据集合的方法：
* 集合：array_fruit[0] : apple  
* map: map_fruit["apple"] : red     
* struct: struct_fruit.apple : red  

各种列值计算：  
* 包括算术运算和按位运算；  
* 各种运算函数，内置函数包括round、floor、ceil、rand，还有各种数学计算函数；  
* 各种聚合函数，包括sum、avg、count、min、max等；  
* 各种表生成函数，与聚合函数相反，是把一列扩展成多列或者多行；explode(field_name) as sub_field;array或者map的一列扩展成多行，

避免mapred的hiveql，也就是 -- 本地模式：
```sql
set hive.exec.mode.local.auto=true; -- 设置后hive会尝试对其他sql也使用本地模式；
select * from table_name;
select * from table_name where dt='' and hr='' limit 10; -- where只限制分区字段，是否y有limit限制都可以
```

* group by 语句  
group by 语句通常和 聚合函数一起使用，按照一个或者多个列对数据进行分组，然后再对每个组做聚合操作；  
* having 语句  
having 语句主要用来对group by 语句产生的列做条件过滤；如果不用having，也可以使用子查询完成同样功能；  
* join 语句
大多数情况下，hive会为每个join操作启动一个MapReduce task(如果多个join的on条件一样则可以自动优化节省MR task)，并且总是从左到右执行，这就要求大家尽量**用小表join大表来提升性能**，另外hive的join连接条件on中只支持等值连接(tb_a.a=tb_b.b)，多个连接条件中间只支持and，不支持or；  
join语句主要分为  
    * inner join :  
    * left outer join :  
    * right outer join :   
    * full outer join :  
    * left semi join :   
    * map side join: 需要设置 hive.auto.convert.join=true;(默认为false)；当左表很小的时候，自动的在map端执行连接操作，可以省掉reduce过程；  
* order by 和 sort by  
order by 是全局排序，sort by 是单个reduce内排序；  
* distribute by  
控制map的输出到reduce的过程中是如何划分的；通常不需要使用，但是如果使用了streaming和某些udaf的时候需要考虑这个；  
* 类型转换  
这个通常是自动向较大范围的字段类型靠齐；强转语法: cast(id as bigint);  
* 抽样查询  
对于太大的结果集，有时候只需要一个有代表性的抽样结果集，而不是完整的结果集；
```sql
-- 分母表示数据分桶的个数，分子表示选择的桶的个数
select id from table_name tablesample(bucket 3 out of 10 on rand()) s; -- rand抽样是随机的
select id from table_name tablesample(bucket 3 out of 10 on id) s; -- 指定抽样结果就是固定的

-- 基于行的按百分比抽样
select id from table_name tablesample(0.1 percent) s;
```
* union all  
用于多表的数据合并，其实也可以用于同一表的数据合并；每个子查询需要有相同的列返回，而且字段类型必须一致；

## 第七章 HiveQL：视图
视图可以允许保存一个查询，并可以向表一样对这个查询做操作；这 **是一个逻辑结构，并不会保存数据**，也就是说hive **不支持物化视图**；  
视图的作用主要是降低查询复杂度；用的不多，也不展开讲了；  

## 第八章 HiveQL: 索引  
hive支持有限的索引功能，hive中没有主键的概念，但是可以通过在一些字段上建立索引来加快一些操作；一张表的索引数据是单独存储在另外的一张表中的(**索引独立存储**)；  

## 第九章 模式设计
本章主要介绍设计hive表的过程中，哪些模式是应该使用的，哪些模式是应该避免使用的；  
* 按天划分表：这种正确的打开方式应该是做分区表，按天分区；  
* 关于分区：要适度分区，太多分区会导致大量的hadoop小文件和小文件夹；这样MR的启动初始化开销会增大；  
* 尽量保证数据有唯一键和标准化，可以充分使用array，map和struct来实现；没有唯一键时join操作应该尽量避免；  
* 同一份数据的多种处理同时处理   
```sql
-- from table 这种语句可以保证后面的执行只对table_name表扫描一次，同时做后面的多种执行
from table_name 
    insert into table_new1 overwrite select a,b,c where dt='' 
    insert into table_new2 overwrite select a,b,c where dt='' ;
```
* 对于每个表的分区；定期每天跑的中间数据尽量对每天的中间结果根据天分区，这样可以避免数据出现跨天覆盖的错误；(需要额外管理中间表分区的数据删除)  
* 如果一个表的一个分区数据量太大，第二个分区又太细，可以采用分桶来存储数据；  
```sql
create table table_name(
    id int,
    name string
)
partitioned by(dt)
clustered by (hash_code) into 32 buckets; -- 根据hash_code分为32个桶

-- 分桶的表数据需要正确填充
-- method1,设置强制bucketing，目的是让reduce个数和bucket个数一致
set hive.enforce.bucketing=true;

-- method2, 强制设置reduce个数和bucket个数一致，然后在insert语句中的 select 部分加 clustered by语句
set hive.mapred.tasks=32
insert overwrite table table_name partition(dt)
    select id,name,dt,hash_code from table_other clustered by hash_code;
```
* 为表加列：hive读取数据的时候根据schema读取的，少的列统一为null，多出的列会忽略掉；加列直接 alter table add column 加到末尾即可，不能加在原字段的中间；
* 尽量使用数据压缩；一般MapReduce是IO密集的，压缩可以减少IO，增加的是CPU使用率，整体性能无影响；





