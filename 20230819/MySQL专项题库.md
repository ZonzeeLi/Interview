# MySQL 专项题库

## 架构

### 说一说执行一条查询 SQL 语句的全过程

首先是通过连接器，建立与 MySQL 的连接，如果语句是 Select 查询语句，会查询缓存，命中直接返回，其他情况，解析器会对 SQL 语句进行解析，如果语法不对则报错，语法正确则交由执行器真正开始执行 SQL 语句，执行器中有三个阶段，分别是预处理阶段、优化阶段、执行阶段，预处理阶段就是看表是否存在和`*`号改为查所有列，优化阶段就是看使用哪个索引，执行阶段就是去存储引擎中查询到记录然后返回。

1. 首先是通过连接器，建立与MySQL的连接；
2. 然后会通过查询缓存查找数据，如果命中直接返回。但是查询缓存在MySQL 8.0被删除了，因为如果对表进行了更新操作，那么查询缓存就会失效，实际场景中，命中率并不高；
3. 然后通过解析器对 SQL 语句进行词法分析和语法分析，构建语法树，如果语法不对则报错；
4. 然后 MySQL 的优化器会基于查询的成本，选择查询成本最小的执行计划；
5. 最后通过执行器来执行计划，去存储引擎中查询到记录返回。

## 存储引擎

### MySQL 存储引擎有哪些？各自有什么区别？

主要有 MyISAM、InnoDB两个比较热门讨论的存储引擎，InnoDB是现在默认的存储引擎。

常见的有 MyISAM、InnoDB、Memory。

MySQL 的默认存储引擎是 InnoDB，支持事务和行级锁，具有提交、回滚、崩溃恢复的功能。

MyISAM 是不支持事务和行级锁的，只有表锁，但是适合只读的场景。

Memory 是将数据存储在内存中，数据的读写比较快，但是数据不具备持久性，适合临时存储的场景。

### MyISAM 和 InnoDB 存储引擎有什么区别？

主要区别有：

- 数据存放的文件规则和格式不同；
- MyISAM不支持事务而InnoDB支持；
- MyISAM的索引只有非聚簇索引；
- MyISAM不支持行锁只有表锁性能不如InnoDB；
- MyISAM只有binlog，所以如果没写入binlog的操作会数据丢失，InnoDB有redolog，可以在事务提交后出现故障，进行数据恢复，因为redologbuffer是语句执行前开始写入的，然后在事务提交后刷盘。
- InnoDB 引擎数据存储采用的是索引组织表，在索引组织表中，数据即索引，索引及数据，数据和索引都存放在一个文件中。MyISAM 引擎数据存储采用的是堆表，数据和索引分开存储，数据和索引存放在不同文件中。
- InnoDB 支持事务和行级锁，MyISAM 不支持。
- InnoDB 的 B+ 树叶子节点存放索引和数据，而 MyISAM 存放索引和数据地址。

### 用 count(*) 哪个存储引擎会更快？

用 MyISAM 更快，因为 MyISAM 会事先对行记录统计记录。

### NULL 值是如何存储的？

每一条行记录都会记录一个NULL值列表，NULL值列表是用二进制来存储NULL值的，1表示是NULL，0表示NOT NULL，并且是倒序排列，好处是从数据的头开始，向右是真实数据，向左读NULL值就是和读真实数据一样的顺序，如果列不存在NULL，那NULL值列表也就不存在。

### char 和 varchar 有什么区别？

char是定长，varchar是变长。

- char是固定长度，占用固定的空间，为定义时指定的固定长度， 如果实际存储的长度小于指定的长度，则会补充空格。
- varchar是可变长度，占用的实际字符串长度的空间，额外会用1-2字节存储可变长字符串的长度。

在设计数据表时，大多数选择varchar，因为char会填充空格浪费空间，而且从磁盘读会读写更多的数据，查找时删除空格会耗费 CPU 性能，从而导致性能下降。

### 假如说一个字段是varchar(10)，但它其实只有6个字节，那他在内存中占的存储空间是多少？在文件中占的存储空间是多少？

在内存中占用的空间是6字节，在文件中占用的空间，由于要记录变长字符的长度、NULL值列表、记录头信息，所以一般是占用8个字节。

Varchar 在保存到文件中，占用的空间是实际的字符串大小，但是在内存中会根据指定的空间长度来分配。所以，在内存中占用的是 10 字节，在文件中占用 6 字节，并且会有 1-2 字节存储可变长字符串的长度。

## 索引结构

### MySQL 有哪些索引类型？

B树索引、哈希索引、fulltext索引

补充：常用的是 B+ 树索引，是 InnoDB 引擎默认的索引类型，支持排序、分组、范围查询、模糊查询等。

### InnodB 引擎的索引数据结构是什么？

B+树

### 为什么索引用 B+ 树？而不用红黑树？

红黑树的实现复杂，且在数据量大的时候层级过多，而 B+ 树的层级稳定维持在3～4层左右，这样就会导致在查询的时候 I/O 次数过多，而红黑树的自平衡能力会导致在表结构发生变化时有较大的改变。

由于红黑树的本质是二叉树，B+ 树是多叉树，存储相同数据量时，红黑树的会比 B+ 树高，树越高，磁盘 I/O 次数越多，而且红黑树的自平衡能力会导致在表结构发生变化时有较大的改变。

### 为什么索引用 B+ 树？而不用 B 树？

B+的非叶子节点只存放索引，而B树会存放索引和数据，所以B+树会存放更多的索引，层级更少，查询更快。B+树的叶子节点是用双向链表连接，可以很快的范围查询，而B树只能回到根节点再向下查找，所以I/O次数多。

补充：层级更少，磁盘 I/O 次数越多。B+树有大量的冗余节点，插入、删除的效率会更高，不会发生复杂的树变化。

### B 树适合什么应用场景？

这个并不了解。

B 树的节点存放索引和数据，所以在进行单个索引查询时，最快可以是 O(1) 的时间复杂度，所以如果是大量的单个索引查询的场景，可以选择 B 树索引，比如 MongoDB。 

### 为什么索引用 B+ 树？而不用哈希表？

哈希虽然查询某一值时有O（1）的时间复杂度，但是没有办法做到范围查询。

补充：哈希还不支持联合索引最左匹配原则，如果重复键值比较多哦，容易造成哈希冲突。

### 聚簇索引和非聚簇索引有什么区别？

聚簇索引具有唯一性，主键可以作为聚簇索引，如果没有主键会选择一个不包含NULL的唯一列，如果也没有，会生成一个自增id作为聚簇索引。

非聚簇索引只存放聚簇索引，而聚簇索引存放的完整的记录信息。

补充：如果查询语句使用的是非聚簇索引，且查询的数据不是主键值，会在叶子节点找到主键值后进行回表，如果是主键值就会进行索引覆盖。

### insert 操作对 B+ 树结构的改变是怎么样的？

B+树会通过二分查找的方法找到插入的位置，如果可以直接插入则构建叶子节点插入，也要和前后叶子节点构建成双向链表，如果超过某一个非叶子节点的最大数量，则会让非叶子节点分类，重新构建依此向上级进行。

B+ 树中的数据是有序的，

- 如果主键是自增的，每次插入的新数据会按照顺序插入到叶子节点的最右侧，如果该页面满了，会开辟一个新的页面，因为每一次插入都是追加操作，不需要重新移动新数据。
- 如果主键是随机的，每次插入的新数据可能会随机插入，这时候会产生大量的移动，如果该页面满了，会发生页分裂，导致多出的数据需要复制到另一个页面，来保证 B+ 树的有序性。页分裂会导致大量的内存碎片，索引结果不紧凑。

### 假如一张表有两千万的数据，B+树的高度是多少？怎么算的？

这个并不了解。

利用公式，$$Total = x^{(z-1) } * y  $$，其中x表示非叶子节点中存放其他页的数量，y表示叶子节点存储的行记录行数，z表示树的层级。

这个问题具体要看数据库表的字段数量和字段类型，假设一行记录的大小是 1kb，MySQL 的数据页的大小是 16 kb，去掉一些头信息，大概可以有 15kb 的空间存储数据。

- 在非叶子节点中记录的是索引页，主要是记录主键和页号，假设主键id的类型是bigint，8字节，页号固定是4字节，所以一行数据的大小是12字节，那么一个页可以存放的记录就是 x = 15 * 1024 / 12 ≈ 1280 行。
- 在叶子节点中记录的是真实的行数据，以 1kb 来计算，那能存放的数据量 y = 15。

所以，根据公式可以得出，z为3的时候，大概可以存放2.45kw行。

## 索引应用

### MySQL 有哪些索引？

按照索引类型，可以分为B树索引、哈希索引、fulltext索引。

按照使用的存储方式，可以分为聚簇索引和非聚簇索引，聚簇索引的叶子节点存放所有的数据记录，而非聚簇索引的叶子节点只存放聚簇索引。

按照使用的列的个数，可以分为单列索引和联合索引。

按照使用的字段，可以分为主键索引、唯一索引、前缀索引和普通索引。

### 为什么要建索引？

索引好比是书的目录，可以快速查找定位信息，加快查询速度。

补充：没有建立索引，查询的时间复杂度是 O(n)，为了提高查询效率，可以建立索引，建立索引后数据按照有序存放，可以通过二分查找的方式快速查哈走啊数据局，也可以避免外部排序和使用临时表问题，可以将随机 I/O 变为 顺序 I/O。

### 我们一般选择什么样的字段来建立索引？

首先是主键字段要作为聚簇索引，用来保存完整记录信息，而其他的需要进行等值查询和排序查询的字段，比较适合建立索引，因为等值查询可以快速定位查找信息，排序的字段，索引会在构建B+树的时候以排序的方式构建，之后可以用索引下推避免回表次数过多浪费时间。

可以对频繁用于where查询的字段建立索引，提高查询速度，如果是多个列，可以考虑建立联合索引。

可以对经常用于排序和分组的字段建立索引，这样可以利用索引下推避免再次回表。

不建议对区分度不高的字段建立索引，比如性别。这种情况下无论使用哪个都会搜索出一半的值，所以在优化器处理阶段可能会选择全表扫描。

### 索引越多越好吗？

并不是，一个索引就要构建一个B+树，所以过多的索引会导致磁盘文件过多，而且一条语句中如果涉及到多个索引，在修改时会对每一个索引的B+树进行结构修改，会导致I/O次数过多。

### 索引怎么优化？

1. 使用覆盖索引，可以利用将需要查询的多个值建立成联合索引，避免再次向上回表。
2. 如果字段过长，可以使用前缀索引来优化长度。
3. 防止索引失效，语句的一些不正确使用会导致索引失效。
4. 不要创建太多的索引，只为常用的字段构建索引，防止浪费过多的存储空间。
5. 主键索引要设置为自增，自增会让一条新纪录在插入时更加方便，否则会让B+树有重新构建的可能。
6. 如果是范围查询或者排序的字段，可以建立索引使用索引下推来优化I/O次数

### 建立了索引，查询的时候一定会用到索引吗？

不一定，存在索引失效的可能，索引失效的情况有：

1. 联合索引不满足最左匹配原则，比如（A，B，C）构建的联合索引，要按照A、B、C的优先级顺序使用索引。
2. 索引如果使用了函数计算则会失效。
3. 索引如果使用了表达式计算则会失效。
4. 索引如果使用了类型转换则会失效。
5. 索引如果遇到>，<，between这种范围查询会停止。
6. 使用where语句的or关键字，如果or的左右两边不全是索引列的话会失效。
7. 如果使用了模糊查询的左模糊匹配，会导致索引失效，因为不满足排序性。

补充：8. MySQL的优化器是基于成本考虑的，利用二级索引的查询会有回表的操作，优化器会计算成本，成本过大会放弃使用索引。

### 什么是最左匹配原则？

联合索引的最左优先级最高，使用索引的时候会优先从左侧开始匹配，比如（A，B，C），要优先使用A进行匹配，然后再用B，再用C，如果不是按照这个顺序则索引失效。

补充：如果遇到某列包含范围查询，则后面的字段无法用到联合索引。

### 联合索引 （a,b,c），下面的查询语句会不会走索引？如果走具体是哪些字段能走？

1. select * from T where a=1 and b=2 and c=3;

会走索引，a、b、c均可以走索引。

1. select * from T where a=1 and b>2 and c=3;

会走索引，只使用a、b索引，因为遇到 b>2 范围查询停止。

1. select * from T where c=1 and a=2 and b=3;

会走索引，由于优化器会将and的顺序调整，所以依然可以满足最左匹配原则，a、b、c均可以走索引

1. select * from T where a=2 and c=3;

会走索引，只使用a索引，因为b没有进行匹配就用了c，不满足优先级。

1. select * from T where b=2 and c=3;

不会走索引，不满足最左匹配。

1. select (a,b) from T where a=1 and b>2;

会走索引，只使用a、b索引，使用索引覆盖，不会回表。

补充：2. select * from T where a=1 and b>2 and c=3;

会走索引，只使用a、b索引，因为遇到 b>2 范围查询停止，c会做索引下推，由于联合索引中包含c的值，不要再次回表直接在索引处定位 c = 3 的记录。

### where a>1 and b = 2 and c <3怎么建立索引？

可以建立（b，a）或者（b，c）索引，保证使用其中两个。

可以创建（b，a，c）或者（b，c，a）索引，保证使用其中两个，另外一个可以利用索引下推减少回表次数。

## 事务

### MySQL 事务有什么特性？

事务的特性可以概括成 ACID，分别是：

原子性：一个事务中的操作要么全部完成，要么全部不完成。

隔离性：多个事务同时对相同数据进行操作，互不干扰。

一致性：数据库中的数据规则符合现实世界中的规则，比如A给B转账了100，那么A的余额就要少100，B的余额就要增加100，两者的钱总量不变。

持久型：事务执行之后，数据永久保存，不会随着故障丢失。

### 事务的隔离性如何保证？

通过加锁和MVCC保证。

### 事务的持久性如何保证？

MySQL有redo log，在事务执行操作前会写入redo log buffer，事务提交后刷盘，即使系统崩溃，也可以通过redo log重写之前的操作恢复。

MySQL 有redo log，通过 WAL（先写日志再写数据）机制，执行操作前会先将操作记录redo log，然后 buffer pool 的脏页会通过后台线程刷盘，即使在还没有刷盘的时候发生了崩溃或者重启，由于执行操作都已经记录到了redo log 中，重启就可以通过redo log恢复脏页数据，从而保证了事务的持久性。

### 事务的原子性如何保证？

MySQL有redo log可以保证事务的全部完成，有undo log可以回滚失败的事务保证事务的全部不完成。

### MySQL 事务隔离级别有哪些？分别解决哪些问题？

读未提交：会发生发生脏读、幻读、不可重复读问题。 读提交：不会发生脏读，会发生幻读、不可重复读。 可重复读：不会发生脏读和不可重复读，会发生幻读。 串行化：不会发生这些问题。

### 脏读幻读有什么区别？

脏读一个事务读到了另一个事务未提交的修改内容，幻读是一个事务读到了N行，另一个事务新增或删除后，原事务再次读并不是 N 行，两次行数不一致。

### MySQL默认的隔离级别是什么？怎么实现的？

可重复读。

MySQL在事务启动后会为事务生成一个 Read View 快照，Read View 会记录当前事务的id、活跃的事务id列表、活跃事务id的最小值、下一个创建的事务id值，MySQL的行数据也会记录最新修改过这一行的事务id，同样会记录该行的上一版本记录的指针，像一个链表一样。当事务A启动后，事务B启动，事务B的活跃事务id列表就是A和B的事务id，现在事务要对一个行数据进行操作，如果活跃事务id列表的最小值比当前行数据记录的最新修改过这一行的事务id值大，说明最新操作该行数据记录的事务已经提交完成，所以可以对该行进行操作，如果大于下一个创建的事务id值，说明这个最新操作该行数据记录的事务是在当前事务A和B启动之后再启动的事务，那么就不可以对这行数据进行操作。之后要分两种情况讨论，如果在活跃的事务id列表之间，说明有其他事务在操作该行，那么不可以对该行操作，会去找该行记录的上一版本指针，如果不在则说明最新操作该行数据记录的事务已经提交，那么可以对该行进行操作，操作了之后，该行记录会更新最新修改这一样的事务id，同样以链表形式将上一版本记录连接起来。

补充：通过MVCC实现的。

### 介绍一下 MVCC

每一个行记录都会存储上一个版本记录的指针，每一个事务要对该行操作需要判断一下最新修改过该行的事务id，是否在其Read View中，如果最终判断不可对该行才做，会去找该行的上一版本记录继续判断。

 补充：将上面详细的过程描述一遍。

### MVCC的如何判断行记录对某一个事务是否可见

每一个行记录都会存储上一个版本记录的指针，每一个事务要对该行操作需要判断一下最新修改过该行的事务id，每个事务的Read View 中记录了当前事务id、活跃的事务id列表、活跃的事务id最小值、下一个要创建的事务id，如果一个事务对一个行记录要进行操作，会通过活跃的事务id最小值和最新改过该行的事务id进行比较，如果大于，说明最新修改过该行的事务已经提交完成，那么对该事务可见，如果小于，需要进一步判断，如果在活跃的事务id列表中，说明最新修改过该行的事务是活跃状态，就不可见，如果不在则说明最新修改过该行的事务已经提交，那么可见，而如果最新修改过该行的事务id大于了下一个要创建的事务id，说明修改该行的事务是在这之后创建的，不可见。

### 读已提交和可重复读隔离级别实现 MVCC 的区别？

读提交的 Read View 是在每次读的时候会创建一个新的 Read View，这样会导致，如果有事务提交，更新了某一行数据，那么新的Read View中的活跃事务id列表会更新，所以会出现幻读、不可重复读的问题。

区别在于创建 Read View的时机不同。

- 读提交的 Read View 是在每次读的时候会创建一个新的 Read View，这样会导致，如果有事务提交，更新了某一行数据，那么新的Read View中的活跃事务id列表会更新，所以会出现幻读、不可重复读的问题。
- 可重复读的 Read View 是在事务开启后，创建的一个 Read View，每次执行操作都是复用这个 Read View，所以不会有不可重复读的问题，很大程度避免了幻读。

### 为什么互联网公司用读已提交隔离级别？

因为读提交可以保证数据的实时准确，假设A和B同时对数据a进行操作，假设一个事务提交了，另一个事务还在执行，另一个事务再次查询数据a可以保证和A的一致性。

读提交因为没有间隙锁，只有记录锁，死锁的概率很低，所以适用于并发场景。

### 可重复读解决了什么问题？有没有完全解决幻读？

可重复读通过MVCC解决了脏读和不可重复读的问题。并没有完全解决幻读。

### 可重复度为什么不能完全避免幻读？什么情况下出现幻读？

因为如果一个事务先读了一组记录，另一个事务提交了一个新的记录，然后原事务可以对新记录进行操作，这时候新记录的最新修改该记录的事务id就变更成原事务id，再次读，由于新记录的最新修改该记录的事务id是原事务id，对自己可见，就可以读取到，这就发生了幻读。

## 锁

### 详细说一下 MySQL数据库中锁的分类

全局锁：加了全局锁，数据库除了读，其他操作会阻塞。比如备份数据的时候，就要加全局锁。

表级锁： 具体分为表锁、元数据锁、意向锁、autoinc锁。

表锁是对整张表加了锁，除了读操作都会阻塞。

元数据锁是为了防止在对表数据进行操作时，其他语句对表的结构进行更改。如果加了元数据读锁，那么要对表结构进行更改就会加元数据写锁，需要等待读锁的释放。 同样读锁要等待写锁的释放。

意向锁是在对表加共享锁和独占锁之前会加意向锁，可以快速的判断表中是否有锁。

autoinc锁是在主键自增的时候加的表锁，防止冲突。

行级锁：具体分为记录锁、间隙锁、next-key锁。

记录锁是对一条记录加锁，分为共享和独占两种。

间隙锁是对一个范围去见的记录加锁，分为共享和独占两种。

next-key锁是记录锁+间隙锁，

### 在线上修改表结构，会发生什么？

修改表结构会施加元数据写锁，如果修改表结构的操作之前有事务在执行读会加元数据读锁，这样修改表结构的操作会阻塞，等待锁释放，并且后面的所有操作都会阻塞，后面的操作会形成一个队列，且写锁的优先级高于读锁，所以如果修改表结构之前的事务如果是大事务，会造成整个业务系统阻塞。

### Innodb 存储引擎中的行级锁有哪些？

行级锁有记录锁、间隙锁、next-key锁。

补充：插入意向锁。

### 一条Update语句没有带where条件，加的是什么锁？

由于没有where条件定位，会走全表扫描，会对每一条行记录加next-key锁。

- 可重复读隔离级别，会走全表扫描，会对每一条行记录加next-key锁。
- 读提交隔离级别，没有间隙锁，会走全表扫描，会对每一条行记录加记录锁。

### 带了where条件没有命中索引，加的是什么锁？

没有命中索引依然走的是全表扫描。会对where条件判断范围内的记录加next-key锁。

- 可重复读隔离级别，会走全表扫描，会对每一条行记录加next-key锁。
- 读提交隔离级别，没有间隙锁，会走全表扫描，会对每一条行记录加记录锁。

### 两条更新语句更新同一条记录，加的是什么锁？

加的是 X 型记录锁。

分情况讨论，在可重复读隔离级别下：

- 如果是唯一索引，
  - 如果记录存在，加的是记录锁。
  - 如果记录不存在，加的是间隙锁。
- 如果是非唯一索引，
  - 如果存在，会对等值的记录加next-key锁，会对扫描到的第一个不符合条件的值加间隙锁。同时，在等值记录的主键索引上加记录锁。
  - 如果不存在，会对第一个不符合条件的值加间隙锁。
- 如果没有索引或者没有命中索引，会走全表扫描，会对每一条行记录加next-key锁。

### 两条更新语句更新同一条记录的不同字段，加的是什么锁？

加的是 S 型记录锁，因为不会发生竞争。

分情况讨论，在可重复读隔离级别下：

- 如果是唯一索引，
  - 如果记录存在，加的是记录锁。
  - 如果记录不存在，加的是间隙锁。
- 如果是非唯一索引，
  - 如果存在，会对等值的记录加next-key锁，会对扫描到的第一个不符合条件的值加间隙锁。同时，在等值记录的主键索引上加记录锁。
  - 如果不存在，会对第一个不符合条件的值加间隙锁。
- 如果没有索引或者没有命中索引，会走全表扫描，会对每一条行记录加next-key锁。

### MySQL 怎么实现乐观锁？

通过 CAS 来实现。乐观锁并不会加锁，每次在更新前通过 CAS 进行比较，如果和之前的值不同则回滚重试，

在表中新增一个版本号字段，利用版本号来实现乐观锁。

每次更新数据都带上版本号，然后将版本号+1，每次更新语句前先要查看版本号和表中的记录是否一致，不一致则不更新，重新获取最新的版本号，然后再尝试更新。

### 了解过 MySQL 死锁问题吗？

MySQL的死锁是主要是插入意向锁和间隙锁冲突的情况，加锁的顺序问题，也就是两个事务在互相等待对方释放锁，导致了死锁。

### MySQL 怎么排查死锁问题？

利用`show engine innodb status;`可以查看事务在使用锁的情况。

补充：然后分析死锁日志，死锁日志分为两部分，上半部分说明了事务1在等待什么锁，下半部分说明了事务2当前持有的锁和等待的锁。

通过死锁日志，可以知道两个事务形成了怎样的循环等待，根据各个事务执行的 SQL 分析加锁的类型和顺序，推断出死锁的产生。

### MySQL 怎么避免死锁？

1. 使用索引去查找记录，避免全表扫描，导致大范围的锁。
2. 不使用大事务，尽量使用小事务减少阻塞等待时间，避免后续操作等待导致死锁。
3. 优化执行顺序，避免占有锁的语句长时间不释放导致死锁。
4. 使用超时控制，如果事务等待时间过长则放弃等待，避免长时间的占用和等待，导致其他事务无法释放锁和等待。

## 日志

### MySQL 有哪些日志？有什么区别？

MySQL 有redo log重做日志、undo log回滚日志、bin log 二进制日志

redo log是在事务的语句执行前写入redo log buffer中，然后事务提交进行刷盘，如果提交后系统崩溃，导致数据没有写入，会通过redo log保证数据的正常写入。

undo log是在事务提交之前，会把语句执行前的状态记录到undo log中，保证如果事务提交前崩溃，可以进行回滚。

bin log是 Server 层用来主从复制和备份，记录所有的数据修改和表结构，每当事务提交，会将修改语句写入到bin log中。

补充：redo log 是循环写入，bin log 是追加写入。

### redo log 和 binlog 的区别和应用场景？

区别：bin log是server层的日志，是记录所有的数据修改日志和表结构，是追加写，而redo log是循环写记录，记录会被覆盖，保留未刷盘的操作。

应用场景：redo log 是用来系统崩溃恢复的，而bin log是用来主从复制和备份的。

### redo log 是怎么实现持久化的？

redo log 会在事务执行过程中，语句执行前写入redo log buffer中，然后会有刷盘机制，比如写满了、定时、事务提交后这些，写入到redo log后刷盘，就保证了持久化。

事务执行过程中，并不是在事务提交的时候把数据刷盘，而是修改buffer pool中数据页，标记为脏页，再找合适的时间刷盘。

Redo log 保存的内容是物理日志，redo log 会在事务执行过程中，语句执行前写入redo log buffer中，然后会有刷盘机制，比如写满了、定时、事务提交后这些，写入到redo log后刷盘，就保证了持久化。

### 为什么事务提交需要两阶段提交？

如果其中一个写入成功，另一个因崩溃而写入失败会导致主从不一致的问题。

为了保证 redo log 和bin log 的逻辑一致，防止其中一个写入成功，另一个因崩溃而写入失败会导致主从不一致的问题。

### 两阶段提交的过程是怎样的？

两阶段的提交主要是将redo log持久化到硬盘的阶段拆分成了两个阶段，prepare阶段和commit阶段。

首先在prepare阶段前也就是事务提交前崩溃的话，事务未提交回滚，不用担心不一致的问题。

在prepare阶段，redo log 会先持久化到硬盘，包含执行的事务id。然后到了commit阶段，会bin log会写入到硬盘，并将redo log的状态变成commit，同时bin log中也会有事务的id。

如果在prepare阶段崩溃，即redo log写入成功，bin log写入失败，在重启后会判断redo log中的事务 id 不存在在bin log中，会回滚。如果在commit阶段崩溃，由于bin log已经写入成功，则可以进行恢复。

## 性能优化

### 怎么找到慢 SQL?

可以使用慢查询日志，查找到执行时间超过阈值的记录，也可以用explain查看语句的完整执行信息。

### 如何优化慢 SQL?

1. 避免使用 * 来查询。会占用过多的内存和解析。
2. 使用索引来快速定位。用索引避免全表扫描，全表扫描会有过多的磁盘 I/O。
3. 避免索引失效。不正当的使用索引会让索引失效，比如左模糊匹配、联合索引优先级使用错误、语句中包含函数表达式等。
4. 将大事务拆分成小事务执行，避免阻塞情况。
5. 避免深分页，深分页会查询全部数据再丢弃不需要的，可以采用游标页的方案优化。

补充：6. 分解联表查询，针对联表查询，可以将联表查询分解成多个单表查询的语句，然后在业务层聚合，或者增加冗余字段减少联表查询。

### 深分页场景如何优化？

深分页的场景是在分页查询时，查询的页很深，会将查询到前面的数据丢弃，所以可以采用游标分页或者通过有序的字段来精确查找的位置，比如每一页的记录个数为N，那么查找第10000～10010的这10个数据，就可以找到10000所在的页然后精确查找，如果是有序的字段那么直接从10000开始查找。

补充：

- 将第几页改成下一页，先记录上一页的最后一条记录id，下次直接从该记录位置开始扫描，避免 MySQL 扫描大量不需要的行然后再抛弃掉的问题。
- 可以通过覆盖索引+子查询的方式改进。子查询语句主要查询分页数据对应的数据库唯一 id 值，因为主键索引在二级索引上就可以查到，所以子查询不需要回表。然后主查询再根据子查询返回的 id，这样就只进行了一次 I/O。

### 如果 SQL 和索引都没问题，查询还是很慢怎么办？

这个时候就要考虑更改表结构以及整体架构的设计。

首先针对表结构的优化，

1. 主键要选择自增，这样在插入时不会过多的有 B+树重构问题。
2. 表中的字段不要过多，太多列会导致表文件过大，所以在内存中的数据页少，会有过多的磁盘I/O。
3. 字段值的类型长度做限制，同样可以减少表文件大小。
4. 尽量不使用NULL，因为NULL同样会增加一个字段来记录NULL值，会增加记录大小，同样会使索引失效。
5. 对索引进行优化，将常用的字段设置为索引，且多用联合索引，可以利用索引覆盖和索引下推来避免回表次数，过长的字段可以使用前缀索引。
6. 针对不同的场景可以考虑其他数据库引擎和索引类型的选择。

针对整体架构的设计，

1. 如果是读比较多的场景，考虑使用缓存来加快查询，比如Redis，将查询语句结果缓存。
2. 如果是写比较多的场景，考虑使用消息中间件来削峰填谷。
3. 考虑读写分离，主机只写，从机只读，减轻服务器的压力，合理分配，达到负载均衡。
4. 如果数据量特别大的表，考虑分库分表，分析业务，将业务拆分开，比如一个商城系统的表拆分成订单表、商品表、用户信息表等。

## 高可用

### MySQL 主从复制的过程是怎么样？

MySQL主机的bin log刷盘，从机建立一个 I/O 线程监控主机的log dump，将bin log写入到从机的relay log中，然后会有另一个线程来读取relay log。

### MySQL 主从复制的数据延迟怎么处理？

主从复制的延迟主要原因是主机这边更新了之后，在从机处查询不到记录。

解决的方案有：

1. 如果是实时性很强的业务，可以考虑读主机的数据库。
2. 主机在执行完写操作后，可以先写入缓存。
3. 采用分库设计，减轻主机的压力。
4. 将大事务拆分成小事务，避免主机这边执行一个大事务时间很久，从而从机也需要很长时间来复制。
5. 采用并行复制。

### MySQL 提供了几种复制模式？默认的复制模式是什么？

全同步复制：主机要等待所有的从机响应才认为此次复制成功。 异步复制：主机并不关心从机的响应。 半同步复制：主机要等待一部分从机的响应。

默认是异步复制。

### MySQL 主从架构中，读写分离怎么实现？

读写分离是将读的操作分离到从机上，写的操作由主机完成，主机上的更改通过主从复制来同步到从机上，从机用负载均衡策略来决定具体落到哪一个从机上，以此来减轻主机的压力。

补充：可以独立部署代理中间件 MyCat 来实现读写分离。

### MySQL 主库挂了怎么办？

主库挂了会有从机接力成为主机，并保持一致性。

这时候也要考虑容灾的方案，容灾分为异地容灾和同城容灾，同城容灾的速度很快，使用同步复制，异地的采用异步复制。当同城的一台主机挂掉，切换到另一台同城的主机，如果是自然灾害一类的，可以考虑切换到异地，这样会丢失一部分数据，之后再做数据迁移。

MySQL 主从复制没有实现发现主服务器宕机和处理故障迁移的功能，要实现自动主从故障迁移，可以使用高可用套件 MHA，MHA 可以在主数据库发生故障后，选择新的主机。

### 什么是分库分表？什么时候需要分表？什么时候需要分库？

分库分表是将一个数据库按照业务需求分成不同种类的库，将表也经过拆分，可以通过水平产分，将单张表单的数据量减小，也可以根据场景进行垂直拆分，将表单更适合业务化扩展。

当一个业务连接数据库过多，就可以将数据库进行分库，比如商城，可以拆分成用户信息库、商品详情库、订单信息库等，这样在建立连接的时候，每个数据库和服务器的压力都会减小。

当表单的数据量过于庞大，比如超过500万行就可以考虑水平拆分，类似于分页一样将表拆分成多个小表，如果表单的字段有几个字段的关联度很高，或者根据实际需求可以再抽象画，就可以垂直分表，将部分字段作为一张单独的表，根据业务进行扩展。

### 分库分表后，会产生什么问题？怎么解决？

分库分表后会出现分布式事务、跨库查询、数据迁移的问题。

跨库查询会带来性能问题，因为数据要在多个库之间传输。可以考虑中间件的方式，或者单独使用一张表将关联的字段进行维护，当作索引使用。

由于服务不停机，想要做到线上数据迁移的话，可以考虑双写的方案，先通过双写读老，将新内容写到两个库，同时将旧数据同步到新库中，然后双写双读，采用灰度方案，稳定后开始双写读新，最后写新读新。

分布式事务问题发生的原因在于，一个操作可能涉及到多个库的操作，这里就存在分布式事务的一致性问题。同样除了使用中间件的方法，可以使用两阶段提交、三阶段提交、TCC提交的方法解决。

分布式事务的两阶段提交很像MySQL的redo log 的两阶段提交，协调者会向参与者发送事务的执行指令，但是不提交，询问是否可以执行事务，等参与者的响应之后，再发送提交命令。

三阶段提交是两阶段提交的优化，三阶段提交多了一步预提交阶段，向所有参与者发送指令，如果接收到参与者的预提交成功响应，则进行提交阶段。

TCC方案是Try-Confirm-Cancel方案，try阶段会为所有业务预留资源，confirm阶段在确认可以执行之后就提交，如果不可以，则进入cancel阶段进行回滚。

补充：分库分表后还会出现分布式 ID 的唯一性问题、跨库跨表的排序问题。

分布式ID问题是可能会产生主键冲突，可以使用雪花算法或者美团的leaf算法生成唯一ID来解决。

跨库跨表的排序问题是在分库分表后，数据被分散，想要对数据列表进行排序就会很复杂，可以通过中间件分别查询然后汇总排序，也可以将数据全量存储到es中进行查询。
