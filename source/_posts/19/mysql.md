#### mysql
show variables like 'binlog_format'  查看binlog 格式
show variables like 'log_bin'  binlog 是否开启
show binary logs 获取binlog列表
show master status  当前正在写入的binlog文件
show binlog events in 'ON.000056'   查看指定binlog文件的内容
1. Statement  保存ddl语句
2. Row    保存改动后记录
3. Mixed

##### 全量备份数据
```sql
mysqldump -uroot -p -B -F -R -x --master-data=2 test > d:/backup.sql
```
-B   指定数据库   -F  刷新日志（刷新创建新的binlog日志，用来记录备份之后数据库增删改flush logs ）
-R 备份存储过程  -x 锁表

##### 从binlog日志恢复数据
恢复命令的语法格式：
```sql
mysqlbinlog mysql-bin.0000xx | mysql -u用户名 -p密码 数据库名
```
常用参数选项解释：
--start-position=875 起始pos点
--stop-position=954 结束pos点
--start-datetime="2016-9-25 22:01:08" 起始时间点
--stop-datetime="2019-9-25 22:09:46" 结束时间点
--database=xxx 指定只恢复xxx数据库(一台主机上往往有多个数据库，只限本地log日志)

##### explain 字段	含义
id	执行编号，值越大越先执行
select_type	显示查询的类型，比如简单查询还是子查询等
table	访问引用哪个表（引用某个查询，如“derived1”）
type	数据访问/读取操作类型（ALL、index、range、ref、eq_ref、const/system、NULL）
possible_keys	列出哪一些索引可能有利于高效的查找
key	显示mysql在本次查询中决定使用的索引
key_len	显示mysql在索引里使用的字节数
ref	显示了之前的表在key列记录的索引中查找值所用的列或常量
rows	为了找到所需的行而需要读取的行数，估算值，不精确。
Extra	额外信息，如using index、filesort等


##### 导致索引失效的查询：
1)查询的数量是大表的大部分，大约30％以上。 
2)索引本身失效
3)查询条件使用函数在索引列上，或者对索引列进行运算，运算包括(+，-，*，/，! 等) 错误的例子：select * from test where id-1=9; 正确的例子：select * from test where id=10; 
4)对小表查询 
5)隐式转换导致索引失效. 错误的例子：select * from test where tu_mdn=13333333333; 正确的例子：select * from test where tu_mdn='13333333333'; 
6)like "%_" 百分号在前. 
7) 向右匹配直到遇到范围查询(>、<、between、like)，后面的索引就会停止匹配
8)单独引用复合索引里非第一位置的索引列. 
9)not in ,not exist. 
10)B-tree索引 is null不会走,is not null会走 
11)联合索引 is not null 只要在建立的索引列（不分先后）都会走, in null时，必须要和建立索引第一列一起使用,当建立索引第一位置条件是is null 时，其他建立索引的列可以是is null（但必须在所有列，都满足is null的时候），或者=一个值； 当建立索引的第一位置是=一个值时,其他索引列可以是任何情况（包括is null =一个值）,以上两种情况索引都会走。其他情况不会走。

##### order by 字段值相同时， 排序可能会随机， 建议再按id 排下序

