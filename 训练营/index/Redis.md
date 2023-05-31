# Redis学习与面试

## Redis 入门

### Redis 是什么？

Redis 是一个开源的内存数据结构存储，用作数据库、缓存、消息代理和流式计算引擎。Redis 提供数据结构，如string、hash、list、set、sorted set、bitmap、hyperloglogs、geospatial index 和 stream，Redis 具有内置的复制、Lua脚本、LRU逐出、事务和不同级别的磁盘持久型，并通过 Redis 哨兵和 Redis 集群自动分区提供高可用性。

### Base 理论

可以说 Base 理论是 CAP 中一致性的妥协，和传统事务的 ACID 截然不同，Base 不追求强一致性，而是允许数据在一段时间内是不一致的，但最终达到一致状态，从而获得更高的可用性和性能。

![RedisBase理论](../picture/RedisBase理论.png)

## Redis 对象

### Redis Object

#### Object 是什么？

Redis 是 key-value 存储，key 和 value 在 Redis 中都被抽象为对象，key 只能是 String 对象，而 Value 支持丰富的对象种类，包括 String、List、Set、Hash、Sorted Set、Stream等。

#### Object在内存中是什么样子

redisObject定义如下：

```c
// from Redis 5.0.5
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time or
                            * LFU data */
    int refcount;
    void *ptr;
} robj;
```

type：是哪种Redis对象；

encoding：表示用哪种底层编码，用OBJECT ENCODING [key] 可以看到对应的编码方式；

lru：记录对象访问信息，用于内存淘汰；

refcount：引用计数，用来描述有多少个指针，指向该对象；

ptr：内容指针，指向实际内容。

### String

String就是字符串，它是Redis中最基本的数据对象，最大为512MB。可以通过`proto-max-bulk-len`来修改。

#### 应用场景

一般可以用来存字节数据、文本数据、序列化后的对象数据等。

#### 常用操作

常用操作聚焦于创建、查询、更新和删除。

- 创建：即产生一个字符串对象数据，可以用SET、SETNX。
- 查询操作：可以用GET，如果想一次获取多个，可以用MGET。
- 更新：其实也是用SET来更新的。
- 删除：针对String对象本身的销毁，用DEL命令。

##### 写操作

`SET key value` 

功能：设置一个key的值为特定的value，成功则返回OK。String对象的创建或者更新都是该命令。

`SETNX key value`

功能：用于在指定的key不存在时，为key设置指定的值，返回值0表示key存在不做操作，1表示设置成功。

`DEL key [key ...]`

功能：删除对象，返回值为删除成功了几行。

##### 读操作

`GET key`

功能：查询某个key，存在就返回对应的value，如果不存在返回nil。

`MGET key [key ...]`

功能：一次查询多个key，如果某个key不存在，对应位置返回nil。

#### 底层实现

String看起来简单，但实际有三种编码方式：

- INT 编码：这个很好理解，就是存一个整型，可以用long表示的整数就以这种编码存储；
- EMBSTR 编码：如果字符串小于等于阈值字节，使用EMBSTR编码；
- RAW 编码：字符串大于阈值字节，则用RAW编码。

这个阈值追溯源码的话，是用常量`OBJ_ENCODING_EMBSTR_SIZE_LIMIT`来表示，3.2版本之前是39字节，3.2版之后是44字节。

EMBSTR 和 RAW 都是由 redisObject 和 SDS 两个结构组成，它们的差异在于，EMBSTR 下redisObject 和 SDS 是连续的内存，RAW 编码下 redisObject 和 SDS 的内存是分开的。

EMBSTR 优点是 redisObject 和 SDS 两个结构可以一次性分配空间，缺点在于如果重新分配空间，整体都需要再分配，所以 EMBSTR 设计为只读，任何写操作之后 EMBSTR 都会变成RAW，理念是发生过修改的字符串通常会认为是易变的。

EMBSTR 内存如下：

![RedisEMBSTR内存](../picture/RedisEMBSTR内存.png)

RAW 内存如下：

![RedisRAW内存](../picture/RedisRAW内存.png)

随着我们的操作，编码可能会转换：

INT -> RAW：当存的内容不再是整数，或者大小超过了long的时候；

EMBSTR -> RAW：任何写操作之后 EMBSTR 都会变成RAW。

字符串编码 EMBSTR 和 RAW 都包含一个结构叫 SDS，它是 Simple Dynamic String 的缩写，即简单动态字符串，这是 Redis 内部作为基石的字符串封装。

#### 为什么要 SDS？

在C语言中，字符串用一个 '\0' 结尾的 char 数组表示。

比如 "hello niuniu" 即 "hello niuniu\0" 。

C语言作为一个比较亲和底层的语言，很多基建就是这么简单，但是，对于某个应用来说不一定好用：

- 每次计算字符串长度的复杂度为O(N)；
- 对字符串进行追加，需要重新分配内存；
- 非二进制安全。

在 Redis 内部，字符串的追加和长度计算很常见，这两个简单的操作不应该成为性能的瓶颈，于是 Redis 封装了一个叫 SDS 的字符串结构，用来解决上述问题。

首先，Redis 中 SDS 分为 sdshdr8、sdshdr16、sdshdr32、sdshdr64，它们的字段属性都是一样，区别在于应对不同大小的字符串，我们以 sdshdr8 为例：

```c
// from Redis 7.0.8
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

其中有两个关键字段，一个是 len，表示使用了多少；一个是 alloc，表示一共分配了多少内存。这两个字段的差值（alloc-len）就是预留空间的大小。flags 则是标记是哪个分类，比如sdshdr8 的分类就是：

```c
#define SDS_TYPE_8  1
```

从 SDS 的结构中，可以看出来 SDS 是如何解决这三个问题的：

- 增加长度字段len，快速返回长度；
- 增加空余空间（alloc-len），为后续追加数据留余地；
- 不再以'\0'作为判断标准，二进制安全。

SDS可以预留空间，那么预留空间有多大呢，规则如下：

- len小于1M的情况下，alloc是2倍len大小，即预留len大小的空间；
- len大于1M的情况下，alloc是1M+len，即预留1M大小的空间。

简单来说，预留空间为 min(len，1M)。

### String面试与分析

#### 面试题

##### Set一个已有的数据会发生什么？

分析：常用操作考察。

回答：会覆盖原有的值，覆盖或者擦出键的过期时间。

##### 浮点型在String是用什么表示？

分析：基础知识考查，只有三种编码模式，INT只针对与整型，所以浮点型必然是字符串存储。

回答：要将一个浮点数放入字符串对象里面，需要先将这个浮点数转换成字符串值，然后再保存转换所得的字符串值，比如浮点数3.14，对应就变成了"3.14"这个字符串。所以浮点数在字符串对象里面是用字符串值表示的。

##### String 可以有多大？

分析：基础知识考查，需要知道明确的数值，更进一步需要来源背书，再进一步需要思考为什么是这个数值。

回答：一个Redis字符串最大为512MB，官网有明确注明，看源码里也是直接写死的。可以通过`proto-max-bulk-len`来修改。

##### Redis 字符串是怎么实现的？

分析：这个问题，首先需要分情况说出编码类型，分别应用在什么场景。EMBSTR编码和RAW编码的选择阈值也比较重要，回答的时候，可以先说自己用的Redis版本的数值，面试官会感觉你确实有关注过编码的阈值。如果能记得是3.2版本之前是39，3.2版本之后才是44，也可以说出来，有一定程度的加分，但是说了这个差别可能会引导面试官追问为什么，如果记不清就不要说版本差别了。

回答：Redis字符串底层是String对象，String对象有三种编码方式：INT型、EMBSTR型、RAW型。如果是存一个整型，可以用long表示的整数就以INT编码存储；如果存字符串，当字符串长度小于等于一个阈值，使用EMBSTR编码；字符串大于阈值，则用RAW编码。在我用的5.0.5版本中阈值是44。

##### 为什么EMBSTR的阈值是44？

分析：这个问题是上个问题的追问，有以下几个要点：

1. Redis使用的是 jemalloc 作为内存分配器；
2. jemalloc 是以64字节作为内存单元做内存分配，如果超出了64个字节就超过了一个内存单元，则会用到RAW编码；反之，如果小于等于64个字节，就认为是一个小字符串，会用到EMBSTR编码。
3. 围绕64字节的关键分界来分析版本变化，Redis的字符串对象是由redisObject和sdshdr这两部分组成，redisObject大小为 4 + 4 + 24 + 32 + 64 = 128bits = 16bytes，这个是一直没变过的。

```c
// from Redis 3.9.5
#define LRU_BITS 24

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time or
                            * LFU data */
    int refcount;
    void *ptr;
} robj;
```

回顾sdshdr结构：

```mysql
// from Redis 3.9.5
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

sdshdr占用的内存大小：1byte + 1byte+ 1byte + 内联数组的大小，由于内联数组中还有一个'\0'占位一个字节，所以能用的大小为 64 - 16(redisObject) - 3(sdshdr非内容属性）- 1('\0') = 44。

回答：Redis是使用 jemalloc 内存分配器，jemalloc以64字节为阈值区分大小字符串。所以EMBSTR的边界数值，其实是受64这个阈值影响。redisObject 占用的内存大小由 redisObject和 sdshdr 这两部分组成，redisObject 16字节，sdshdr中已分配、已申请、标记三个字段固定占了3个字节，'\0'占了一个字节，能存放的数据就是 64-(16+4)= 44。

##### 你知道为什么EMBSTR曾经的阈值是39吗？

分析：实际上一般面试官一般不会主动这么问，一般聊到这个还是因为在上面问题里提到了版本差异。要回答的话，要从SDS结构来分析，sdshdr3.2之前版本结构如下：

```c
struct SDS {
  unsigned int capacity; // 4byte
  unsigned int len; // 4byte
  byte[] content; // 内联数组，长度为 capacity
}
```

当时都是通用的SDS结构，非数据字段一共占据了8个字节，为了节约内存，就在3.2版本之后将SDS分为sdshdr5（这个结构不被使用）、sdshdr8、sdshdr16、sdshdr32、sdshdr64，EMBSTR使用sdshdr8节约了6个字节，但多引入了一个flags字段占据1字节，所以相比3.2版本之前的SDS多了5个字节，从这一点也能看出Redis也是不断在优化内存和性能的。

回答：3.2之后的版本，SDS结构进行了拆分，EMBSTR用的sdshdr8，总容量和已使用容量字段减少了6个字节，但由于增加了一个flags字段，所以最终节约了5个字节。

##### SDS有什么用？

分析：Redis是用C语言写的，SDS可以说是对C字符串的封装，一般对比普通C字符串。可以从计算长度、扩容、缩容、二进制存储这几个场景来描述。

回答：主要有三点：

1. SDS包含已使用容量字段，O(1)时间快速返回有字符串长度，相比之下，C原生字符串需要O(n)。
2. 有预留空间，在扩容时如果预留空间足够，就不用再重新分配内存，节约性能，缩容时也可以将减少的空间先保留下来，后续可以再使用。
3. 不再以‘\0’作为判断标准，二进制安全，可以很方便地存储一些二进制数据。

### List

Redis List是一组连接起来的字符串集合。 List最大元素个数是 2^32 - 1 (4,294,967,295)。现在是 2 ^ 64 - 1。

#### 应用场景

List作为一个列表存储，属于比较底层的数据结构，可以使用的场景非常多，比如存储一批任务数据，存储一批消息等。

#### 常用操作

常用操作还是聚焦于创建、查询、更新和删除。

- 创建：即产生一个List对象，一般用LPUSH、RPUSH，分别对应从左侧进队列和右侧进队列。
- 查询：可以用LRANGE来进行范围查询，可以用LLEN查询元素个数。
- 更新：对列表对象而言就是追加元素、删除元素等操作，涉及LPUSH, LPOP, RPUSH, RPOP等操作。
- 删除：针对List对象本身的生成和销毁，用DEL命令。

##### 写操作

`LPUSH key value [value ...]`

功能：从头部增加元素，返回值为List中元素的总数。

`RPUSH key value [value ...]`

功能：从尾部增加元素，返回值为List中元素的总数。

`LPOP key`

功能：移出并获取列表的第一个元素。

`RPOP key`

功能：移出并获取列表的最后一个元素。

`DEL key [key ...]`

功能：删除对象，返回值为删除成功了几行。

##### 读操作

`LLEN key`

功能：查看List的长度，即List中元素的总数。

`LRANGE key start stop`

功能：查看start到stop为角标的元素。

#### 底层实现

##### 编码方式

3.2 版本之前，List 对象有两种编码方式，一种 ZIPLIST，另一种是 LINKEDLIST。

当满足如下条件时，用 ZIPLIST 编码：

1. 列表对象保存的所有字符串对象长度都小于64字节；
2. 列表对象元素个数少于 512 个，注意，这是 LIST 的限制，而不是 ZIPLIST 的限制/

ZIPLIST底层用压缩列表实现，ZIPLIST内存排列得很紧凑，可以有效节约内存空间。

如果不满足 ZIPLIST 编码的条件，则使用 LINKEDLIST 编码。使用 LINKEDLIST 数据是以链表的形式连接在一起，实际上删除更为灵活，但是内存不如ZIPLIST紧凑，所以只有在列表个数或节点数据长度比较大的时候，才会使用LINKEDLIST编码，以加快处理性能，一定程度上牺牲了内存。

##### QUICKLIST横空出世

上面的分析有说到，ZIPLIST 是为了在数据较少时节约内存，LINKEDLIST 是为了数据多时提高更新效率，ZIPLIST 数据稍多时插入数据会导致很多内存复制。但如果节点非常多的情况，LINKEDLIST 链表的节点就很多，会占用不少的内存。

3.2版本就引入了 QUICKLIST。QUICKLIST 其实就是 ZIPLIST 和 LINKEDLIST 的结合体。

LINKEDLIST 原来是单个节点，只能存一个数据，现在单个节点存的是一个 ZIPLIST，即多个数据。

![RedisQuickList](../picture/RedisQuickList.png)

这种方案其实是用 ZIPLIST、LINKEDLIST 综合的结构，取代二者本身。当数据较少的时候，QUICKLIST 的节点就只有一个，此时其实相当于就是一个 ZIPLIST。当数据很多的时候，则同时利用了 ZIPLIST 和 LINKEDLIST 的优势。

##### ZIPLIST 优化

ZIPLIST 本身存在一个连锁更新的问题，所以 Redis 7.0 之后，使用了 LISTPACK 的编码模式取代了 ZIPLIST，而他们其实本质都是一种压缩的列表，所以其实可以统一叫做压缩列表。

### 底层数据结构压缩列表

压缩列表，顾名思义，就是排列紧凑的列表。压缩列表在Redis中有两种编码方式，一种是ZIPLIST，一种是LISTPACK，LISTPACK是在Redis 5.0引入，直到Redis7.0完全替换了ZIPLIST，可以说是ZIPLIST的进阶版本。压缩列表主要用做为底层数据结构提供紧凑型的数据存储方式。

#### ZIPLIST 整体结构

虽然已经有LISTPACK，但实际面试中聊得比较多的，还是ZIPLIST。Redis代码注释中，非常清晰描述了ZIPLIST的结构：

```c
* The general layout of the ziplist is as follows:
*
* <zlbytes> <zltail> <zllen> <entry> <entry> .. <entry> <zlend>
```

比如这就是有3个节点的ziplist结构：

![RedisZipList结构](../picture/RedisZipList结构.png)

- zlbytes：表示该ZIPLIST一共占了多少字节数，这个数字是包含zlbytes本身占据的字节的。
- zltail：表示相对最后一个节点偏移了多少个字节，通过这个字段可以快速定位到尾部节点。
- zllen：表示有多少个数据节点，在本例中就有3个节点。
- entry1~entry3：表示压缩列表数据节点。
- zlend：一个特殊的entry节点，表示ZIPLIST的结束。

#### ZIPLIST 节点结构

ZIPLIST ENTRIES定义如下：

```c
 <prevlen> <encoding> <entry-data>
```

- prevlen：表示上一个节点的数据长度。通过这个字段可以定位上一个节点的数据，实现从后往前操作，所以压缩列表才可以从后往前遍历。如果前一节点的长度小于254字节， 那么prevlen属性需要用1字节长的空间来保存这个长度值。如果前一节点的长度大于等于254字节，那么prevlen属性需要用5字节长的空间来保存这个长度值。
- encoding：编码类型。
- entry-data：实际的数据。

##### encoding

encoding字段是一个整型数据，其二进制编码由内容数据的类型和内容数据的字节长度两部分组成，根据内容类型有如下几种情况：

| 编码                                             | 大小  | 类型                                                         |
| ------------------------------------------------ | ----- | ------------------------------------------------------------ |
| 00pppppp                                         | 1字节 | String类型，且字符串长度小于2^6，即小于等于63                |
| 01pppppp\|qqqqqqqq                               | 2字节 | String类型，长度小于2^14次方，即小于等于16383                |
| 10000000\|qqqqqqqq\|rrrrrrrr\|ssssssss\|tttttttt | 5字节 | String类型，长度小于2^32次方                                 |
| 11000000                                         | 1字节 | 2个字节的int16类型                                           |
| 11010000                                         | 1字节 | 4个字节的int32类型                                           |
| 11111110                                         | 1字节 | 8个字节的int64类型                                           |
| 1111xxxx                                         | 1字节 | xxxx从1到13一共13个值，这时就用这13个值来表示真正的数据。注意，这里是表示真正的数据，而不是数据长度了，这种情况<entry-data>就没有了 |

注意，如果是String类型，那么encoding有两部分，一般是前几位标识类型、后几位标识长度。

但如果是int类型，整体1字节编码，就只是标识了类型，为什么没有大小呢，因为int的具体类型就自带了大小，比如int32，就是32位，4字节的大小，不需要encoding特别标识。

encoding的编码规则比较复杂，我们其实只需要理解它的核心思想，面试中能讲清楚怎么区分不同类型即可，不用去背它的这些具体编码，这个很难记住，也没有必要去记。

#### ZIPLIST 查询数据

##### 查询 ZIPLIST 的数据总量

由于 ZIPLIST 的 header 定义了记录节点数量的字段 zllen，所以通常是可以在 O(1) 时间复杂度直接返回的，为什么说通常呢？是因为 zllen 是2个字节的，当 zllen 大于 65535 时，zllen 就存不下了，此时 zllen 等于65535，所以真实的节点数量需要遍历来得到。

这样设计的原因是Redis中应用ZIPLIST都是为了节点个数少的场景，所以将zllen设计得较小，节约内存空间。

##### 在ZIPLIST中查询指定数据的节点

在ZIPLIST中查询指定数据的节点，需要遍历这个压缩列表，平均时间复杂度是O(N)。

#### ZIPLIST更新数据

ZIPLIST的更新就是增加、删除数据，ZIPLIST提供头尾增减的能力，但是操作平均时间复杂度是O(N)，因为在头部增加一个节点会导致后面节点都往后移动，所以更新的平均时间复杂度，可以看作O(N)。

其中要注意的是更新操作可能带来连锁更新。注意上面所说的增加节点导致后移，不是连锁更新。连锁更新是指这个后移，发生了不止一次，而是多次。

比如增加一个头部新节点，后面依赖它的节点，需要prevlen字段记录它的大小，原本只用1字节记录，因为更新可能膨胀为5字节，然后这个entry的大小就也膨胀了。所以，当这个新数据插入导致的后移完成之后，还需要逐步迭代更新。这种现象就是连锁更新，时间复杂度是O(N^2)。

大家可能会比较担心连锁更新带来的性能问题，但在实际的业务中，很少会刚好遇到需要迭代更新超过2个节点的情况，所以ZIPLIST更新平均时间复杂度，还是可以看作O(N)。不过，ZIPLIST最大的问题还是连锁更新导致性能不稳定。

#### LISTPACK 优化

##### 连锁更新原因分析

LISTPACK是为了解决ZIPLIST最大的痛点——连锁更新，我们先来看，ZIPLIST的问题本源。

我们知道，ZIPLIST需要支持LIST，LIST是一种双端访问结构，所以需要能从后往前遍历，上面有讲，ZIPLIST的数据节点的结构是这样的：

```c
<prevlen> <encoding> <entry-data>
```

其中，prevlen就表示上一个节点的数据长度，通过这个字段可以定位上一个节点的数据，可以说，连锁更新问题，就是因为prevlen导致的。

##### 解决方案

我们需要一种不记录prevlen，并且还能找到上一个节点的起始位置的办法，Redis使用了很巧妙的一种方式。我们直接看LISTPACK的节点定义：

```c
<encoding-type><element-data><element-tot-len>
```

encoding-type是编码类型，element-data是数据内容，element-tot-len存储整个节点除它自身之外的长度。

element-tot-len 所占用的每个字节的第一个 bit 用于标识是否结束。0是结束，1是继续，剩下7个bit来存储数据大小。当我们需要找到当前元素的上一个元素时，我们可以从后向前依次查找每个字节，找到上一个Entry的 element-tot-len 字段的结束标识，就可以算出上一个节点的首位置了。

举个例子：

如果上个节点的element-tot-len为 00000001 10000100，每个字节第一个bit标志是否结束，所以这里的element-tot-len一共就两个字节，大小为0000001 0000100，即132字节。

### List面试与分析

#### 面试题

##### List 是完全先入先出吗？

分析：题目所说的完全先入先出，很明显就是指只允许从尾部入队、从头部出队，而 List 是一个双端操作对象，所以不是完全先入先出，他也可以后入先出。这道题就是考察是否对 List有最基本的认识。

回答：List是双端操作对象，所以不是完全的先入先出，List也可以后入先出。

##### List 对象底层编码方式是什么？

分析：在 Redis3.2 版本之前和之后，List对象的底层编码方式是不同的。面试的时候如果能对不同版本Redis的List对象底层编码方式做出分析，是一个加分项。

回答：在 3.2 版本之前，List 对象的编码是 ZIPLIST 和 LINKEDLIST。ZIPLIST 适用于元素数量较少、且元素都较短的情况，否则用 LINKEDLIST。

3.2 版本之后，List 对象的编码全部由 QUICKLIST 实现。QUICKLIST 是一个压缩列表组成的双向链表，结合了 ZIPLIST 和 LINKEDLIST 两者的优点，在后面比较新的版本中 ZIPLIST 优化为了 LISTPACK。

##### ZIPLIST是怎么压缩数据的？

分析：这个问题就是对 ZIPLIST 数据结构本身的考察。ZIPLIST 是为了节约内存而开发的，回答这个问题可以从 ZIPLIST 的结构入手。下图是 ZIPLIST 的结构：

```c
* The general layout of the ziplist is as follows:
*
* <zlbytes> <zltail> <zllen> <entry> <entry> .. <entry> <zlend>
```

其中entry结构为：

```c
<prevlen> <encoding> <entry-data>
```

为了方便记忆和理解，可以将 ZIPLIST 分为三部分讲述：

1. 结构头，即header，包括`<zlbytes> <zltail> <zllen>`字段。
2. 数据部分，即entry列表，entry中有`<prevlen> <encoding> <entry-data>`字段。
3. 结尾标识，即zlend。

回答：ZIPLIST 可以看作三个部分，结构头、数据部分、结尾标识。其中，结构头记录了字节数、起始地址距尾部节点距离、节点个数；数据部分记录了上个节点的长度、内容编码，内容本身；最后有一个1字节的结尾标识。

##### ZIPLIST 下 List 可以从后往前遍历吗？

分析：首先，一切的基石，在于要知道 List 是一种双端数据结构，无论哪种底层编码，都需要能支持从后往前遍历。

接着，需要阐述 ZIPLIST 是如何做到从后往前遍历的，其实还是在考察ZIPLIST的数据结构。

ZIPLIST 下 entry 的结构包含了上一个节点的长度，所以可以通过上一个节点的长度，找到上个节点的起始位置，这样就能实现从后往前遍历。

回答：可以，List 是双端数据结构，无论哪种底层编码，都需要能支持从后往前遍历。ZIPLIST每个节点中都保存了上一个节点的长度，所以可以用当前节点地址减去上一个节点长度来找到上个节点起始位置，进而实现从后往前的遍历。

##### 在ZIPLIST数据结构下，查询节点个数的时间复杂度是多少？

分析：由于 ZIPLIST 的 header 中定义了记录节点数量的字段 zllen，所以通常是可以在 O(1) 时间复杂度直接返回，为什么说通常呢？是因为 zllen 是2个字节的，当 zllen 大于 65535 时，zllen 就存不下了，所以真实的节点数量需要遍历来得到。

回答：在 ZIPLIST 编码下，查询节点个数的时间复杂度是 O(1)，因为 ZIPLIST 中的 header 定义了记录节点数量的字段，但是这里也有个限制，记录节点数量的字段只有 2 字节，也就说如果节点数量超过 65535，就失效了，此时只能通过 O(n) 复杂度的遍历来查节点总数。

##### LINKEDLIST编码下，查询节点个数的时间复杂度是多少？

分析：这个问题我们可以回顾一下LINKEDLIST的表头结构：

```c
// from Redis 5.0.5
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

LINKEDLIST的表头结构中定义了链表所包含节点数量的字段 len，所以 LINKEDLIST 编码下，查询节点个数的时间复杂度是 O(1)。

回答：LINKEDLIST 编码下，查询节点个数的时间复杂度是 O(1)。因为 LINKEDLIST 的表头结构中定义了链表所包含节点数量的字段 len。

### Set

#### 是什么？

Redis 的 Set 是一个不重复、无序的字符串集合。其中 IntSet 是有序的。

#### 适用场景

适用于无序集合场景，比如某个用户关注了哪些公众号，这些信息就可以放进一个集合，Set还提供了查交集、并集的功能，可以很方便地实现共同关注的能力。

#### 常用操作

我们还是从创建、查询、更新、删除这几个基本操作来了解 Set。

- 创建：即产生一个Set对象，可以使用 SADD 创建。
- 查询：Set 的查询非常丰富，SISMEMBR 可以查询元素是否存在的；SCARD、SMEMBERS、SSCAN可以查询集合元素数据；SINTER、SUNION、SDIFF可以对多个集合查交集并集和差异。
- 更新：可以使用SADD增加元素，SREM删除元素。
- 删除：DEL可以删除一个Set对象。

##### 写操作

`SADD key member [member ...]`

功能：添加元素，返回值为成功添加了几个元素。

`SREM key member [member ...]`

功能：删除元素，返回值为成功删除了几个元素。

##### 读操作

`SISMEMBER key member`

功能：查询元素是否存在。

`SCARD key`

功能：查询集合元素个数。

`SMEMBERS key`

功能：查看集合的所有元素。

`SSCAN key cursor [MATCH pattern] [COUNT count]`

功能：查看集合元素，可以理解为指定游标进行查询，可以指定个数，默认个数为10。也可以使用 Match 模糊查询。

`SINTER key [key ...]`

功能：返回在第一个集合里，同时在后面所有集合都存在的元素。

`SUNION key [key ...]`

功能：返回所有集合的并集，集合个数大于等于2。

`SDIFF key [key ...]`

功能：返回第一个集合有，且在后续集合中不存在的元素，集合个数大于等于2，注意，是以第一个集合和后面比，看第一个集合多了哪些元素。

#### 底层实现

Redis出于性能和内存的综合考虑，也支持两种编码方式，如果集群元素都是整数，且元素数量不超过512个，就可以用INTSET编码，结构如下图，可以看到INTSET排列比较紧凑，内存占用少，但是查询时需要二分查找。

![RedisSet底层实现IntSet](../picture/RedisSet底层实现IntSet.png)

如果不满足 INTSET 的条件，就需要用 HASHTABLE，HASHTABLE 结构如下图，可以看到HASHTABLE 查询一个元素的性能很高，能 O(1) 时间就能找到一个元素是否存在。

![RedisSet底层实现HashTable](../picture/RedisSet底层实现HashTable.png)

### HSet

#### 是什么？

Redis HSet是一个 field、value 都为 string 的 hash 表，存储在 Redis 的内存中。

Redis 中每个 hash 可以存储 2^32 - 1 键值对（40多亿）。

#### 适用场景

适用于 O(1) 时间字典查找某个 field 对应数据的场景，比如任务信息的配置，就可以任务类型为 field，任务配置参数为 value。

#### 常用操作

还是从创建、查询、更新、删除这几个基本操作来了解HSet。

- 创建：即产生一个 HSet 对象，可以使用 HSET、HSETNX 创建。
- 查询：支持 HGET 查询单个元素；HGETALL 查询所有数据；HLEN 查询数据总数；HSCAN 进行游标迭代查询。
- 更新：HSET可以用于增加新元素，HDEL删除元素。
- 删除：DEL可以删除一个HSet对象。

##### 写操作

`HSET key field value`

功能：为集合对应 field 设置 value 数据。

`HSETNX key field value`

功能：如果 field 不存在，则为集合对应 field 设置 value 数据。

`HDEL key field [field ...]`

功能：删除指定 field，可以一次删除多个。

`DEL key [key ...]`

功能：删除 HSET 对象。

##### 读操作

`HGETALL key`

功能：查找全部数据。

`HGET key field`

功能：查找某个field。

`HLEN key`

功能：查找HSet中元素总数。

`HSCAN key cursor [MATCH pattern] [COUNT count]`

功能：从指定位置查询一定数量的 HSet 数据。

#### 底层原理

HSet底层有两种编码结构，一个是压缩列表，一个是 HASHTABLE。同时满足以下两个条件，用压缩列表：

1. HSet 对象保存的所有值和键的长度都小于64字节；
2. HSet 对象元素个数少于512个。

两个条件任何一条不满足，编码结构就用HASHTABLE。

ZIPLIST 之前有讲解过，其实就是在数据量较小时将数据紧凑排列，对应到HSet，就是将filed-value当作entry放入ZIPLIST，结构如下：

![RedisHset底层原理Ziplist](../picture/RedisHset底层原理Ziplist.png)

HASHTABLE 在之前无序集合Set中也有应用，和 Set 的区别在于，在 Set 中 value 始终为NULL，但是在HSet中，是有对应的值的。

![RedisHset底层实现Hashtable](../picture/RedisHset底层实现Hashtable.png)

### 底层结构 HASHTABLE

#### HASHTABLE概述

HASHTABLE，可以想象成目录，要翻看什么内容，直接通过目录能找到页数，翻过去看。如果没有目录，我们需要一页一页往后翻，效率很低。在计算机世界里，HASHTABLE 就扮演着这样一个快速索引的角色，通过 HASHTABLE 我们可以只用 O(1) 时间复杂度就能快速找到 key 对应的 value。

#### HASHTABLE结构

Redis 中 HASHTABLE 的结构如下：

```c
// from Redis 5.0.5
/* This is our hash table structure.  */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

- table：指向实际hash存储。存储可以看做一个数组，所以是 *table的表示，在C语言中 *table 可以表示一个数组。
- size：哈希表大小。实际就是 dictEntry 有多少元素空间。
- sizemask：哈希表大小的掩码表示，总是等于 size-1。这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面，规则 Index = hash&sizemask。
- used：表示已经使用的节点数量。通过这个字段可以很方便地查询到目前 HASHTABLE 元素总量。

#### Hash表渐进式扩容

渐进式扩容顾名思义就是一点一点慢慢扩容，而不是一股脑直接做完，那具体流程是怎样的呢？

其实为了实现渐进式扩容，Redis中没有直接把dictht暴露给上层，而是再封装了一层：

```c
// from Redis 5.0.5
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

可以看到dict结构里面，包含了2个 dictht 结构，也就是2个 HASHTABLE 结构。

![Redisdict](../picture/Redisdict.png)

实际上平常使用的就是一个HASHTABLE，在触发扩容之后，就会两个HASHTABLE同时使用，详细过程是这样的：

当向字典添加元素时，发现需要扩容就会进行Rehash。Rehash的流程大概分成三步：

首先，生成新Hash表ht[1]，为 ht[1] 分配空间。新表大小为第一个大于等于原表2倍used的2次方幂。举个例子，原表如果 used=500，2倍就是1000，那第一个大于1000的2次方幂则为1024。此时字典同时持有 ht[0] 和 ht[1] 两个哈希表。字典的偏移索引从静默状态-1，设置为0，表示 Rehash 工作正式开始。

然后，迁移 ht[0 ]数据到 ht[1]。在 Rehash 进行期间，每次对字典执行增删查改操作，程序会顺带迁移一个 ht[0] 上的数据，并更新偏移索引。与此同时，部分情况周期函数也会进行迁移。

随着字典操作的不断执行，最终在某个时间点上，ht[0] 的所有键值对都会被 Rehash 至 ht[1]，此时再将 ht[1] 和 ht[0] 指针对象互换，同时把偏移索引的值设为 -1，表示 Rehash 操作已完成。

总结一下，渐进式扩容的核心就是定时迁移+操作时顺带迁移。

#### 扩容时机

Redis 提出了一个负载因子的概念，负载因子表示目前 Redis HASHTABLE 的负载情况，是游刃有余，还是不堪重负了。我们设负载因子为 k，那么 k = ht[0].used / ht[0].size，也就是使用空间和总空间大小的比例。

Redis会根据负载因子的情况来扩容：

1. 负载因子大于等于1，说明此时空间已经非常紧张。新数据是在链表上叠加的，越来越多的数据其实无法在 O(1) 时间复杂度找到，还需要遍历一次链表，如果此时服务器没有执行 BGSAVE 或 BGREWRITEAOF 这两个复制命令，就会发生扩容。
2. 负载因子大于5，这时候说明 HASHTABLE 真的已经不堪重负了，此时即使是有复制命令在进行，也要进行 Rehash 扩容。

##### 缩容

扩容是数据太多存不下了，那如果太富裕呢，比如原来可能数据较多，发生了扩容，但后面数据不再需要，被删除了，此时多余的空间就是一种浪费。

缩容过程其实和扩容是相似的，也是渐进式缩容。同样的，Redis还是用负载因子来控制什么时候缩容。当负载因子小于 0.1，即负载率小于 10%，此时进行缩容，新表大小为第一个大于等于原表 used 的2次方幂。

### Set&HSet 面试与分析

#### 面试考点分析

Set是一个无序集合对象，HSet是字典对象，他们底层都有着相同的实现HashTable。

面试的考察点集中在 Set、HSet常规操作和底层实现，常规操作必须熟练掌握，底层实现要了解其编码方式，其中HashTable是重中之重，面试大热门考点。

#### 面试题

##### Set是有序的吗？

分析：Set 的底层实现是整数集合或字典，前者是有序的，后者是无序的。

回答：Set 的底层实现是整数集合或字典，前者是有序的，后者是无序的。整体来看，建议不依赖SET的顺序。

##### Set编码方式？

分析：Set 底层使用了两种编码，一种是整数集合，另一种是字典。成员的数据结构和成员数量会触发 Set 更改底层编码，当 Set 同时满足元素都是整数且元素个数不超过512这两个条件，会使用整数集合编码，否则使用字典编码。

回答：Set 使用整数集合和字典作为底层编码，当元素都是整数同时元素个数不超过512个，会使用整数集合编码，否则使用字典编码。

##### HSet 的编码方式是什么？

分析：HSet 底层有两种编码结构，一个是 ZIPLIST，一个是 HashTable。ZIPLIST 适用于元素较少且单个元素长度较小的情况，这里的阈值分别是元素个数少于512个，值和键长度都小于64字节。

回答：一个是 ZIPLIST，一个是 HashTable。ZIPLIST 适用于元素较少且单个元素长度较小的情况，其它情况使用 HashTable。

##### HSet 查找某个 key 的平均时间复杂度是多少？

分析：这种问对象复杂度的，要考虑多种底层编码。ZIPLIST 需要遍历，平均复杂度为 O(N)，HashTable 是字典，可以 O(1) 找到对应 key。

回答：HSet 有两种底层结构，ZIPLIST 是 O(N)，HashTable 则是 O(1)。

##### HashTable 查找元素总数的平均时间复杂度是多少?

分析：这题考察的是 Redis 字典的表头结构，如果表头结构中有储存键值对个数的字段，那么查找元素总数的平均时间复杂度就是 O(1)，而如果没有这个字段，那字典就需要去遍历所有的键值对。

下面是Redis字典的表头结构：

```c
// from Redis 5.0.5 
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemark;
    // 键值对数量
    unsigned long used;
} dictht;
```

可以发现字典的表头结构中的used，就记录了当前键值对数量的字段。

回答：HashTable 查找元素总数的平均时间复杂度是 O(1)，因为 HashTable 的表头结构中有储存键值对数量的字段 used。

##### 为什么要用两种编码方式?

分析：我们可以从编码转换的条件来进行思考，Set的底层编码从 INTSET 到 HASHTABLE 的条件是元素个数或者元素类型的变化。所以采用两种编码方式的原因是 INTSET 更节约内存，所以在小数据量时使用，而数据多起来了，需要 HASHTABLE 的查找性能。

实际上如果 Set 中保存的所有元素都是整数，而且元素个数不是特别多的情况下，使用 intset会比较节约内存，Redis 用一个含有三个字段的结构体来表示 intset，分别是编码方式、元素数量和实际存储元素的有序柔性数组：

```c
typedef struct intset {
    uint32_t encoding;  //编码格式
    uint32_t length;    //元素数量
    int8_t contents[];  //保存元素的数组
} intset; 
```

intset 用来保存元素的数组默认情况下是 int16 编码，后续如果插入更大的整数，才会升级到int32 或者 int64 编码。这种策略可以尽可能的节约内存以及提升整数集合的灵活性。

但是升级也有弊端，升级之后，整个数组的编码会变成与最大元素的类型一致。假如这个时候，元素的数量非常多，就不那么节约内存了，而且数组遍历的平均时间复杂度是 O(logn)，不如使用字典编码。

回答：Set 的底层编码是整数集合和字典，当元素数量小并且全部是整数的时候，会使用整数集合编码，更加的节约内存。元素数量变大会使用字典编码，查找元素的速度会更快。

##### 一个数据在 HashTable中 的存储位置，是怎么计算的?

分析：当一个新的键值对要插入到 HashTable 中时，首先会使用哈希函数计算这个 key 的哈希值，Redis 使用的哈希算法是 MurmurHash2 哈希算法，然后把哈希值和哈希掩码做与运算得到索引值，哈希掩码其实就是哈希表数组的大小减去1。然后程序会根据索引值把键值对插入到相应的位置。

回答：首先会通过哈希函数计算出 key 的哈希值，然后与哈希掩码做与运算得到索引值，索引值就是这个数据在 HashTable 中的存储位置。

##### HashTable 怎么扩容?

分析：HashTable 的扩容通过渐进式 rehash 操作来完成。

回答：首先程序会为 HashTable 的1号表分配空间，空间大小是第一个大于等于0号表大小 * 2的 2 ^ n。在 rehash 进行期间，标记位 rehashidx 从0开始，每次对字典的键值对执行增删改查操作后，都会将 rehashidx 位置的数据迁移到 1 号表，然后将 rehashidx 加1，随着字典操作的不断执行，最终 0 号表的所有键值对都会被 rehash 到1号表上。之后，1 号表会被设置成 0 号表，接着在 1 号表的位置创建一个新的空白表。

##### HashTable怎么缩容?

分析：HashTable 的缩容也通过渐进式 rehash 操作来完成。

回答：首先程序会为 HashTable 的1号表分配空间，新表大小为第一个大于等于原表 used 的 2次方幂。在 rehash 进行期间，标记位 rehashidx 从0开始，每次对字典的键值对执行增删改查操作后，都会将 rehashidx 位置的数据迁移到1号表，然后将rehashidx加1，随着字典操作的不断执行，最终 0 号表的所有键值对都会被 rehash 到1号表上。之后，1 号表会被设置成 0 号表，接着在 1 号表的位置创建一个新的空白表。

##### 什么时候扩容，什么时候缩容

分析：HashTable 的扩容与缩容由哈希表的负载因子决定，负载因子 = 键值对数量 / 哈希表大小。

回答：当以下两个条件中的任意一个被满足时，哈希表会自动开始扩容：

- 服务器目前没有在执行 BGSAVE 或者 BGREWRITEAOF，并且负载因子 >= 1；
- 服务器目前正在执行 BGSAVE 或者 BGREWRITEAOF，并且负载因子 >= 5。

缩容：

当哈希表的负载因子小于 0.1 时，程序会自动开始对哈希表进行收缩操作。

### ZSet 

#### 是什么？

ZSet 就是有序集合，也叫 Sorted Set，是一组按关联积分有序的字符串集合，这里的分数是个抽象概念，任何指标都可以抽象为分数，以满足不同场景。

积分相同的情况下，按字典序排序。

#### 适用场景

用于需要排序集合的场景，最为典型的就是游戏排行榜。

#### 常用操作

##### 写操作

`ZADD key [NX|XX] [CH] [INCR] score member [score member ...]`

功能：向Sorted Set增加数据，如果是已经存在的Key，则更新对应的数据。

`ZREM key member [member ...]`

功能：删除ZSet中的元素。

##### 读操作

`ZCARD key`

功能：查看ZSet中成员总数。

`ZRANGE key start stop [WITHSCORES]`

功能：查询从 start 到 stop 范围的 ZSet 数据，WITHSCORES 选填，不写输出里就只有 key，没有 score 值。

`ZREVRANGE key start stop [WITHSCORES]`

功能：即 reverse range，从大到小遍历，WITHSCORES 选填，不写输出里就只有 key，没有score 值。

`ZCOUNT key min max`

功能：计算 min-max 积分范围的成员个数。

`ZRANK key member`

功能：查看 ZSet 中 member 的排名索引，索引是从0开始，所以如果排第一，索引就是0。

`ZSCORE key member`

功能：查询ZSet中成员的分数。

#### 底层实现

ZSet 底层编码有两种，一种是 ZIPLIST，一种是 SKIPLIST + HT。

在 ZSet 中 ZIPLIST 也是用于数据量比较小时候的内存节省，结构如下：

![RedisZsetZiplist结构](../picture/RedisZsetZiplist结构.png)

如果满足如下规则，ZSet 就用 ZIPLIST 编码：

1. 列表对象保存的所有字符串对象长度都小于 64 字节；
2. 列表对象元素个数少于 128 个。

两个条件任何一条不满足，编码机构就用 SKIPLIST+HT，其中 SKIPLIST 也是底层数据结构。SKIPLIST 是一种可以快速查找的多级链表结构，通过 skiplist 可以快速定位到数据所在。它的排名操作、范围查询性能都很高。除了 SKIPLIST，Redis 还使用了 HT 来配合查询，这样可以在 O(1) 时间复杂度查到成员的分数值。

### 底层数据结构跳表

#### 跳表的结构

![Redis跳表的结构](../picture/Redis跳表的结构.png)

可以看到，这个图某些节点不光只有一层，如果只用普通链表，只能一步一步往后走，如果用这种有高层的节点，那是不是可以一次多走几步，理论上，层次越高平均步长越大，但并不完全像示意图一样是绝对均衡的，节点的层高其实是概率随机的。

为了理解这个结构有什么好处，我们分几个场景来分析：

场景一：查找分数为35的数据

如果只有原始的链表，那需要走4步，如果有图中的二级索引，只用走一步，那如果找45呢，其实就是从第1个节点出发，通过二级索引走到35，再查看到下一个节点是65，已经超过了，所以降低到下方的索引，这里直接是原始链表，再走一步，就到了。

场景二：插入一条score为36的数据

首先，定位到第一个比score大的位置，这里是45，定位方式和查询类似，不再赘述。然后，构造一个新的节点，这里我们假设节点层高随机到3，最后，将各层链表补齐，其实就是在每一层进行链接，效果如图：

![Redis跳表举例](../picture/Redis跳表举例.png)

在Redis中，跳表是用来支持有序集合的，标准的跳表有如下限制：

1. score值不能重复；
2. 只有向前指针，没有回退指针。

所以Redis对跳表做了一些优化，下面我们来看看Redis的跳表。

#### Redis的跳表实现

我们直接看这个示意图，score可以重复并且我们的每个节点多了一下回退指针。

![Redis跳表实现](../picture/Redis跳表举例.png)

下面我们结合源码，来和示意图对照学习，以加深理解，这是Redis 跳表单个节点的定义：

```c
// from Redis 7.0.8
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;
```

- ele：SDS结构，用来存储数据。
- score：节点的分数，浮点型数据。
- backward：指向上一个节点的回退指针。
- level：是个zskiplistLevel结构体数组，zskiplistLevel这个结构体包含了两个字段，一个是forward，指向该层下个能跳到的节点，span记录了距离下个节点的步数，数组结构就表示每个节点都可能是个多层结构。

#### Redis跳表单个节点有几层？

层次的决定，需要比较随机，才能在各个场景表现出较为平均的性能，这里Redis使用概率均衡的思路来确定新插入节点的层数。

Redis 跳表决定每一个节点，是否能增加一层的概率为25%，而最大层数限制在 Redis 5.0 是64层，在 Redis 7.0 是32层。

#### Redis跳表的性能优化了多少

这个其实可以很容易想到，跳表的查找过程，其实是走高层，行得通跳过去，行不通走相对下层，很像二叉树的另一种表现形式，实际上他们性能也是差不多的，平均时间复杂度都是O(logn)，区别是二叉树平均情况下也是 O(logn) 比较稳定，而跳表的最坏时间复杂度是O(N)。当然，实际的生产过程中，体现出来的基本都是跳表的平均时间复杂度。

有序集合无论是查找、还是增加删除元素，都是需要先定位到数据位置，所以跳表将这三个操作的时间复杂度，都从O(N)降低到了log(N)。

### ZSet面试与分析

#### 面试题

##### ZSet底层有几种编码方式？

分析：基础知识考查，ZSet的底层编码可以是 ziplist 或者 skiplist+字典 ，需要分情况说出不同的编码类型。ziplist 和 skiplist + 字典 编码的选择阈值不一定可以记得清，记不清说大概是128或者256也可。当有ZSet对象可以同时满足以下两个条件时，对象使用ziplist编码：

- ZSet保存的元素数量小于128个；
- ZSet保存的所有元素成员的长度都小于64字节。

不能满足以上两个条件的有序集合对象将使用 skiplist + 字典编码。

回答：ZSet 就是有序集合对象，ZSet 对象的底层有两种编码方式：ziplist 或者 skiplist+字典。如果一个ZSet对象中的所有元素同时满足：元素数量小于128个以及所有元素成员的长度都小于64字节，那么会使用 ziplist 编码，否则使用 skiplist+字典编码。

##### 跳表编码模式下，查询节点总数的平均时间复杂度是多少？

分析：这个问题是对 Redis 跳表数据结构的考察，首先我们可以想想跳表的表头结构：

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    int level;
} zskiplist;
```

跳表的表头结构中定义了保存节点数量的字段：length，所以对 Redis 跳表查询节点总数的平均时间复杂度应该为 O(1)。我们也可以进一步查看相关的API底层源码：

```c
unsigned int zsetLength(robj *zobj) {
    int length = -1;
    // O(N)
    if (zobj->encoding == REDIS_ENCODING_ZIPLIST) {
        length = zzlLength(zobj->ptr);
    // O(1)
    } else if (zobj->encoding == REDIS_ENCODING_SKIPLIST) {
        length = ((zset*)zobj->ptr)->zsl->length;
    } else {
        redisPanic("Unknown sorted set encoding");
    }
    return length;
}
```

查询有序集合成员总数的API是 zsetLength，而当编码模式是REDIS_ENCODING_SKIPLIST，源码中会直接返回跳表表头结构的length字段，平均时间复杂度是 O(1)。

回答：跳表编码模式下，查询节点总数的平均时间复杂度是 O(1)，因为跳表的表头结构中定义了一个保存节点数量的字段 length，源码中调用查询节点总数的 API 时会直接返回这个字段。

##### 跳表中一个节点的层高是怎么决定的？

分析：这个问题是对跳表这种数据结构本身的考察，跳表中插入一个节点之前会选择一个随机化的层数，因为如果跳表的层数从下至上呈一定的比例关系，那么后期插入和删除的时候就又需要去维护这种比例关系，会使时间复杂度退化。所以跳表选择在插入节点的时候，选择一个随机化的层数。

但是生成的随机层数得遵循一个算法，使得生成小数值层数的概率很大，而生成大数值层数的概率很小，这个算法就是幂次定律。跳表在插入新节点之前，会利用这个幂次定律算法生成一个随机层数。

回答：跳表在插入新节点之前会计算一个随机的层高，具体来说，跳表的每一个节点一开始默认都是1层，然后每增加一层的概率都是 25%，Redis 7.0 最高为32层。

##### 跳表插入一条数据的平均时间复杂度是多少？

分析：这个问题同样是对跳表这种数据结构本身的考察，跳表实际上就是在一维链表上建立多层索引的二维链表，多出来的层数可以让跳表实现类似二分查找的算法。所以跳表的插入平均时间复杂度是O(logn)。

回答：跳跃表是一种支持多级索引的结构，查询效率可以媲美二分查找，它插入一条数据的平均时间复杂度是O(logN)。

##### 为什么跳表和HashTable要配合使用？

分析：使用了两种数据结构来实现 ZSet，为了让有序集合拥有这两种数据结构的优势。可以结合跳表和字典相较于各自的优势来进行回答。

回答：为了结合这两种数据结构各自的优势，当ZSet要根据成员来查找分值的时候，将使用字典来查找，时间复杂度为O(1)。而当ZSet要执行范围操作时，比如ZRANK、ZRANGE等命令时，将使用原本就有序的跳跃表来实现。

##### 为什么用跳表而不是 B+ 树？

分析：这个问题可以结合跳表和B+树的结构，以及B+树是为了解决什么问题而被作为MySQL的索引结构来回答。

B+树属于多路平衡查找树，由于是多叉树所以层数低，可以减少磁盘访问；而且数据只储存在叶子节点中，非叶子节点只储存索引值，使得一次IO读取的数据量很大。这样的设计使得 B+ 树非常适合作为磁盘数据库索引的结构。

而 Redis 是内存型数据库，要求的是高效率以及低内存，显然 Redis 对 B+ 树层数低的优点并不感冒，除此之外跳表占用内存更少、实现更加简单。而且跳表的插入性能更高，虽然两者的插入平均时间复杂度相当，但是跳表插入数据后只需要修改前进和后退指针即可，而B+树还需要维护树的平衡。

回答：B+ 树的数据都存在叶子节点中，而且它是多叉树，这两个特点使得它适合磁盘存储。而Redis是一个内存数据库，要求的是高效率以及低内存，跳表相较于 B+ 树占用内存更少、实现更加简单。而且跳表的插入性能更高，虽然两者的插入平均时间复杂度相当，但是跳表插入数据后只需要修改前进和后退指针即可，而 B+ 树还需要维护树的平衡。

##### 为什么用跳表而不是用红黑树？

分析：这个问题可以结合跳表和红黑树的结构、两者的内存占用、实现的难易程度来回答。

回答：

- 跳表的范围查询操作比红黑树简单，红黑树找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。而在跳表上进行范围查找就非常简单，只需要在找到小值之后，对第 1 层链表进行若干步的遍历就可以实现。
- 跳表的内存占用少于红黑树，跳表的调整只需要维护相邻节点，而红黑树调整的是一颗子树，更加耗费内存。
- 跳表的算法实现更加简单，红黑树的插入和删除操作可能引发子树的调整，逻辑复杂，而跳表的插入和删除只需要修改相邻节点的指针，操作简单又快速。

### Stream

#### 是什么？

Stream 是 Redis 5.0 版本新增加的操作对象。  Stream 可以看作一个拥有持久化能力的轻量级消息队列。

#### 适用场景

Stream可以作为轻量级队列，可以支持发布订阅场景。

#### 常用操作

流 ID 可以独一无二地标志一个Stream。

##### 写操作

`XADD key ID(流ID，可以用*自动生成) field string [field string ...]`

功能：向 Stream 中添加流数据，如果没有就新建一个Stream，返回值是一个流ID,如果输入时是自定义流ID，返回值与之相同，如果用 * 自动生成，返回的就是自动生成的流ID。

##### 读操作

`XLEN key`

功能：返回流数目。

`XRANGE key start end [COUNT count]`

功能：范围查询流信息。

##### 群组操作

除了常规的读写操作，Stream 还支持按群组操作，这里单独列出来。如果了解过 Kafka 的同学，应该知道 Kafka 也有消费者组的概念，虽然实现南辕北辙，但逻辑上很相似。

Stream 群组具备如下特性：

1. 群组里可以多个成员，成员名称由消费方自定义，组内唯一即可。
2. 同个群组共享消息，消息被其中一个消费者消费之后，其它消费者不会再重复消费。
3. 不在群组内的客户端，也可以通过XREAD命令来和群组一起消费，即群组和非群组可以混用，这个时候其实把不在群组的客户端理解为一个单独的群组。

`XGROUP CREATE key groupname id-or-$`

功能：创建一个新的消费群组，成功返回OK。

`XREADGROUP GROUP ${group_id} ${member_id} COUNT ${num} STREAMS ${stream_id}`

- group_id：群组id，也就是群组名字；
- member_id：群组成员id；
- num：消费几条数据；
- stream_id：stream流id。

功能：消费流中的数据。

### 其他操作对象

#### Bitmaps

Bitmaps 提供二进制位操作，可以设置位的值，可以获取位的值。举个例子，你需要存储一个 unit8 的二进制数据，并支持更改他的位值，比如原来这个uint8是 10000000，你想将其最后一位改为 1，变成 10000001，就可以用 Bitmaps 对象。

##### 应用场景

假设有1000个传感器，需要记录每小时这1000个传感器是否有上报，就可以用Bitmaps。

##### 基本操作

`SETBIT key offset value`

功能：设置位图对应位置为1，返回值为更新前的值。

`GETBIT key offset`

功能：设置位图对应位置为1，返回对应数据。

#### HyperLogLog

Redis 在 2.8.9 版本添加了 HyperLogLog 结构。Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的。在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。

#### Geospatial

在 Redis 的3.2版本中加入了地理空间 Geospatial 的功能，在特定的地理应用里能发挥作用。

### 对象过期时间

Redis 的过期时间是给一个 key，指定一个时间点，等达到这个时间，数据就被认为是过期数据，可以由 Redis 进行回收。

如果不是需要常驻的数据，设置过期时间，可以有效地节约内存。另外，有些场景功能也需要过期时间支持，比如缓存，如果存在时间过久，就可能导致和数据源数据差距过大，而设置过期时间，可以很方便的清楚缓存以便后续再次加载进去。又比如分布式锁，就是需要一定时间之后，数据自动消失，以实现最大占据时间的特性。

#### 怎么设置过期时间

如果是简单的字符串对象，可以使用如下语法：

- `SET key value EX seconds`：设置多少秒之后过期；
- `SET key value PX milliseconds`：设置多少毫秒之后过期；
- `TTL key`：查看还有多久过期。

更通用的过期命令是 EXPIRE，它可以对所有数据对象设置过期时间，EXPIRE 也分秒和毫秒：

- `EXPIRE key seconds`：设置一个key的过期时间，单位秒；
- `PEXPIRE key milliseconds`：设置一个key的过期时间，单位毫秒。

#### 键过期了多久删除

过期之后的键实际上不是立刻删除的，一般过期键清除策略有三种，分别是定时删除、定期删除和惰性删除。

- 定时删除，是在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作，定时删除对内存比较友好，但是对CPU不友好，如果某个时间段比较多的Key过期，可能会影响命令处理性能。
- 惰性删除，是指使用的时候，发现Key过期了，此时再进行删除，这个策略的思路是对应用而言，只要不访问，那么才用感知到过期，但是这样的代价就是如果某些Key一直不来访问，那本该过期的Key，就变成常驻的Key。这种策略对CPU最友好，对内存不太友好。
- 定期删除，每隔一段时间，程序就对数据库进行一次检查，每次删除一部分过期键，这属于一种渐进式兜底策略。

定时删除实现起来其实没有想象的容易，主要考虑是如果出现异常，有Key遗漏了怎么办，以及如果程序重启，原来的定时器就随重启消失了，那就需要在启动时对过期键进行一些操作，可能是重建定时器，这些都是额外的工作，而且引入了多余的复杂度。

从实际功能而言，其实并不需要那么实时，所以惰性删除是可以考虑的，但是出于应删尽删的考虑，要保证最终没有漏网之鱼，那就需要加上定期删除作为兜底。所以Redis过期键采用的是惰性删除+定期删除二者结合的方式进行删除的。

惰性删除不用再多说，Redis每次访问Key前都会进行检查，如已过期就删除。定期删除就需要关注两个问题：

1. 定期删除的频率

这个其实取决于 Redis 周期任务的执行频率，周期任务里面会做关闭过期客户端、删除过期Key的一系列任务，可以用 INFO 查看周期任务频率。

2. 每次删除的数量

每次检查的数量是写死在代码里的，每次20个，但是这里还会检查过期key数量占比，大于 25%，则再抽出20个来检查，重复流程，这里是一个循环。

Redis 为了保证定期删除不会出现循环过度，导致线程卡死现象，为此增加了定期删除循环流程的时间上限，默认不会超过 25ms。

### 对象引用计数

引用计数，顾名思义是记录某个了内存对象被引用了多少次，它是计算机里一种内存管理技术，通常而言，引用计数大于0，就代表这个对象还在被引用，引用计数等于0，说明这个对象已经没有被引用，可以对其进行释放。

#### Redis对象的引用计数

redisObject的结构定义，其中有个字段叫refcount，这个refcount就是Redis中的引用计数。

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

当 refcount 减少到0，就会触发对象的释放。Redis的引用计数，目前是为整数数据服务的。

Redis会在初始化服务器时，会创建10000个数值从0到9999的字符串对象。当服务器、新创建的键需要用到值为0到9999的字符串对象时，服务器就会使用这些共享对象，而不是新创建对象。

为什么只做0-9999的字符串对象池呢，关键因素有两点：

1. 0-9999的整数，被使用的几率是很大的，复用是有场景的。
2. 整数存储空间比较小，而每个redisObject内部结构至少占16字节，这比整数本身占据的空间还大，频繁分配整数是比较大的开销。
3. 要复用对象，就需要进行数值比较，而整数对象进行比较，成本最低，如果是其它字符串，需要遍历字符串所有字符，而其它如List、ZSet的对比成本就更高了。

#### 引用计数验证

我们可以使用`OBJECT REFCOUNT [arguments [arguments ...]]`命令查看Redis对象的引用计数，比如，我们设置num为100，再查看num的引用计数：

```bash
127.0.0.1:6379> SET num 100
OK
127.0.0.1:6379> OBJECT REFCOUNT num
(integer) 2147483647
```

这里看到num引用计数是2147483647，因为设置为100，是属于0到9999这个覆盖，所以引用计数不止是1，这个不奇怪，但是为什么会是2147483647这么大呢，实际上2147483647，就是int32的最大值，为了更加清晰，我们来看下源码，初始化服务器函数是initServer，它会调用createSharedObjects函数来创建共享对象，我们关注这个片段：

```c
for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
    shared.integers[j] =
        makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
    shared.integers[j]->encoding = OBJ_ENCODING_INT;
}
```

OBJ_SHARED_INTEGERS 定义是 10000，也就是为从 0 到 9999 的数字创建共享对象。makeObjectShared 中指定了 refCount 为 OBJ_SHARED_REFCOUNT。

```c
robj *makeObjectShared(robj *o) {
    serverAssert(o->refcount == 1);
    o->refcount = OBJ_SHARED_REFCOUNT;
    return o;
}
```

OBJ_SHARED_REFCOUNT是多少呢？

```c
#define OBJ_SHARED_REFCOUNT INT_MAX
```

是的，就是INT的最大值。所以0-9999这个范围的数字，都是共享对象，同时引用计数会保持在INT_MAX。

#### 引用计数这个字段就用在这？

这里看起来有一点不合理，为什么不直接共享 [0-9999] 的数据是不是可以呢，而需要多引入一个引用计数的字段。在 redisObject 头里面增加了一个 refCount，足足有4字节，在其它地方都是一点一点节约内存，这里加个字段，每个对象都多四字节。

这个是大家稍微了解Redis Object结构之后，可能会有的疑问，事实上，对于外部能操作的对象，refcount 还真没啥用，就 [0-9999] 的区间可以复用，而且这个数值还是统一在INT_MAX，不会更改，refcount 可以说没有意义。

但是，在Redis内部很多场景，其实是用 refcount 来进行引用计数的，最大的作用其实是最大的应用是传递参数，避免拷贝。

## Redis 是怎么运作的

### Redis 在内存中是怎么存储的

#### 数据库结构

redisDb代表Redis数据库结构，各种操作对象，就是存储在dict数据结构里。

```c
// from Redis 5.0.5

/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

这里我们重点关注dict结构，其它字段可以先忽略，它代表了我们存入的key-value存储，我们平常添加数据，就是往dict里添加。可以看到，dict其实就是我们前面介绍的Hash对象结构。

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

redisDb即数据库对象，指向了数据字典，字典里包含了我们平常存储的k-v数据，k是字符串对象，value支持任意Redis对象。



#### 过期键

Redis数据都可以设置过期键，这样到了一定的时间，这些对象就会自动过期并回收。那么过期键，又是存储在哪里的呢？过期键是存在expires字典上。

```c
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

注意，这里的dict中和expires中Key对象，实际都是存储的String对象指针，所以并不是会重复占用内容，Redis对内存的使用都是很珍惜的。

### Redis内存面试与分析

#### 面试题

##### SET a b，这个数据的存储结构是怎样的？

分析：考察对内存存储的理解，在Redis中，存储是一个字典结构。

回答：Redis中的存储是字典结构，SET a b之后，a会放在字典的对应偏移位置，b作为对应的value进行存储。

### Redis是单线程，还是多线程？

Redis是一个能高效处理请求的组件，一般而言，对这种组件，我们都需要了解其并发模型是怎样的，比如 Nginx 是多进程模型，Mysql 是多线程模型，那 Redis 又是什么呢？多线程模型吗？

先说结论，核心处理逻辑，Redis一直都是单线程的，其它辅助模块也会有一些多线程、多进程的功能，比如：复制模块用的多进程；

某些异步流程从 4.0 开始用的多线程，例如 UNLINK、FLUSHALL ASYNC、FLUSHDB ASYNC 等非阻塞的删除操作；

网络I/O解包从 6.0 开始用的是多线程；

但是这种分支模块，都只是辅助，最核心的还是处理架构，这块 Redis 始终是单线程的。

#### Redis 为何选择单线程？

Redis 的核心处理模块选择用单线程来实现，这可能会让人很疑惑，毕竟多线程可以充分利用多核的优势，而 Redis 官方的对于此的回答是：Redis 的定位，是内存k-v存储，是做短平快的热点数据处理，一般来说执行会很快，执行本身不应该成为瓶颈，而瓶颈通常在网络 I/O，处理逻辑多线程并不会有太大收益。

同时，Redis本身秉持简洁高效的理念，代码的简单性、可维护性是Redis一直以来的追求，引入多线程带来的复杂性远比想象的要大，而且多线程本身也会引入额外成本，分析如下：

1. 多线程引入的复杂性是极大的

首先，多线程引入之后，Redis原来的顺序执行特性就不复存在，为了支持事务的原子性、隔离性，Redis就不得不引入一些很复杂的实现；

其次，Redis的数据结构，可以说是极其高效，在单线程模式下做了很多特性的优化，如果引入多线程，那么所有底层数据结构都要改造为线程安全，这会是极其复杂的工作；

而且，多线程模式也使得程序调试更加复杂和麻烦，会带来额外的开发成本及运营成本，也更容易犯错。

所以，引入多线程，会带来很大的复杂度，对于追求简洁的Redis而言，这是一个需要非常谨慎的事，事实上，Redis 6.0 之后为 I/O 处理引入了多线程来提高性能，核心处理逻辑还是保留单线程，但是即使这样，6.0 之后的复杂性还是多了许多，更别说完全改成多线程处理了。

2. 多线程带来额外的成本

除了引入复杂度，多线程还会带来额外的成本。包括：

- 上下文切换成本，多线程调度需要切换线程上下文，这个操作先存储当前线程的本地数据、程序指针等，然后载入另一个线程数据，这种内核操作的成本不可忽视。
- 同步机制的开销，一些公共资源，在单线程模式下直接访问就行了，多线程需要通过加锁等方式去进行同步，这也是不可忽视的CPU开销。
- 一个线程本身也占据内存大小，对 Redis 这种内存数据库而言，内存非常珍贵，多线程本身带来的内存使用的成本也需要谨慎决策。

所以综合来看，多线程其实会带来非常多的成本，如果将处理模块改为多线程，即使在性能上，可能也很难有一个很高的预期，毕竟 Redis 单线程的处理，已经够快了。

### 单线程为什么能这么快

我们前面说到，Redis核心的请求处理是单线程，通常来说，单线程的处理能力要比多线程差很多，但是 Redis 却能使用单线程模型达到每秒数万级别的处理能力，这是为什么呢？其实，这是 Redis 多方面极致设计的一个综合结果。

几个关键点：

第一，Redis 的大部分操作在内存上完成，内存操作本身就特别快；

第二，Redis追求极致，选择了很多高效的数据结构，并做了非常多的优化，比如ziplist, hash，跳表，有时候一种对象底层有几种实现以应对不同场景；

第三，Redis 采用了多路复用机制，使其在网络 IO 操作中能并发处理大量的客户端请求，实现高吞吐量。

#### I/O 可能的潜在点

首先，我们知道 Redis 是完全在内存中处理数据，所以我们最应该考虑的瓶颈是 I/O，我们下面通过分析一次请求，来看一下，一个单线程在一次完整的处理中，会有哪些地方可能拖慢整个流程。

Redis 的服务端在启动的时候，已经 bind 了端口，并且用 listen 操作监听客户端请求，此时客户端就可以发起连接请求。

此时，客户端发起一次处理请求，比如，客户端发来一个GET请求，服务端需要哪些事情：

1. 客户端请求到来时候，使用accept建立连接；
2. 调用recv从套接字中读取请求；
3. 解析客户端发送请求，拿到参数；
4. 处理请求，这里是 Get，那么 Redis 就是通过 Key 获取对应的数据；
5. 最后将数据通过 send 发送给客户端。

我们要知道，套接字是默认阻塞模式的，这里阻塞可能会发生在两个地方。一个是 accept，比如 accept 建立时间过长，另一个是 recv 时客户端一直没有发送数据。此时，Redis 服务就会阻塞在那里。

Redis本身定位就是单线程，发生这种阻塞会将整个服务都卡住。所以不能让这两个操作阻塞，这里Redis将套接字设置为非阻塞式的，这样accept和recv都可以非阻塞调用。

非阻塞调用下，如果没数据，不会阻塞在那里，而是让你返回做其它事情。这样可以解决卡死的问题。但我们也需要一种机制，能回过头来看看这些操作是否已经就绪。

最简单的思路，我们可以通过一个循环来不断轮询，但这种方式显然低效。好在各个操作系统实现了一种机制，叫I/O多路复用。

什么叫 I/O 多路复用，简单理解来说，就是有 I/O 操作触发的时候，就会产生通知，收到通知，再去处理通知对应的事件，针对 I/O 多路复用，Redis 做了一层包装，叫 Reactor 模型。

如下图，本质就是监听各种事件，当事件发生时，将事件分发给不同的处理器。

![RedisIO多路复用监听](../picture/RedisIO多路复用监听.png)

这样就不会阻塞在某一个操作上，充分发挥性能，可以说 I/O 多路复用让 Redis 单线程也有了较大的并发度，注意这里是并发，而不是并行，在这种模式下，Redis 单核的性能可以说是被充分的利用了。

### Redis处理过程-源码解析

#### Redis的 AE 库

redis实现了一个AE库，它是“A simple Event drived programming library”的简写，翻译过来就是一个简单的事件驱动库，它会根据系统的内核版本自动选择select，epoll，kqueue等不同多路复用的方式。

由于生产环境通常是在linux上面，所以下面我们以常见的epoll为例，以下代码片段均来自自redis 5.0.5。

#### 处理流程

##### 监听端口，注册事件

main->initServer 中，使用 listenToPort 监听端口，创建监听套接字。

```c
listenToPort(server.port,&server.ipfd)  
```

接下来，还是在 initServer 函数中，会调用 aeCreateFileEvent 函数绑定对应的接收处理函数 acceptTcpHandler，绑定之后等到客户端连接请求发回来，就可以关联到这个处理函数。

```c
/* Create an event handler for accepting new connections in TCP and Unix
 * domain sockets. */
for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
        acceptTcpHandler,NULL) == AE_ERR)
        {
            serverPanic(
                "Unrecoverable error creating server.ipfd file event.");
        }
}
```

main 函数调用 initServer 之后，就开始在 aeMain 函数中开始一个 EvenLoop 事件循环，通过 epoll_wait() 的方式等待连接达到，此时暂时还没有已经建立的连接，所以没有读写事件的发生，另外计时器等事件也是注册在事件框架中，但这里我们主要还是关注 I/O 事件。

```c
// from Redis 5.0.5
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

##### 连接到达处理

当一个连接到达，事件循环就能收到这个信息，在监听套接字关联的 acceptTcpHandler 函数这时候就开始行动了，先生成一个客户端套接字，但这里也不是马上开始读客户端发送过来的数据，因为也没法确定此时一定能无阻塞地读取到完整的请求包，因此为它注册一个可读事件，仍然放到如上所述的同一个 epoll 实例中，让内核去通知我们该 fd 已经就绪了，这样就解决了 recv 阻塞等待的问题。其源码是从在回调函数中注册回调函数。

acceptTcpHandler -> acceptCommonHandler -> createClient 

createClient中关键片段：

```c
// from Redis 5.0.5

client *c = zmalloc(sizeof(client));

/* passing -1 as fd it is possible to create a non connected client.
 * This is useful since all the commands needs to be executed
 * in the context of a client. When commands are executed in other
 * contexts (for instance a Lua script) we need a non connected client. */
if (fd != -1) {
    anetNonBlock(NULL,fd);
    anetEnableTcpNoDelay(NULL,fd);
    if (server.tcpkeepalive)
        anetKeepAlive(NULL,fd,server.tcpkeepalive);
    if (aeCreateFileEvent(server.el,fd,AE_READABLE,
        readQueryFromClient, c) == AE_ERR)
    {
        close(fd);
        zfree(c);
        return NULL;
    }
}
```

可以看到，这里关键信息是创建了一个客户端对象，然后将客户端套接字设置为非阻塞，然后加入到事件循环里，这里的关联函数是 readQueryFromClient，这里大家应该已经能理解，等这个客户端套接字的读请求过来，就会关联到 readQueryFromClient 来处理。

##### 客户端数据处理

收到一个客户端发来的数据后，readQueryFromClient 就开始行动了，这里我们聚焦关注命令处理，忽略其它细节。

readQueryFromClient->processInputBuffer->processCommand，processInputBuffer 会解析命令，实际执行命令是在 processCommand 中：

第一步：使用lookupCommand，找到对应命令，

```c
/* Now lookup the command and check ASAP about trivial error conditions
 * such as wrong arity, bad command name and so forth. */
c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
```

第二步：做各项检查，比如检查，确认Redis可以执行本条命令。

第三步：真正执行命令，更改内存数据，下面是说如果客户端启动了事务，则将命令放入队列中，这里我们不过多关注。如果没有开启事务，则调用 call 方法更改内存。

```c
/* Exec the command */
if (c->flags & CLIENT_MULTI &&
    c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
    c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
{
    queueMultiCommand(c);
    addReply(c,shared.queued);
} else {
    call(c,CMD_CALL_FULL);
    c->woff = server.master_repl_offset;
    if (listLength(server.ready_keys))
        handleClientsBlockedOnKeys();
}
```

call 里面的实现关键是调用了 proc 方法，这个方法对应的就是命令执行函数。

```c
c->cmd->proc(c);
```

如果看完了这一系列操作，可能会比较奇怪，回包是在哪里进行的呢？实际上，执行完成之后，Redis 会将结果放入 client 结构的输出缓冲区中，这个逻辑是在命令执行函数中，比如GET 命令，它的 proc 其实就是：

```c
void getCommand(client *c) {
    getGenericCommand(c);
}
```

getGenericCommand 里面使用 addReply 将结果放入 client 的输出缓冲区。

```c
int getGenericCommand(client *c) {
    robj *o;

    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
        return C_OK;

    if (o->type != OBJ_STRING) {
        addReply(c,shared.wrongtypeerr);
        return C_ERR;
    } else {
        addReplyBulk(c,o);
        return C_OK;
    }
}
```

注意这里也没有直接 send() 到客户端，那么什么时候出发回包到客户端呢？

##### 给客户端回包

我们回到上面aeMain主循环的地方。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

在进入aeProcessEvents之前会调用eventLoop->beforesleep(eventLoop)，beforeSleep中有handleClientsWithPendingWrites，就是专门做回包的。

```c
/* This function is called just before entering the event loop, in the hope
 * we can just write the replies to the client output buffer without any
 * need to use a syscall in order to install the writable event handler,
 * get it called, and so forth. */
int handleClientsWithPendingWrites(void)
```

其具体工作就是将client结构输出缓冲区的内容，发送给对应的客户端socket。

### 多线程是怎么回事

#### Redis多线程模型

由于Redis从一开始就是基于单线程模型的，Redis 里所有的数据结构都是非线程安全的，规避了数据竞争问题，使得Redis对各种数据结构的操作非常简单、便捷。

Redis选择单线程的核心原因是Redis是都是内存操作，CPU处理都非常快，瓶颈更容易出现在I/O而不是CPU，所以选择了单线程模型。

随着时代的发展，很多业务的请求量都达到了一个曾经难以想象的高度，I/O 操作确实成为了瓶颈，而之前Redis处理流程中读取请求、发送回包都属于 I/O 操作，所以 Redis 引入了多线程，这里的多线程也不是说将整个处理逻辑都多线程化，如果这么做，需要对所有数据结构进行线程安全重构，这是巨大的成本，并且，也不见得能有多大提升，甚至可能因为损耗而降低，毕竟瓶颈大多时候都不在处理，瓶颈实际一般都在于网络 IO。

因为上述情况，Redis选择了引入多线程来处理网络I/O，仍然使用单线程框架来执行Redis命令，这样既保持了Redis核心的单线程处理架构，完全兼容以前的实现，又引入了多线程解决提升网络I/O的性能。

下面是 Redis 多线程的设计思路，如下图：

![Redis多线程设计思路1](../picture/Redis多线程设计思路1.png)

![Redis多线程设计思路2](../picture/Redis多线程设计思路2.png)

首先，看下什么时候启动多线程：

服务初始化的时候，会调用 initThreadedIO 来初始化多线程，根据 server.io_thread 来配置，如果为 1，表示只有一个主线程，那这里就不会再创建其它线程，如果大于1，并且不超过128，则会进入多线程模式，第一步就是会为多线程模式创建资源。

```c
/* Spawn and initialize the I/O threads. */
for (int i = 0; i < server.io_threads_num; i++) {
    /* Things we do for all the threads including the main thread. */
    io_threads_list[i] = listCreate();
    if (i == 0) continue; /* Thread 0 is the main thread. */

    /* Things we do only for the additional threads. */
    pthread_t tid;
    pthread_mutex_init(&io_threads_mutex[i],NULL);
    setIOPendingCount(i, 0);
    pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
    if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
        serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
        exit(1);
    }
    io_threads[i] = tid;
}
```

这里为所有线程在 io_threads_list 中创建对象，提前锁定了pthread_mutex_lock 互斥锁资源，pthread_create 就创建了多线程，子线程运行 IOThreadMain 操作，IOThreadMain 是一个无限循环函数，也就是说子线程会一直处于运行状态，直至进程结束。

注意，Redis 的多线程模式默认是关闭的，需要用户在 redis.conf 配置文件中开启：

```c
io-threads 4
io-threads-do-reads yes
```

#### 主线程视角

##### 事件循环

主线程和 6.0 版本之前一样，还是使用aeMain来处理事件循环。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```

当有客户端有连接过来，acceptTcpHandler 被调用，主线程使用 AE 的 API 将 readQueryFromClient 命令读取处理器绑定到新连接对应的文件描述符上，并初始化一个 client 绑定这个客户端连接。

当 client 的读写请求过来，就会调用 readQueryFromClient 这个方法，老版本是在 readQueryFromClient 函数中同步完成读取、解析、执行、将回包放入客户端输出缓冲区。但是多线程模式下，则只会调用 postponeClientRead 将 client 加入到 clients_pending_read 任务队列中去，后面主线程再分配 I/O 线程去读取客户端请求命令：

```c
void readQueryFromClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    int nread, big_arg = 0;
    size_t qblen, readlen;

    /* Check if we want to read from the client later when exiting from
     * the event loop. This is the case if threaded I/O is enabled. */
    if (postponeClientRead(c)) return;
    
    // 省略...
 }
```

```c
int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_BLOCKED)) &&
        io_threads_op == IO_THREADS_OP_IDLE)
    {
        listAddNodeHead(server.clients_pending_read,c);
        c->pending_read_list_node = listFirst(server.clients_pending_read);
        return 1;
    } else {
        return 0;
    }
}
```

放进队列之后，主线程会在事件循环 beforeSleep 函数中，调用 handleClientsWithPendingReadsUsingThreads，目的是通过  Round-Robin 轮询负载均衡策略把所有任务分配给 I/O 线程和主线程去读取并解析客户端命令。

handleClientsWithPendingReadsUsingThreads 步骤如下：

1. 分开clients给不同的线程；

```c
    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }
```

2. 激活其它线程，其它线程会将请求数据存入 client 的 querybuf 中，并解析第一个命令；

```c
 /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }
```

3. 主线程也去读自己负责的一部分

```c
/* Also use the main thread to process a slice of clients. */
listRewind(io_threads_list[0],&li);
while((ln = listNext(&li))) {
    client *c = listNodeValue(ln);
    readQueryFromClient(c->conn);
}
listEmpty(io_threads_list[0]);
```

4. 等待其它线程也完成读取

```c
/* Wait for all the other threads to end their work. */
while(1) {
    unsigned long pending = 0;
    for (int j = 1; j < server.io_threads_num; j++)
        pending += getIOPendingCount(j);
    if (pending == 0) break;
}
```

5. 执行上面读取完的所有命令，具体执行函数为processPendingCommandAndInputBuffer，在这个函数中，会先调用 processCommandAndResetClient 执行第一条已经解析好的命令，然后调用 processInputBuffer 解析并执行客户端连接的所有命令。

```c
int processPendingCommandAndInputBuffer(client *c) {
    if (c->flags & CLIENT_PENDING_COMMAND) {
        c->flags &= ~CLIENT_PENDING_COMMAND;
        if (processCommandAndResetClient(c) == C_ERR) {
            return C_ERR;
        }
    }

    /* Now process client if it has more data in it's buffer.
     *
     * Note: when a master client steps into this function,
     * it can always satisfy this condition, because its querbuf
     * contains data not applied. */
    if (c->querybuf && sdslen(c->querybuf) > 0) {
        return processInputBuffer(c);
    }
    return C_OK;
}
```

##### 回包

和单线程时候一样，多线程版本也是在 beforeSleep 来触发回包，不同的是多线程版本下的beforeSleep会使用多线程来进行回包，关键函数handleClientsWithPendingWritesUsingThreads，

```c
/* Handle writes with pending output buffers. */
handleClientsWithPendingWritesUsingThreads();
```

利用 Round-Robin 轮询负载均衡策略，把等待回包队列中的连接均匀地分配给 I/O 线程各自的本地 FIFO 任务队列 和主线程自己，主线程轮询等待所有 I/O 线程完成回包任务。

```c
/* Wait for all the other threads to end their work. */
while(1) {
    unsigned long pending = 0;
    for (int j = 1; j < server.io_threads_num; j++)
        pending += getIOPendingCount(j);
    if (pending == 0) break;
}
```

#### 子线程视角

##### 等待事件

1. 自旋1000000次等待资源
2. 获取 io_threads_mutex 互斥锁，一开始会阻塞在这里。

此时，就需要等待主线程来触发资源。

##### 处理事件

I/O 线程在会感知到主线程通知的事件变化，会遍历自己的本地任务队列 io_threads_list，取出client 来执行。

可以看到，相比于主线程，子线程的逻辑还是比较清晰简单的。子线程会根据事件类型来决定是回包 writeToClient，还是读取数据 readQueryFromClient。

如果是writeToClient，就通过socket把缓冲区数据发送给客户端，如果是读取操作，就通过socket读取客户端命令，存入读取缓存，这里最终只会解析到第一条命令，然后就结束，不会去执行命令。

在全部任务执行完之后把自己的原子计数器置 0，以告知主线程自己已经完成了工作。

```c
while(1) {
    /* Wait for start */
    for (int j = 0; j < 1000000; j++) {
        if (getIOPendingCount(id) != 0) break;
    }

    /* Give the main thread a chance to stop this thread. */
    if (getIOPendingCount(id) == 0) {
        pthread_mutex_lock(&io_threads_mutex[id]);
        pthread_mutex_unlock(&io_threads_mutex[id]);
        continue;
    }

    serverAssert(getIOPendingCount(id) != 0);

    /* Process: note that the main thread will never touch our list
     * before we drop the pending count to 0. */
    listIter li;
    listNode *ln;
    listRewind(io_threads_list[id],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        if (io_threads_op == IO_THREADS_OP_WRITE) {
            writeToClient(c,0);
        } else if (io_threads_op == IO_THREADS_OP_READ) {
            readQueryFromClient(c->conn);
        } else {
            serverPanic("io_threads_op value is unknown");
        }
    }
    listEmpty(io_threads_list[id]);
    setIOPendingCount(id, 0);
}
```

#### Redis多线程性能

多线程性能比单线程提高了一倍左右。

### Redis 命令执行面试与分析

#### 面试题

##### Redis是单线程还是多线程？

分析：

回答：

##### Redis单线程性能如何？

分析：

回答：

##### 为什么单线程还能这么快？

分析：

回答：

##### Redis6.0之后引入了多线程，你知道为什么吗？

分析：

回答：

##### Redis6.0的多线程是默认开启的吗？

分析：

回答：

##### Redis6.0的多线程主要负责命令执行的哪一块？

分析：

回答：

### 内存满了怎么办？

#### Redis可以存多少数据？

使用 maxmemory 配置，默认是被注释掉的，也就是默认值是 0，在 32 位操作系统中，maxmemory 的默认值是 3G，因为 32 位的机器最大只支持 4GB 的内存，而系统本身就需要一定的内存资源来支持运行，默认 3G 相对合理。

现在的机器基本都是64位机器，64位机器不会限制内存的使用。

我们也可以主动配置 maxmemory，当Redis存储超过这个配置值，则触发Redis内存淘汰。

#### 内存淘汰

Redis支持多久淘汰策略，大的方向有两个，一个是 noeviction，默认就是这种策略，此时如果内存达到 maxmemory，则写入操作会失败，但不会淘汰已有数据。

第二个是多种淘汰策略，主要支持LRU，LFU，RANDOM, TTL 这几个方式：

- lru：根据LRU（Least recently used，最近最少使用）算法尝试回收最长时间未使用的；
- lfu：根据LFU（Least Frequently Use）驱逐最不常用的键，lfu是在4.0引入的；
- random：回收随机的键使得新添加的数据有空间存放；
- ttl：回收在过期集合的键，并且优先回收存活时间（TTL）较短的键，使得新添加的数据有空间存放。

这四种策略，可以选择volatile，也就是设置了过期时间的Key，或者是allkeys，即全部的Key，所以一共有8种淘汰方式。

#### 选择哪种淘汰算法

淘汰算法根据业务需求决定，比如，如果数据非常重要，不能丢失，那就选择不淘汰，这种情况会导致写入失败，有完善的告警机制配合人工介入。

如果是缓存场景，业务方一般使用 LRU/LFU 这种灵活的淘汰策略。

#### 淘汰时机

什么时候触发淘汰呢？实际上，每次运行读写命令的时候，都会调用 processCommand 函数，processCommand 中又会调用 freeMemoryIfNeeded，这时候就会尝试去释放一定内存，策略就按我们上述配置的策略。

### 内存淘汰算法-LRU

LRU一直以来都是一个非常流行的资源淘汰算法，为了减少内存消耗，Redis 使用了近似 LRU ，下面我们来介绍一个LRU算法是什么，为什么Redis不用标准的LRU，Redis的近似LRU是怎样的？

#### LRU 算法是什么？

最近最久未使用，即记录每个 Key 的最近访问时间，维护一个访问时间的数据。

#### Redis 用 LRU 算法会有什么问题？

如果为所有数据维护一个顺序列表，实际就是做一个双向链表，但是如果Redis数据稍微多些，这个链表就是巨大的成本，对于Redis而言，内存是最宝贵的，所以Redis选择了近似LRU算法。

#### Redis 近似 LRU 算法

在 LRU 模式，redisObject 对象中 lru 字段存储的是 key 被访问时 Redis 的时钟server.lruclock，当 key 被访问的时候，Redis 会更新这个 key 的 redisObject 的 lru 字段。

注意，Redis 为了保证核心单线程服务性能，缓存了 Unix 操作系统时钟，默认每毫秒更新一次，缓存的值是 Unix 时间戳取模 2^24。

近似 LRU 算法在现有数据结构的基础上采用随机采样的方式来淘汰元素，当内存不足时，就执行一次近似 LRU 算法。

具体步骤是随机采样 n 个 key，这个采样个数默认为 5，然后根据时间戳淘汰掉最旧的那个 key，如果淘汰后内存还是不足，就继续随机采样来淘汰。

##### 采样范围

Redis可以选择范围策略，有两种：

1. allkeys，所有key中随机采样。
2. volatile，从有过期时间的 key 随机采样。

分别对应 allkeys-lru，volatitle-lru。

##### 淘汰池优化

近似LRU，优点在于节约了内存，但是它的缺点就是不是一个完整的LRU，随机采样得到的结果，其实不是全局真正的最久未访问，在数据量大的情况，尤其是这样。

Redis 3.0 对近似 LRU 算法进行了一些优化。新算法会维护一个大小为 16 的候选池，池中的数据根据访问时间进行排序。第一次随机选取的 key 都会放入池中，然后淘汰掉最久未访问的，比如第一次选了 5 个，淘汰了 1 个，剩下 4 个继续留在池子里。

随后每次随机选取的 key 只有活性比池子里活性最小的 key 还小时才会放入池中，当池子装满了，如果有新的 key 需要放入，则将池中活性最大的 key 移除。

通过池子存储，在池子里的数据会越来越接近真实的活性最低，所以其表现也会非常接近真正的LRU。

### 内存淘汰算法-LFU

LFU 淘汰算法，即 Least Frequently Used，最不频繁淘汰算法，顾名思义，优先淘汰活跃最低，使用频率最低的。

#### 源码分析

我们复习下redisObject，其结构如下：

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

如果使用 LRU，那么 redisObject 中 lru 字段，就是用来存储最近访问时间的，这个字段长度是 LRU_BITS，这个值一直都是24位。

如果是 LFU，因为 LRU、LFU 是不会同时开启的，所以两者可以说是互斥，基于这个情况，加上节约内存的考虑，Redis 在 LFU 策略下复用 lru 字段，还是用它来表示 LFU 的信息，不过将 24 拆解，高 16bit 存储 ldt(Last Decrement Time)，低 8bit 存储 logc(Logistic Counter)。高的16位保存了上次访问时间戳，因为少了8位，所以LFU下时间精度是1分钟。后8位存储的是一个访问次数。

一个 Key 是否活跃，就是这两个字段综合决定的。如果上一次访问时间很久，那么访问次数就会衰减。而本身访问，会增加访问次数。

```c
/* ===================== Creation and parsing of objects ==================== */

robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```

其中，LFU_INIT_VAL 初始值为 5。

```c
/* Update LFU when an object is accessed.
 * Firstly, decrement the counter if the decrement time is reached.
 * Then logarithmically increment the counter, and update the access time. */
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```

1. 计算次数衰减。因为无论是多快，相对于上次访问，一定有时间间隔，根据间隔，来计算应该减少的次数。使用的函数就是 LFUDecrAndReturn。
2. 一定概率增加访问次数。这里可能会疑问，为什么是一定概率增加访问次数，次数不足5次，那一定会增加，如果大于5次，小于255次，会一定概率加1，原来的次数越大，越困难。除了原来的次数影响之外，还有一个 lfu-log-factor 参数可以被设置的。也就是说，可以通过 lfu-log-factor 参数来调节难度，这个越大，难度也越大，如果为0，那么每次必然+1，很快就能255。
3. 更新。当前时间更新到高16位，次数更新到低8位。

### 内存淘汰面试与分析

#### 面试题

##### Redis有几种内存回收策略？

分析：

回答：大的方向有两种，一种是 noeviction 不进行淘汰，此时如果内存达到 maxmemory，写入操作失败，但是不会淘汰已有数据。另一种是多种淘汰策略，lru（最近最少使用，算法尝试回收最长时间未使用的）、lfu（驱逐最不常用的（次数）），random（回收随机的键）、ttl（回收过期集合的键，优先回收存活时间（ttl）较短的键）。

##### 内存回收是什么时候发起？

分析：

回答：实际上，每次进行读写的时候，都会调用 processCommand 函数，processCommand 函数又会调用 freememoryIfNeeded，这个时候就会尝试去释放一定的内存。

##### 介绍下Redis LRU回收算法

分析：

回答：LRU 最近最久未使用，即记录每个key的最近访问时间，维护一个访问时间的数据。

##### Redis LRU算法是标准的吗？为什么不用标准的？

分析：

回答：采用近似 LRU，如果为所有的数据维护一个顺序列表，实际上就是做一个双向链表，但是如果 redis 数据过大这个链表的成本就巨大，对于 redis 来说内存是宝贵的，所以 redis 采用采样的方式来做，也就是近似 LRU 算法。

首先。redisObject 对象中的 lru 字段存储的是 key 被访问时 redis 的时钟，当 key 被访问的时候，redis 会更新这个 key 的 redisObject 的 lru 字段。近似 lru 算法在原有的基础上采用随机采样的方式来淘汰元素，当内存不足时，执行一次近似 lru 算法。具体步骤为随机采样 n 个 key（默认为5）然后根据时间戳淘汰掉最旧的那个 key，有两种采样范围，allkeys，所有的key随机采样，allkeys-lru；volatile 从有过期时间的 key 随机采样，volatile-lru。

##### 什么是LFU算法，为什么Redis要引入LFU算法？

分析：

回答：LFU，最不频繁淘汰算法，优先淘汰活跃度最低，使用频率最低的。对于 lru，只是记录了访问时间，而我们会有根据访问频率淘汰的时候，所以就出现了 lfu。redis 的 lru 字段一共是 24 位，高十六位存储上一次访问的时间戳，低八位存储访问次数，因此，一个key是否活跃是由两个字段来决定的。

## Redis 数据丢失怎么办

### 持久化介绍

Redis 是跑在内存里的，当程序重启或者服务崩溃，数据就会丢失，如果业务场景希望重启之后数据还在，就需要持久化，即把数据保存到可永久保存的存储设备中。

#### 持久化方式

Redis 提供两种方式来持久化：

1. RDB（Redis Database），记录 Redis 某个时刻的全部数据，这种方式本质就是数据快照，直接保存二进制数据到磁盘，后续通过加载 RDB 文件恢复数据。
2. AOF（Append Only File），记录执行的每条命令，重启之后通过重放命令来恢复数据，AOF本质是记录操作日志，后续通过日志重放恢复数据。

RDB是快照恢复，AOF是日志恢复，区别如下：

- 体积方面：相同数据量下，RDB体积更小，因为RDB是记录的二进制紧凑型数据。
- 恢复速度：RDB是数据快照，可以直接加载，而AOF文件恢复，相当于重放情况，RDB显然会更快。
- 数据完整性：AOF记录了每条日志，RDB是间隔一段时间记录一次，用AOF恢复数据通常会更为完整。

#### 用RDB好，还是AOF好

还是根据业务场景来：

- 如果业务本身只是缓存数据且并不是一个海量访问，可以不用开持久化。
- 如果对数据非常重视，可以同时开启 RDB 和 AOF，同时开启的情况下，优先使用 AOF，逻辑是 AOF 的内容通常会比 RDB 更全。
- 如果可以接受丢几分钟级别的数据，那么建议只开 RDB。单独开 AOF，Redis 官方不建议，因为如果决定要走数据备份，那么镜像保存始终是数据库领域非常行之有效的解决方案，所以在配置中 RDB 是默认打开的，而 AOF 不是。

### RDB详解

#### 怎么开启RDB持久化

我们先从实践上入手，看一下怎么开启RDB持久化。

首先，我们打开redis配置文件， mac 下是在 /usr/local/etc/redis.conf，在其中搜索可以发现如下配置：

```c
save 900 1
save 300 10
save 60 10000
```

这里的配置语法是 save interval num，表示每间隔 interval 秒，至少有 num 条写数据操作，写数据操作指增加、删除及更新，就会激活 RDB 持久化。

上面有 3 条 save 配置，他们的意思分别是：每900s，有1条写数据操作；每300s，有10条写数据操作；每60s，有10000写数据操作。

他们之间是并集关系，即只要满足其中一个条件，就达到了RDB持久化的条件。

这三条配置不是我们增加，是默认就存在的，这就是说redis默认已经开启了RDB持久化。

#### RDB文件存哪里

下面的参数决定了文件会存到哪里：

```c
 # The filename where to dump the DB
 dbfilename dump.rdb

 # The working directory.
 dir /Users/niuniumart/code/redis
```

#### 什么时候进行持久化？

Redis持久化会在下面三种情况下进行：

1. 主动执行命令 save。执行了 save 命令，就会在主线程生成 RDB 文件，由于和执行操作命令在同一个线程，所以如果写入 RDB 文件的时间太长，会阻塞主线程，这个命令慎用。
2. 主动执行命令 bgsave。和save不同，会创建一个子进程来生成 RDB 文件，这样可以避免主线程的阻塞。
3. 达到持久化配置阈值。上面有提到，Redis可以配置持久化策略，达到策略就会触发持久化，这里的持久化使用的命令是 bgsave，可以看到 Redis 比较推荐的方式也是后台执行持久化，尽可能减少对主流程影响。达到阈值之后，是由周期函数触发持久化。
4. 在程序正常关闭的时候执行。在关闭时，Redis会启动一次持久化，以记录更全的数据。所以正常关闭丢失的数据不会太多，崩溃才会。

#### RDB具体做了什么

我们这里聚焦达到RDB持久化策略时，是如何进行持久化的，我们先看一下Redis输出：

```bash
62581:M 01 Jan 2023 09:04:26.279 * 1 changes in 3600 seconds. Saving...
62581:M 01 Jan 2023 09:04:26.281 * Background saving started by pid 50162
50162:C 01 Jan 2023 09:04:26.283 * DB saved on disk
62581:M 01 Jan 2023 09:04:26.383 * Background saving terminated with success
```

从输出能看出，RDB确实是通过子进程来进行的，具体做了什么呢？下面是官方的答案：

```bash
How it works
Whenever Redis needs to dump the dataset to disk, this is what happens:
1.Redis forks. We now have a child and a parent process.
2.The child starts to write the dataset to a temporary RDB file.
3.When the child is done writing the new RDB file, it replaces the old one.
This method allows Redis to benefit from copy-on-write semantics.
```

从整体上，是做了以下事项：

1. Fork出一个子进程来专门做 RDB 持久化；
2. 子进程写数据到临时的 RDB 文件；
3. 写完之后，用新 RDB 文件替换旧的 RDB 文件。

下面还有一句：This method allows Redis to benefit from copy-on-write semantics. 就是说这种方式让 Redis 从写时复制技术受益，Redis 官方文档基本没废话，这句话看似无关轻重，实际上说明了：执行 RDB 持久化过程中，Redis 依然可以继续处理操作命令的，也就是数据是能被修改的，这就是通过写时复制技术实现的。

具体而言，fork 创建子进程之后，通过写时复制技术，子进程和父进程是共享同一片内存数据的，因为创建子进程的时候，会复制父进程的页表，但是页表指向的物理内存还是一个。只有在发生修改内存数据的情况时，物理内存才会被复制一份。

就是这样，Redis 使用 bgsave 对当前内存中的所有数据做快照，这个操作是由 bgsave 子进程在后台完成的，执行时不会阻塞父进程中的主线程，这就使得主线程同时可以修改数据。

这样的目的是为了减少创建子进程时的性能损耗，从而加快创建子进程的速度，毕竟创建子进程的过程中，是会阻塞主线程的。

可以看到，复制期间，读数据互不影响，如果有写操作发生，则复制一份给子进程，主进程再修改原来的数据。

### AOF详解

#### AOF写入流程

执行请求时，每条日志都会写入到AOF。这不免会让人担心，是否会影响 Redis 的执行性能，答案是肯定的，多了一步操作，或多或少都会带来些损耗，但是Redis实际是提供了不同的策略来选择不同程度的损耗。Redis 提供了 3 种刷盘策略：

1. appendfsync always，每次请求都刷入AOF，用官方的话说，非常慢，非常安全。
2. appendfsync everysec，每秒刷一次盘，用官方的话来说就是足够快了，但是在崩溃场景下你可能会丢失1秒的数据。
3. appendfsync no，不主动刷盘，让操作系统自己刷，一般情况Linux会每30秒刷一次盘，这种策略下，可以说对性能的影响最小，但是如果发生崩溃，可能会丢失相对比较多的数据。

Redis建议是方案二，也就是每秒刷一次盘，这种方式下速度也足够快了，同时崩溃时损失的数据只有1s，这在大多数场景都是可以接受的。

当然了，我们要根据实际业务来选择，比如就是做简单的缓存，并且不存在什么超级热点缓存，那么丢失30秒也不是什么大事，这时候如果追求性能的机制，可以选择方案3。

方案一说实话倒是很少有场景会使用，因为Redis本身是无法做到完全不丢数据，Redis的定位就不是完全可靠，通常也就没必要损耗大量性能去追求立刻刷盘。

#### 写入AOF细节

写入AOF，其实是分了好几步来的。

第一步，其实是将数据写入 AOF 缓存中，这个缓存名字是 aof_buf，其实就是一个sds数据，

```
sds aof_buf;      /* AOF buffer, written before entering the event loop */
```

第二步，aof_buf 对应数据刷入磁盘缓冲区，什么时候做这个事情呢？事实上，Redis源码中一共有4个时机，会调用一个叫 flushAppendOnlyFile 的函数，这个函数会使用 write 函数来将数据写入操作系统缓冲区：

1. 处理完事件处理后，等待下一次事件到来之前，也就是 beforeSleep 中。
2. 周期函数serverCron中。
3. 服务器退出之前的准备工作时。
4. 通过配置指令关闭AOF功能时。

注意，如果是 appendfsync everysec 策略，那么在上一次 fsync 还没完成之前，并且时长不超过两秒时，就本次 flushAppendOnlyFile 就会放弃，也就是说先不写入 aof_buf。

第三步，刷盘，即调用系统的flush函数，刷盘其实还是在 flushAppendOnlyFile 函数中，是在write 之后，但是不一定调用了 flushAppendOnlyFile，flush 就一定会被调用，这里其实是支持一个刷盘时机的配置，这一步受刷盘策略影响是最深的，如下面代码所示，如果是 appendfsync always 策略，那么就立刻调用 redis_fsync 刷盘，如果是 AOF_FSYNC_EVERYSEC 策略，满足条件后会用 aof_background_fsync 使用后台线程异步刷盘。

```c
/* Perform the fsync if needed. */
if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
    /* redis_fsync is defined as fdatasync() for Linux in order to avoid
     * flushing metadata. */
    latencyStartMonitor(latency);
    redis_fsync(server.aof_fd); /* Let's try to get this data on the disk */
    latencyEndMonitor(latency);
    latencyAddSampleIfNeeded("aof-fsync-always",latency);
    server.aof_last_fsync = server.unixtime;
} else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
            server.unixtime > server.aof_last_fsync)) {
    if (!sync_in_progress) aof_background_fsync(server.aof_fd);
    server.aof_last_fsync = server.unixtime;
}
```

讲到这里，我们应该可以很清晰画出一个相对宏观的完整写入示意图：

![RedisAOF写入流程](../picture/RedisAOF写入流程.png)

#### AOF 重写

AOF 是不断写入的，这很容易带来一个疑问，如此下去 AOF 不是会不断膨胀吗？

针对这个问题，Redis 采用了重写的方式来解决：

Redis 可以在 AOF 文件体积变得过大时，自动地在后台 Fork 一个子进程，专门对 AOF 进行重写。说白了，就是针对相同 Key 的操作，进行合并，比如同一个 Key 的 set 操作，那就是后面覆盖前面。

在重写过程中，Redis 不但将新的操作记录在原有的 AOF 缓冲区，而且还会记录在 AOF 重写缓冲区。一旦新 AOF 文件创建完毕，Redis 就会将重写缓冲区内容，追加到新的 AOF 文件，再用新 AOF 文件替换原来的 AOF 文件。

这里可能会问，AOF达到多大会重写，实际上，这也是配置决定，默认如下，同时满足这两个条件则重写。

```c
# 相比上次重写时候数据增长100%
auto-aof-rewrite-percentage 100
# 超过
auto-aof-rewrite-min-size 64mb
```

也就是说，超过64M的情况下，相比上次重写时的数据大一倍，则触发重启，很明显，最后实际上还是在周期函数来检查和触发的。

### 混合持久化

混合部署实际发生在AOF重写阶段，将当前状态保存为RDB二进制内容，写入AOF文件，再将重写缓冲区的内容追加到AOF文件。

此时的AOF文件，就不再单纯的是日志数据，而是二进制数据+日志数据的混合体，所以叫混合持久化。

#### 混合持久化解决什么问题？

混合持久化是是发生在原有的 AOF 流程，如果从这个视角来看，其实本质还是 AOF，只是重写时使用了 RDB 进行了优化。

如果从更高层面来看，这个确实是属于 RDB 和 AOF 优点的融合，属于一种折中方案吧，变成了可读性降低的 AOF 或者说性能变差的 RDB？

如果是考虑到对 Redis 核心处理性能的影响，那还是需要用 RDB，如果是为了相对更可靠的数据记录，也就是尽可能丢失更少的数据，那还是得用 AOF，同时，如果对可读性没有太大执念，那进一步开启混合持久化，是一个很好的选择，毕竟其实生产上，关注 AOF 可读性的情况实际比较少。

#### 混合持久化开启之后，服务启动时如何加载持久化数据

如果同时启用了 AOF 和 RDB，Redis 重新启动时，会使用 AOF 文件来重建数据集，因为通常来说， AOF 的数据会更完整，那这里其实是一样的，混合持久化还是属于 AOF，所以如果有混合持久化，那肯定是优先使用混合持久化的数据。至于如何区别是否有AOF混合持久化的数据，可以使用文件开头是否为“REDIS”来判断。

完整的具体加载流程如下：

![Redis混合持久化数据载入流程](../picture/Redis混合持久化数据载入流程.png)

### MP-AOF方案

Redis AOF 文件随着不断膨胀，会通过重写机制来减少 AOF 文件大小。功能上，倒是没啥问题，但是从性能和方案上来说，AOF 重写功能都显得比较粗糙，这也是正常的，持久化作为一个相对分支的模块，在前期没有花费太多精力去优化完全可以理解，随着 Redis 不断完善，AOF 重写功能终于在 7.0 得到了一个很大程度的优化。

#### 原有AOF重写方案弊端

我们详细分析下原有的方案：

![Redis原有AOF重写方案](../picture/Redis原有AOF重写方案.png)

优势：

1. 方案容易想到，前期快速实现。

劣势：

1. 占用主进程CPU时间，在重写期间额外向 aof_rewrite_buf 写数据；通过管道向子进程发送 aof_rewrite_buf 中的数据；以及子进程结束后将剩余 aof_rewrite_buf 写入临时文件，这都是需要消耗 CPU 时间的，而 Redis 核心执行是单线程的，CPU资源的损耗会带来响应速度的降低。
2. 额外的内存开销，在 AOF 重写期间，主进程会将 fork 之后的数据变化同时写进 aof_buf 和 aof_rewrite_buf 中，这部分内容是重复的。另外在子进程读取时候，会开辟一个读取内存空间，下面代码展示了读取时候会开辟内存缓冲区。

```c
// from Redis 5.0.5
/* This function is called by the child rewriting the AOF file to read
 * the difference accumulated from the parent into a buffer, that is
 * concatenated at the end of the rewrite. */
ssize_t aofReadDiffFromParent(void) {
    char buf[65536]; /* Default pipe buffer size on most Linux systems. */
    ssize_t nread, total = 0;

    while ((nread =
            read(server.aof_pipe_read_data_from_parent,buf,sizeof(buf))) > 0) {
        server.aof_child_diff = sdscatlen(server.aof_child_diff,buf,nread);
        total += nread;
    }
    return total;
}
```

3. 额外磁盘开销，主进程除了会将执行过的写命令写到 aof_buf 之外，还会写一份到 aof_rewrite_buf 中。aof_buf 最终写入老文件，aof_rewrite_buf 最终写入新AOF文件，这相当于是两次磁盘消耗，而数据明明是一样的。

为了解决上述问题，我们需要先认清为什么问题的本质是有两个不太合理的设计：

1. 同时向 aof_buf 和 aof_rewrite_buf 写入；
2. 父子进程传输文件数据。

这两个是问题的本质，理清楚了本质，优化起来并不难，Redis 在 7.0 版本引入了 MP-AOF 方案。

#### MP-AOF 方案概述

MP-AOF，全称 Multi Part AOF，也就是多部件 AOF，通俗点说就是原来是一个 AOF 文件，现在变成了多个 AOF 文件的配合。MP-AOF 文件核心有两个Part组成：

1. BASE AOF文件，基础AOF文件，记录了基本的命令。
2. INCR AOF文件，记录了在重写过程中新增的操作命令。

这两个部件，就可以一起组成完整的AOF操作记录，原来是一个 AOF 文件，里面包含了操作的命令记录，MP-AOF 则是提出了 BASE AOF 和 INCR AOF 两种文件的组合，一个 BASE AOF 结合  INCR AOF，一起来表示所有的操作命令。

MP-AOF 的流程如下：

![RedisMP-AOF流程](../picture/RedisMP-AOF流程.png)

可以看到，现在重写阶段，在主进程中写 aof_buf 即可，aof_buf 的数据最终落入新打开的INCR AOF 文件，至于子进程，就安心将原来的 BASE AOF 重写并生成新的一个BASE AOF 文件。新生成的 BASE AO F和新打开的 INCR AOF 就代表了当前时刻 Redis 的全部数据。

写完之后呢，也不是覆盖原有的 BASE AOF 和 INCR AOF，而只需要更新 manifest 文件，manifest 是文件清单，描述了当前有效的 BASE AOF、INCR AOF 是哪个，这里可以简单理解为 manifest 就是一个配置。

那之前旧的 BASE AOF、INCR AOF 文件怎么办呢？Redis是将它们标记为 HISTORY AOF，这些 HISTORY AOF 会被 Redis 异步删除掉。

通过上述讲解，我们可以看到：

1. 我们在 AOFRW 期间不再需要 aof_rewrite_buf；
2. 我们也不再需要父子进程通过管道传输操作数据；

如此一来，CPU、内存、磁盘的性能损耗会降低很多，甚至代码复杂度也会降低，整体更为简单清晰，这也符合 Redis 的简洁的理念。

### 持久化面试题

#### 面试题

##### RDB 和 AOF 本质区别是什么？

分析：如果是讲区别可以从文件类型，文件恢复速度，安全性来进行回答，但本质区别就是RDB 是使用快照进行持久化，AOF是日志。

其他可以列举的区别点 都是能从快照与日志中的对比找出的：

- 文件类型：RDB 生成的是二进制文件（快照），AOF 生成的是文本文件（追加日志）。
- 安全性：缓存宕机时，RDB 容易丢失较多的数据，AOF 根据策略决定（默认的 everysec 可以保证最多有一秒的丢失）。
- 文件恢复速度：由于 RDB 是二进制文件，所以恢复速度也比 AOF 更快。
- 操作的开销：每一次 RDB 保存都是一次全量保存，操作比较重，通常设置至少 5 分钟保存一次数据。而 AOF 的刷盘是一次追加操作，操作比较轻，通常设置策略为每一秒进行一次刷盘。

回答：本质区别就是 RDB 是保存快照进行持久化，而 AOF 则是 追加日志文件进行持久化。

##### RDB 为什么不每一秒做一次快照？

分析：

回答：虽然可以通过 fork 出的子进程来做全量快照，但是如果每一秒一次，会导致很大的恶性能开销，可能这一秒的快照都没完成，下一秒又 fork 出一个子进程来做快照，而 fork 子进程是会导致住县城阻塞的，所以 RDB 的快照触发间隔是比较难确定的。

##### 如果RDB和AOF只能选一种，你选哪个？

分析：其实就是让我们在性能和 可靠之间做一个选择。这里不提供给大家一个所谓完全正确的答案，主要是让大家对两种持久化方式有更深刻的理解。

回答：如果我们能接受分钟级别的数据丢失，可以考虑选择RDB；如果需要尽量保证数据安全，可以考虑混合持久化；如果只用AOF，那么优先选择 everysec 策略进行刷盘（在可靠和性能之间有一个平衡）。Redis 不推荐单独开 AOF，如果需要持久化，快照始终是一个好办法，从默认配置也能看出来 RDB 是默认的，AOF 不是。

##### RDB 触发条件是什么？触发时机又是什么？

分析：从源码来看，RDB持久化函数整体上在这几个地方会被使用：

- Redis Shutdown（Redis关闭之前进行一次持久化）；
- 客户端发送save命令；
- 客户端发送bgsave命令；
- 每一次事件循环ServerCron检查是否需要bgsave（也就是判断我们的RDB配置）；
- 主从全量复制发送RDB文件（也进行一次RDB持久化）；
- 客户端执行数据库清空命令FLUSHALL。

虽然 RDB 的触发条件有很多，但实际执行过程很简单，而区别点在于是子进程来执行还是主进程执行这个流程：

- 打开一个临时的RDB文件（c语言库函数fopen）；
- 将执行命令这一时刻数据库数据写入到 IO缓冲区（c语言库函数fwrite）；
- 执行fflush（将 IO 缓冲区里的数据刷新到内核缓冲区）；
- 执行fsync（可以将内核缓冲区里的数据刷到磁盘）；
- 执行fclose （关闭这个临时文件）；
- 修改临时文件名字，并让后台线程 BIO_LAZY_FREE 去删除旧的RDB（到此，一次RDB过程结束）。

简单来说 RDB 的流程就是触发RDB持久化时，让主进程或子进程（区分条件）来将这一时刻的数据库数据写到一个新的 RDB 文件中。

回答：根据分析理清楚RDB流程是怎样的，触发条件大致有哪些，酌情回答(无须细究源码，大概了解就行)。

##### AOF 触发条件是什么？触发时机又是什么？

分析：首先一定要弄清楚AOF的流程是怎样的，整个 AOF 的流程可以分为三个过程：命令追加（append到aod_buf）、文件写入（write进内核缓冲区）、文件同步（fsync让内核缓冲区数据进入磁盘文件 ）。

从源码来看，AOF刷盘函数在三个地方会被触发：

- Redis Shutdown的时候；
- 每一次事件循环 钩子函数 beforeSleep()；
- 每一次事件循环的时间事件对应的handler——serverCron()。

回答：根据分析 理清楚AOF流程是怎样的，不同appendfsync策略是如何执行的，酌情回答。

##### RDB 对主流程有什么影响？

分析：主要从RDB的整个流程来寻找一些明显或潜在的风险。

回答：

- 当执行Save命令的时候，由主进程进行 RDB快照保存，会阻塞主进程。
- 当执行BgSave或是配置自动触发时，由fork出的子进程来进行RDB快照保存。
	- 如果数据量比较大的时候，会导致fork子进程 这个操作比较耗时，从而阻塞主进程。
	- 由于采用了 写时复制技术，如果在进行 RDB 快照保存的时候，有大量的写入操作执行，会导致主进程多拷贝一份数据给子进程，消耗大量额外的内存。

##### AOF 对主流程有什么影响？

分析：

- 明显的影响：使用 AOF 持久化，如果我们选择 always 的策略，每当 Redis 命令执行之后，需要由主进程进行 write + fsync 的操作，如果数据过大的话，主进程就会花费较大的时间 用于写 AOF 日志，对后续请求影响较大（阻塞知道写 AOF 完成，才能响应客户端执行成功）
- 潜在的影响：
  - 如果我们选择 everysecond 策略，虽然 fsync 是交给后台线程 BIO_AOF_FSYNC 来完成，但是主进程还需要进行 write 操作，如果后台线程上一轮 fsync 没有完成，那么主进程进 write 的时候仍然会阻塞（因为 write 作用在一个正在 fsync 的 fd 上，会阻塞）。
  - AOF 重写是由 fork 出的子进程进行的，类似于上面提到的风险，fork 子进程这个操作有可能阻塞主进程。

回答：

- 当 appendfsync 使用 always，如果AOF写入日志压力过大会导致主进程无法继续处理其他请求。
- 当 appendfsync 使用 everysec，如果后台线程上一轮的 fsync 没有完成，会导致我们本轮主线程执行 write 被阻塞（直到fsync完成）。
- 当 AOF 重写发生时，如果数据量比较大，会导致 fork 子进程 这个操作比较耗时，从而阻塞主进程。

##### AOF 混合持久化方案是什么？

分析：需要记住的是，AOF混合持久化，就是在AOF重写的基础上做了一些改动。

回答：

- AOF 混合持久化会使用 RDB 持久化函数将内存数据写入到新的 AOF 文件中 （数据格式也是 RDB 格式）。
- 而重写期间新的写入命令追加到新的 AOF 文件仍然是 AOF 格式。
- 此时新的 AOF 文件就是由 RDB 格式和 AOF 格式组成的日志文件。

##### AOF 混合持久化有什么优缺点？

分析：

回答：

优点：重写之后的 AOF 文件体积更小，并且含有 RDB 格式（二进制）的数据，能加快恢复的速度。

缺点：含有一部分 RDB 格式数据，降低了可读性（对于人来说）。

##### 简单描述AOF重写流程

分析： 记住简单流程：

- 可以将 AOF 重写流程 记为 ”一次拷贝，两处日志“；
- 一次拷贝：重写发生时，主进程会 fork 出一个子进程，让子进程将这些 Redis 数据写入重写日志；
- 两处日志：重写发生时，我们需要注意  AOF 缓冲和 AOF 重写缓冲；当重写时，有新的写入命令执行，会由主进程分别写入 AOF 缓冲 和 AOF 重写缓冲；AOF 缓冲用于保证此时发生宕机，原来的 AOF 日志也是完整的，可用于恢复。AOF 重写缓冲用于保证新的 AOF 文件也不会丢失最新的写入操作。
- 额外补充：在重写时 AOF 重写缓冲会通过管道传送给子进程，再由子进程刷入新的 AOF 日志（此时，AOF 重写完成）。

回答：我将AOF重写流程理解成 “一次拷贝，两处日志”：

- 一次拷贝：重写发生时，主进程会 fork 出一个子进程，被将 Redis 内存拷贝给子进程，让子进程将这些内存写入重写日志。
- 两处日志：当重写时，有新的写入命令执行，会由主进程分别写入 AOF 缓冲 和 AOF 重写缓冲；AOF 缓冲用于保证 此时发生宕机，原来的 AOF 日志也是完整，可用于恢复。AOF 重写缓冲用于保证新的 AOF 文件也不会丢失最新的写入操作。

##### AOF 重写你觉得有什么不足之处么？

分析：我们可以从上题的 “一次拷贝，两处日志”来进行分析：

- 一次拷贝：使用子进程来重写日志，没有问题。
- 两处日志：重写期间 新的写命令，会将数据写入到两份日志（AOF 缓冲和 AOF 重写缓冲）中，这是额外的 CPU 和内存开销 ；重写时会将 AOF 缓冲和 AOF 重写缓冲分别写入到 旧日志和新日志中，这是额外的磁盘开销。
- 除了清楚 AOF 的不足之处，如果还知道改进方案，那么会令人刮目相看，Redis 7.0 就做了新的改进。

回答：

不足之处：

- 额外的CPU开销：在重写时，主进程需要将新的写入数据 写入到 AOF重写缓冲(aof_rewrite_buf)；主进程 还需要通过管道向子进程发送AOF重写缓冲的数据;  子进程还需要将这些数据 写入到新的AOF日志中。
- 额外的内存开销：在重写时，AOF缓冲 和 AOF重写缓冲 中的数据都是一样的（浪费了一份）。
- 额外的磁盘开销：在重写时，AOF缓冲需要刷入 旧的AOF日志，AOF重写缓冲也需要刷入 到新的AOF日志，导致在重写时磁盘多占一份数据。

改进之处：

- 在 Redis 7.0 版本，对 AOF 重新作出了优化，提出了 MP-AOF 方案，原来的 AOF 重写缓冲被移除，AOF 日志也分成了 Base AOF 日志 ，Incr AOF 日志。MP-AOF：Multi Part AOF = one BASE AOF + many INCR AOFs。
- Base AOF 日志记录重写之前的命令；Incr AOF 日志记录重写时新的写入命令（正常 AOF 刷盘的时候写的是 Incr AOF）。
- 当重写发生时，主进程 fork 出一个 子进程，对 AOF 日志进行重写（将当前内存数据写入到新的 Base AOF 日志）；如果此时有新的写入命令，会由主进程写入到 aof_buf，再将缓冲数据刷入新的 Incr AOF 日志。这样新的 Incr AOF日志 + 新的 Base AOF 日志就构成了完整的新的 AOF 日志。
- 子进程重写结束时，主进程会负责更新 manifest 文件，将新生成的 BASE AOF 和 INCR AOF 信息加进清单，并将之前的 BASE AOF 和 INCR AOF 标记为 HISTORY。mianfest 用于追踪管理AOF文件。
- 这些 HISTORY 文件默认会被 Redis 异步删除（unlink），一旦 manifest 文件更新完成，就代表着整个 AOFRW 流程结束。

## Redis 版本特性

只需要记住 6.0 版本引入了多线程。

# Redis 学习指引

## 数据结构

### Redis 常见数据类型和应用场景

#### String

String 是最基本的 key-value 结构，key 是唯一标识，value 是具体的值，value其实不仅是字符串， 也可以是数字（整数或浮点数），value 最多可以容纳的数据长度是 512M。

##### 内部实现

String 类型的底层的数据结构实现主要是 int 和 SDS（简单动态字符串）。

SDS 和我们认识的 C 字符串不太一样，之所以没有使用 C 语言的字符串表示，因为 SDS 相比于 C 的原生字符串：

- SDS 不仅可以保存文本数据，还可以保存二进制数据。因为 SDS 使用 len 属性的值而不是空字符来判断字符串是否结束，并且 SDS 的所有 API 都会以处理二进制的方式来处理 SDS 存放在 buf[] 数组里的数据。所以 SDS 不光能存放文本数据，而且能保存图片、音频、视频、压缩文件这样的二进制数据。
- SDS 获取字符串长度的时间复杂度是 O(1)。因为 C 语言的字符串并不记录自身长度，所以获取长度的复杂度为 O(n)；而 SDS 结构里用 len 属性记录了字符串长度，所以复杂度为 O(1)。
- Redis 的 SDS API 是安全的，拼接字符串不会造成缓冲区溢出。因为 SDS 在拼接字符串之前会检查 SDS 空间是否满足要求，如果空间不够会自动扩容，所以不会导致缓冲区溢出的问题。

字符串对象的内部编码（encoding）有 3 种 ：int、raw和 embstr。

如果一个字符串对象保存的是整数值，并且这个整数值可以用 long 类型来表示，那么字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面（将void * 转换成 long），并将字符串对象的编码设置为 int。

如果字符串对象保存的是一个字符串，并且这个字符申的长度小于等于 32 字节（redis 2.+版本），那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串，并将对象的编码设置为embstr， embstr编码是专门用于保存短字符串的一种优化编码方式。

如果字符串对象保存的是一个字符串，并且这个字符串的长度大于 32 字节（redis 2.+版本），那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串，并将对象的编码设置为raw。

注意，embstr 编码和 raw 编码的边界在 redis 不同版本中是不一样的：

- redis 2.+ 是 32 字节
- redis 3.0-4.0 是 39 字节
- redis 5.0 是 44 字节

embstr 和 raw 编码都会使用SDS来保存值，但不同之处在于 embstr 会通过一次内存分配函数来分配一块连续的内存空间来保存 redisObject 和 SDS，而 raw 编码会通过调用两次内存分配函数来分别分配两块空间来保存 redisObject 和 SDS。Redis 这样做会有很多好处：

- embstr 编码将创建字符串对象所需的内存分配次数从 raw 编码的两次降低为一次；
- 释放 embstr 编码的字符串对象同样只需要调用一次内存释放函数；
- 因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面可以更好的利用 CPU 缓存提升性能。

但是 embstr 也有缺点的：

如果字符串的长度增加需要重新分配内存时，整个 redisObject 和 sds 都需要重新分配空间，所以 embstr 编码的字符串对象实际上是只读的，redis 没有为 embstr 编码的字符串对象编写任何相应的修改程序。当我们对 embstr 编码的字符串对象执行任何修改命令（例如append）时，程序会先将对象的编码从 embstr 转换成 raw，然后再执行修改命令。

##### 应用场景

###### 缓存对象

使用 String 来缓存对象有两种方式：

- 直接缓存整个对象的 JSON，命令例子： `SET user:1 '{"name":"xiaolin", "age":18}'`。
- 采用将 key 进行分离为 user:ID: 属性，采用 MSET 存储，用 MGET 获取各属性值，命令例子： `MSET user:1:name xiaolin user:1:age 18 user:2:name xiaomei user:2:age 20`。

###### 常规计数

因为 Redis 处理命令是单线程，所以执行命令的过程是原子的。因此 String 数据类型适合计数场景，比如计算访问次数、点赞、转发、库存数量等等。

###### 分布式锁

SET 命令有个 NX 参数可以实现「key不存在才插入」，可以用它来实现分布式锁：

- 如果 key 不存在，则显示插入成功，可以用来表示加锁成功；
- 如果 key 存在，则会显示插入失败，可以用来表示加锁失败。

一般而言，还会对分布式锁加上过期时间，分布式锁的命令如下：

```c
SET lock_key unique_value NX PX 10000
```

- lock_key 就是 key 键；
- unique_value 是客户端生成的唯一的标识；
- NX 代表只在 lock_key 不存在时，才对 lock_key 进行设置操作；
- PX 10000 表示设置 lock_key 的过期时间为 10s，这是为了避免客户端发生异常而无法释放锁。

而解锁的过程就是将 lock_key 键删除，但不能乱删，要保证执行操作的客户端就是加锁的客户端。所以，解锁的时候，我们要先判断锁的 unique_value 是否为加锁客户端，是的话，才将 lock_key 键删除。

可以看到，解锁是有两个操作，这时就需要 Lua 脚本来保证解锁的原子性，因为 Redis 在执行 Lua 脚本时，可以以原子性的方式执行，保证了锁释放操作的原子性。

```lua
// 释放锁时，先比较 unique_value 是否相等，避免锁的误释放
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这样一来，就通过使用 SET 命令和 Lua 脚本在 Redis 单节点上完成了分布式锁的加锁和解锁。

###### 共享 Session 信息

通常我们在开发后台管理系统时，会使用 Session 来保存用户的会话(登录)状态，这些 Session 信息会被保存在服务器端，但这只适用于单系统应用，如果是分布式系统此模式将不再适用。

例如用户一的 Session 信息被存储在服务器一，但第二次访问时用户一被分配到服务器二，这个时候服务器并没有用户一的 Session 信息，就会出现需要重复登录的问题，问题在于分布式系统每次会把请求随机分配到不同的服务器。

因此，我们需要借助 Redis 对这些 Session 信息进行统一的存储和管理，这样无论请求发送到那台服务器，服务器都会去同一个 Redis 获取相关的 Session 信息，这样就解决了分布式系统下 Session 存储的问题。

#### List

List 列表是简单的字符串列表，按照插入顺序排序，可以从头部或尾部向 List 列表添加元素。列表的最大长度为 2^32 - 1，也即每个列表支持超过 40 亿个元素。

##### 内部实现

List 类型的底层数据结构是由双向链表或压缩列表实现的：

- 如果列表的元素个数小于 512 个（默认值，可由 list-max-ziplist-entries 配置），列表每个元素的值都小于 64 字节（默认值，可由 list-max-ziplist-value 配置），Redis 会使用压缩列表作为 List 类型的底层数据结构；
- 如果列表的元素不满足上面的条件，Redis 会使用双向链表作为 List 类型的底层数据结构；

但是在 Redis 3.2 版本之后，List 数据类型底层数据结构就只由 quicklist 实现了，替代了双向链表和压缩列表。

##### 应用场景

###### 消息队列

消息队列在存取消息时，必须要满足三个需求，分别是消息保序、处理重复的消息和保证消息可靠性。

Redis 的 List 和 Stream 两种数据类型，就可以满足消息队列的这三个需求。这里先了解基于 List 实现消息队列的方法。

1. 如何满足消息保序需求？

List 本身就是按先进先出的顺序对数据进行存取的，所以，如果使用 List 作为消息队列保存消息的话，就已经能满足消息保序的需求了。

List 可以使用 LPUSH + RPOP （或者反过来，RPUSH+LPOP）命令实现消息队列。

- 生产者使用`LPUSH key value[value...]`将消息插入到队列的头部，如果 key 不存在则会创建一个空的队列再插入消息。

- 消费者使用`RPOP key`依次读取队列的消息，先进先出。

不过，在消费者读取数据时，有一个潜在的性能风险点。

在生产者往 List 中写入数据时，List 并不会主动地通知消费者有新消息写入，如果消费者想要及时处理消息，就需要在程序中不停地调用 RPOP 命令（比如使用一个while(1)循环）。如果有新消息写入，RPOP命令就会返回结果，否则，RPOP命令返回空值，再继续循环。

所以，即使没有新消息写入List，消费者也要不停地调用 RPOP 命令，这就会导致消费者程序的 CPU 一直消耗在执行 RPOP 命令上，带来不必要的性能损失。

为了解决这个问题，Redis提供了 BRPOP 命令。BRPOP 命令也称为阻塞式读取，客户端在没有读到队列数据时，自动阻塞，直到有新的数据写入队列，再开始读取新数据。和消费者程序自己不停地调用RPOP命令相比，这种方式能节省CPU开销。

2. 如何处理重复的消息？

消费者要实现重复消息的判断，需要 2 个方面的要求：

- 每个消息都有一个全局的 ID。
- 消费者要记录已经处理过的消息的 ID。当收到一条消息后，消费者程序就可以对比收到的消息 ID 和记录的已处理过的消息 ID，来判断当前收到的消息有没有经过处理。如果已经处理过，那么，消费者程序就不再进行处理了。

但是 List 并不会为每个消息生成 ID 号，所以我们需要自行为每个消息生成一个全局唯一ID，生成之后，我们在用 LPUSH 命令把消息插入 List 时，需要在消息中包含这个全局唯一 ID。

3. 如何保证消息可靠性？

当消费者程序从 List 中读取一条消息后，List 就不会再留存这条消息了。所以，如果消费者程序在处理消息的过程出现了故障或宕机，就会导致消息没有处理完成，那么，消费者程序再次启动后，就没法再次从 List 中读取消息了。

为了留存消息，List 类型提供了 BRPOPLPUSH 命令，这个命令的作用是让消费者程序从一个 List 中读取消息，同时，Redis 会把这个消息再插入到另一个 List（可以叫作备份 List）留存。

这样一来，如果消费者程序读了消息但没能正常处理，等它重启后，就可以从备份 List 中重新读取消息并进行处理了。

好了，到这里可以知道基于 List 类型的消息队列，满足消息队列的三大需求（消息保序、处理重复的消息和保证消息可靠性）。

- 消息保序：使用 LPUSH + RPOP；
- 阻塞读取：使用 BRPOP；
- 重复消息处理：生产者自行实现全局唯一 ID；
- 消息的可靠性：使用 BRPOPLPUSH。

> List 作为消息队列有什么缺陷？

List 不支持多个消费者消费同一条消息，因为一旦消费者拉取一条消息后，这条消息就从 List 中删除了，无法被其它消费者再次消费。

要实现一条消息可以被多个消费者消费，那么就要将多个消费者组成一个消费组，使得多个消费者可以消费同一条消息，但是 List 类型并不支持消费组的实现。

这就要说起 Redis 从 5.0 版本开始提供的 Stream 数据类型了，Stream 同样能够满足消息队列的三大需求，而且它还支持「消费组」形式的消息读取。

#### Hash

Hash 是一个键值对（key - value）集合，其中 value 的形式如： `value=[{field1，value1}，...{fieldN，valueN}]`。Hash 特别适合用于存储对象。

##### 内部实现

Hash 类型的底层数据结构是由压缩列表或哈希表实现的：

- 如果哈希类型元素个数小于 512 个（默认值，可由 hash-max-ziplist-entries 配置），所有值小于 64 字节（默认值，可由 hash-max-ziplist-value 配置）的话，Redis 会使用压缩列表作为 Hash 类型的底层数据结构；
- 如果哈希类型元素不满足上面条件，Redis 会使用哈希表作为 Hash 类型的 底层数据结构。

在 Redis 7.0 中，压缩列表数据结构已经废弃了，交由 listpack 数据结构来实现了。

##### 应用场景

###### 缓存对象

Hash 类型的 （key，field， value） 的结构与对象的（对象id， 属性， 值）的结构相似，也可以用来存储对象。

###### 购物车

以用户 id 为 key，商品 id 为 field，商品数量为 value，恰好构成了购物车的3个要素。

![RedisHash应用场景购物车](../picture/RedisHash应用场景购物车.png)

涉及的命令如下：

- 添加商品：`HSET cart:{用户id} {商品id} 1`
- 添加数量：`HINCRBY cart:{用户id} {商品id} 1`
- 商品总数：`HLEN cart:{用户id}`
- 删除商品：`HDEL cart:{用户id} {商品id}`
- 获取购物车所有商品：`HGETALL cart:{用户id}`

当前仅仅是将商品ID存储到了Redis 中，在回显商品具体信息的时候，还需要拿着商品 id 查询一次数据库，获取完整的商品的信息。

#### Set

Set 类型是一个无序并唯一的键值集合，它的存储顺序不会按照插入的先后顺序进行存储。

一个集合最多可以存储 2^32-1 个元素。概念和数学中个的集合基本类似，可以交集，并集，差集等等，所以 Set 类型除了支持集合内的增删改查，同时还支持多个集合取交集、并集、差集。

Set 类型和 List 类型的区别如下：

- List 可以存储重复元素，Set 只能存储非重复元素；
- List 是按照元素的先后顺序存储元素的，而 Set 则是无序方式存储元素的。

##### 内部实现

Set 类型的底层数据结构是由哈希表或整数集合实现的：

- 如果集合中的元素都是整数且元素个数小于 512 （默认值，set-maxintset-entries 配置）个，Redis 会使用整数集合作为 Set 类型的底层数据结构；
- 如果集合中的元素不满足上面条件，则 Redis 使用哈希表作为 Set 类型的底层数据结构。

##### 应用场景

集合的主要几个特性，无序、不可重复、支持并交差等操作。

因此 Set 类型比较适合用来数据去重和保障数据的唯一性，还可以用来统计多个集合的交集、错集和并集等，当我们存储的数据是无序并且需要去重的情况下，比较适合使用集合类型进行存储。

但是要提醒你一下，这里有一个潜在的风险。Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞。

在主从集群中，为了避免主库因为 Set 做聚合计算（交集、差集、并集）时导致主库被阻塞，我们可以选择一个从库完成聚合统计，或者把数据返回给客户端，由客户端来完成聚合统计。

###### 点赞

Set 类型可以保证一个用户只能点一个赞，这里举例子一个场景，key 是文章id，value 是用户id。`uid:1` 、`uid:2`、`uid:3`三个用户分别对`article:1`文章点赞了。

```bash
# uid:1 用户对文章 article:1 点赞
> SADD article:1 uid:1
(integer) 1
# uid:2 用户对文章 article:1 点赞
> SADD article:1 uid:2
(integer) 1
# uid:3 用户对文章 article:1 点赞
> SADD article:1 uid:3
(integer) 1
```

`uid:1` 取消了对 `article:1` 文章点赞。

```bash
> SREM article:1 uid:1
(integer) 1
```

获取`article:1`文章所有点赞用户 :

```bash
> SMEMBERS article:1
1) "uid:3"
2) "uid:2"
```

获取`article:1`文章的点赞用户数量：

```bash
> SCARD article:1
(integer) 2
```

判断用户`uid:1`是否对文章`article:1`点赞了：

```bash
> SISMEMBER article:1 uid:1
(integer) 0  # 返回0说明没点赞，返回1则说明点赞了
```

###### 共同关注

Set 类型支持交集运算，所以可以用来计算共同关注的好友、公众号等。key 可以是用户 id，value 则是已关注的公众号的 id。

`uid:1`用户关注公众号 id 为 5、6、7、8、9，`uid:2` 用户关注公众号 id 为 7、8、9、10、11。

```bash
# uid:1 用户关注公众号 id 为 5、6、7、8、9
> SADD uid:1 5 6 7 8 9
(integer) 5
# uid:2  用户关注公众号 id 为 7、8、9、10、11
> SADD uid:2 7 8 9 10 11
(integer) 5
```

`uid:1`和`uid:2`共同关注的公众号：

```bash
# 获取共同关注
> SINTER uid:1 uid:2
1) "7"
2) "8"
3) "9"
```

给`uid:2`推荐`uid:1`关注的公众号：

```bash
> SDIFF uid:1 uid:2
1) "5"
2) "6"
```

验证某个公众号是否同时被`uid:1`或`uid:2`关注:

```bash
> SISMEMBER uid:1 5
(integer) 1 # 返回0，说明关注了
> SISMEMBER uid:2 5
(integer) 0 # 返回0，说明没关注
```

###### 抽奖活动

存储某活动中中奖的用户名 ，Set 类型因为有去重功能，可以保证同一个用户不会中奖两次。key 为抽奖活动名，value为员工名称，把所有员工名称放入抽奖箱 ：

```bash
>SADD lucky Tom Jerry John Sean Marry Lindy Sary Mark
(integer) 5
```

如果允许重复中奖，可以使用 SRANDMEMBER 命令。

```bash
# 抽取 1 个一等奖：
> SRANDMEMBER lucky 1
1) "Tom"
# 抽取 2 个二等奖：
> SRANDMEMBER lucky 2
1) "Mark"
2) "Jerry"
# 抽取 3 个三等奖：
> SRANDMEMBER lucky 3
1) "Sary"
2) "Tom"
3) "Jerry"
```

如果不允许重复中奖，可以使用 SPOP 命令。

```bash
# 抽取一等奖1个
> SPOP lucky 1
1) "Sary"
# 抽取二等奖2个
> SPOP lucky 2
1) "Jerry"
2) "Mark"
# 抽取三等奖3个
> SPOP lucky 3
1) "John"
2) "Sean"
3) "Lindy"
```

#### Zset

Zset 类型（有序集合类型）相比于 Set 类型多了一个排序属性 score（分值），对于有序集合 ZSet 来说，每个存储元素相当于有两个值组成的，一个是有序集合的元素值，一个是排序值。有序集合保留了集合不能有重复成员的特性（分值可以重复），但不同的是，有序集合中的元素可以排序。

##### 内部实现

Zset 类型的底层数据结构是由压缩列表或跳表实现的：

- 如果有序集合的元素个数小于 128 个，并且每个元素的值小于 64 字节时，Redis 会使用压缩列表作为 Zset 类型的底层数据结构；
- 如果有序集合的元素不满足上面的条件，Redis 会使用跳表作为 Zset 类型的底层数据结构；

在 Redis 7.0 中，压缩列表数据结构已经废弃了，交由 listpack 数据结构来实现了。

##### 应用场景

Zset 类型（Sorted Set，有序集合） 可以根据元素的权重来排序，我们可以自己来决定每个元素的权重值。比如说，我们可以根据元素插入 Sorted Set 的时间确定权重值，先插入的元素权重小，后插入的元素权重大。

在面对需要展示最新列表、排行榜等场景时，如果数据更新频繁或者需要分页显示，可以优先考虑使用 Sorted Set。

###### 排行榜

有序集合比较典型的使用场景就是排行榜。例如学生成绩的排名榜、游戏积分排行榜、视频播放排名、电商系统中商品的销量排名等。

我们以博文点赞排名为例，小林发表了五篇博文，分别获得赞为 200、40、100、50、150。

```bash
# arcticle:1 文章获得了200个赞
> ZADD user:xiaolin:ranking 200 arcticle:1
(integer) 1
# arcticle:2 文章获得了40个赞
> ZADD user:xiaolin:ranking 40 arcticle:2
(integer) 1
# arcticle:3 文章获得了100个赞
> ZADD user:xiaolin:ranking 100 arcticle:3
(integer) 1
# arcticle:4 文章获得了50个赞
> ZADD user:xiaolin:ranking 50 arcticle:4
(integer) 1
# arcticle:5 文章获得了150个赞
> ZADD user:xiaolin:ranking 150 arcticle:5
(integer) 1
```

文章`arcticle:4`新增一个赞，可以使用 ZINCRBY 命令（为有序集合key中元素member的分值加上increment）：

```bash
> ZINCRBY user:xiaolin:ranking 1 arcticle:4
"51"
```

查看某篇文章的赞数，可以使用 ZSCORE 命令（返回有序集合key中元素个数）：

```bash
> ZSCORE user:xiaolin:ranking arcticle:4
"50"
```

获取小林文章赞数最多的 3 篇文章，可以使用 ZREVRANGE 命令（倒序获取有序集合 key 从start 下标到 stop 下标的元素）：

```bash
# WITHSCORES 表示把 score 也显示出来
> ZREVRANGE user:xiaolin:ranking 0 2 WITHSCORES
1) "arcticle:1"
2) "200"
3) "arcticle:5"
4) "150"
5) "arcticle:3"
6) "100"
```

获取小林 100 赞到 200 赞的文章，可以使用 ZRANGEBYSCORE 命令（返回有序集合中指定分数区间内的成员，分数由低到高排序）：

```bash
> ZRANGEBYSCORE user:xiaolin:ranking 100 200 WITHSCORES
1) "arcticle:3"
2) "100"
3) "arcticle:5"
4) "150"
5) "arcticle:1"
6) "200"
```

###### 电话、姓名排序

使用有序集合的 ZRANGEBYLEX 或 ZREVRANGEBYLEX 可以帮助我们实现电话号码或姓名的排序，我们以 ZRANGEBYLEX （返回指定成员区间内的成员，按 key 正序排列，分数必须相同）为例。

注意：不要在分数不一致的 SortSet 集合中去使用 ZRANGEBYLEX和 ZREVRANGEBYLEX 指令，因为获取的结果会不准确。

1. 电话排序

我们可以将电话号码存储到 SortSet 中，然后根据需要来获取号段：

```bash
> ZADD phone 0 13100111100 0 13110114300 0 13132110901 
(integer) 3
> ZADD phone 0 13200111100 0 13210414300 0 13252110901 
(integer) 3
> ZADD phone 0 13300111100 0 13310414300 0 13352110901 
(integer) 3
```

获取所有号码：

```bash
> ZRANGEBYLEX phone - +
1) "13100111100"
2) "13110114300"
3) "13132110901"
4) "13200111100"
5) "13210414300"
6) "13252110901"
7) "13300111100"
8) "13310414300"
9) "13352110901"
```

获取 132 号段的号码：

```bash
> ZRANGEBYLEX phone [132 (133
1) "13200111100"
2) "13210414300"
3) "13252110901"
```

获取132、133号段的号码：

```bash
> ZRANGEBYLEX phone [132 (134
1) "13200111100"
2) "13210414300"
3) "13252110901"
4) "13300111100"
5) "13310414300"
6) "13352110901"
```

2. 姓名排序

```bash
> zadd names 0 Toumas 0 Jake 0 Bluetuo 0 Gaodeng 0 Aimini 0 Aidehua 
(integer) 6
```

获取所有人的名字：

```bash
> ZRANGEBYLEX names - +
1) "Aidehua"
2) "Aimini"
3) "Bluetuo"
4) "Gaodeng"
5) "Jake"
6) "Toumas"
```

获取名字中大写字母A开头的所有人：

```bash
> ZRANGEBYLEX names [A (B
1) "Aidehua"
2) "Aimini"
```

获取名字中大写字母 C 到 Z 的所有人：

```bash
> ZRANGEBYLEX names [C [Z
1) "Gaodeng"
2) "Jake"
3) "Toumas"
```

#### BitMap

Bitmap，即位图，是一串连续的二进制数组（0和1），可以通过偏移量（offset）定位元素。BitMap 通过最小的单位 bit 来进行 0|1 的设置，表示某个元素的值或者状态，时间复杂度为O(1)。

由于 bit 是计算机中最小的单位，使用它进行储存将非常节省空间，特别适合一些数据量大且使用二值统计的场景。

##### 内部实现

Bitmap 本身是用 String 类型作为底层数据结构实现的一种统计二值状态的数据类型。

String 类型是会保存为二进制的字节数组，所以，Redis 就把字节数组的每个 bit 位利用起来，用来表示一个元素的二值状态，你可以把 Bitmap 看作是一个 bit 数组。

##### 应用场景

Bitmap 类型非常适合二值状态统计的场景，这里的二值状态就是指集合元素的取值就只有 0 和 1 两种，在记录海量数据时，Bitmap 能够有效地节省内存空间。

###### 签到统计

在签到打卡的场景中，我们只用记录签到（1）或未签到（0），所以它就是非常典型的二值状态。

签到统计时，每个用户一天的签到用 1 个 bit 位就能表示，一个月（假设是 31 天）的签到情况用 31 个 bit 位就可以，而一年的签到也只需要用 365 个 bit 位，根本不用太复杂的集合类型。

假设我们要统计 ID 100 的用户在 2022 年 6 月份的签到情况，就可以按照下面的步骤进行操作。

第一步，执行下面的命令，记录该用户 6 月 3 号已签到。

```bash
SETBIT uid:sign:100:202206 2 1
```

第二步，检查该用户 6 月 3 日是否签到。

```bash
GETBIT uid:sign:100:202206 2 
```

第三步，统计该用户在 6 月份的签到次数。

```bash
BITCOUNT uid:sign:100:202206
```

这样，我们就知道该用户在 6 月份的签到情况了。

> 如何统计这个月首次打卡时间呢？

Redis 提供了`BITPOS key bitValue [start] [end]`指令，返回数据表示 Bitmap 中第一个值为 bitValue 的 offset 位置。

在默认情况下， 命令将检测整个位图， 用户可以通过可选的 start 参数和 end 参数指定要检测的范围。所以我们可以通过执行这条命令来获取 userID = 100 在 2022 年 6 月份首次打卡日期：

```bash
BITPOS uid:sign:100:202206 1
```

需要注意的是，因为 offset 从 0 开始的，所以我们需要将返回的 value + 1 。

###### 判断用户登陆态

Bitmap 提供了 GETBIT、SETBIT 操作，通过一个偏移值 offset 对 bit 数组的 offset 位置的 bit 位进行读写操作，需要注意的是 offset 从 0 开始。

只需要一个 key = login_status 表示存储用户登陆状态集合数据， 将用户 ID 作为 offset，在线就设置为 1，下线设置 0。通过 GETBIT 判断对应的用户是否在线。 5000 万用户只需要 6 MB 的空间。假如我们要判断 ID = 10086 的用户的登陆情况：

第一步，执行以下指令，表示用户已登录。

```bash
SETBIT login_status 10086 1
```

第二步，检查该用户是否登陆，返回值 1 表示已登录。

```bash
GETBIT login_status 10086
```

第三步，登出，将 offset 对应的 value 设置成 0。

```bash
SETBIT login_status 10086 0
```

###### 连续签到用户总数

如何统计出这连续 7 天连续打卡用户总数呢？我们把每天的日期作为 Bitmap 的 key，userId 作为 offset，若是打卡则将 offset 位置的 bit 设置成 1。key 对应的集合的每个 bit 位的数据则是一个用户在该日期的打卡记录。

一共有 7 个这样的 Bitmap，如果我们能对这 7 个 Bitmap 的对应的 bit 位做 『与』运算。同样的 UserID offset 都是一样的，当一个 userID 在 7 个 Bitmap 对应的 offset 位置的 bit = 1 就说明该用户 7 天连续打卡。

结果保存到一个新 Bitmap 中，我们再通过 BITCOUNT 统计 bit = 1 的个数便得到了连续打卡 7 天的用户总数了。

Redis 提供了`BITOP operation destkey key [key ...]`这个指令用于对一个或者多个 key 的 Bitmap 进行位元操作。

- operation 可以是 and、OR、NOT、XOR。当 BITOP 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 0 。空的 key 也被看作是包含 0 的字符串序列。

假设要统计 3 天连续打卡的用户数，则是将三个 bitmap 进行 AND 操作，并将结果保存到 destmap 中，接着对 destmap 执行 BITCOUNT 统计，如下命令：

```bash
# 与操作
BITOP AND destmap bitmap:01 bitmap:02 bitmap:03
# 统计 bit 位 =  1 的个数
BITCOUNT destmap
```

即使一天产生一个亿的数据，Bitmap 占用的内存也不大，大约占 12 MB 的内存（10^8/8/1024/1024），7 天的 Bitmap 的内存开销约为 84 MB。同时我们最好给 Bitmap 设置过期时间，让 Redis 删除过期的打卡数据，节省内存。

#### HyperLogLog

Redis HyperLogLog 是 Redis 2.8.9 版本新增的数据类型，是一种用于「统计基数」的数据集合类型，基数统计就是指统计一个集合中不重复的元素个数。但要注意，HyperLogLog 是统计规则是基于概率完成的，不是非常准确，标准误算率是 0.81%。所以，简单来说 HyperLogLog 提供不精确的去重计数。

HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的内存空间总是固定的、并且是很小的。在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。

##### 应用场景

###### 百万级网页 UV 计数

Redis HyperLogLog 优势在于只需要花费 12 KB 内存，就可以计算接近 2^64 个元素的基数，和元素越多就越耗费内存的 Set 和 Hash 类型相比，HyperLogLog 就非常节省空间。所以，非常适合统计百万级以上的网页 UV 的场景。

在统计 UV 时，你可以用 PFADD 命令（用于向 HyperLogLog 中添加新元素）把访问页面的每个用户都添加到 HyperLogLog 中。

```bash
PFADD page1:uv user1 user2 user3 user4 user5
```

接下来，就可以用 PFCOUNT 命令直接获得 page1 的 UV 值了，这个命令的作用就是返回 HyperLogLog 的统计结果。

```bash
PFCOUNT page1:uv
```

不过，有一点需要你注意一下，HyperLogLog 的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是 0.81%。这也就意味着，你使用 HyperLogLog 统计的 UV 是 100 万，但实际的 UV 可能是 101 万。虽然误差率不算大，但是，如果你需要精确统计结果的话，最好还是继续用 Set 或 Hash 类型。

#### GEO

Redis GEO 是 Redis 3.2 版本新增的数据类型，主要用于存储地理位置信息，并对存储的信息进行操作。在日常生活中，我们越来越依赖搜索“附近的餐馆”、在打车软件上叫车，这些都离不开基于位置信息服务（Location-Based Service，LBS）的应用。LBS 应用访问的数据是和人或物关联的一组经纬度信息，而且要能查询相邻的经纬度范围，GEO 就非常适合应用在 LBS 服务的场景中。

##### 内部实现

GEO 本身并没有设计新的底层数据结构，而是直接使用了 Sorted Set 集合类型。

GEO 类型使用 GeoHash 编码方法实现了经纬度到 Sorted Set 中元素权重分数的转换，这其中的两个关键机制就是「对二维地图做区间划分」和「对区间进行编码」。一组经纬度落在某个区间后，就用区间的编码值来表示，并把编码值作为 Sorted Set 元素的权重分数。

这样一来，我们就可以把经纬度保存到 Sorted Set 中，利用 Sorted Set 提供的“按权重进行有序范围查找”的特性，实现 LBS 服务中频繁使用的“搜索附近”的需求。

##### 应用场景

###### 滴滴叫车

假设车辆 ID 是 33，经纬度位置是（116.034579，39.030452），我们可以用一个 GEO 集合保存所有车辆的经纬度，集合 key 是 cars:locations。

执行下面的这个命令，就可以把 ID 号为 33 的车辆的当前经纬度位置存入 GEO 集合中：

```bash
GEOADD cars:locations 116.034579 39.030452 33
```

当用户想要寻找自己附近的网约车时，LBS 应用就可以使用 GEORADIUS 命令。

例如，LBS 应用执行下面的命令时，Redis 会根据输入的用户的经纬度信息（116.054579，39.030452 ），查找以这个经纬度为中心的 5 公里内的车辆信息，并返回给 LBS 应用。

```bash
GEORADIUS cars:locations 116.054579 39.030452 5 km ASC COUNT 10
```

#### Stream

Redis Stream 是 Redis 5.0 版本新增加的数据类型，Redis 专门为消息队列设计的数据类型。在 Redis 5.0 Stream 没出来之前，消息队列的实现方式都有着各自的缺陷，例如：

- 发布订阅模式，不能持久化也就无法可靠的保存消息，并且对于离线重连的客户端不能读取历史消息的缺陷；
- List 实现消息队列的方式不能重复消费，一个消息消费完就会被删除，而且生产者需要自行实现全局唯一 ID。

基于以上问题，Redis 5.0 便推出了 Stream 类型也是此版本最重要的功能，用于完美地实现消息队列，它支持消息的持久化、支持自动生成全局唯一 ID、支持 ack 确认消息的模式、支持消费组模式等，让消息队列更加的稳定和可靠。

##### 应用场景

###### 消息队列

生产者通过 XADD 命令插入一条消息：

```bash
# * 表示让 Redis 为插入的数据自动生成一个全局唯一的 ID
# 往名称为 mymq 的消息队列中插入一条消息，消息的键是 name，值是 xiaolin
> XADD mymq * name xiaolin
"1654254953808-0"
```

插入成功后会返回全局唯一的 ID："1654254953808-0"。消息的全局唯一 ID 由两部分组成：

- 第一部分“1654254953808”是数据插入时，以毫秒为单位计算的当前服务器时间；
- 第二部分表示插入消息在当前毫秒内的消息序号，这是从 0 开始编号的。例如，“1654254953808-0”就表示在“1654254953808”毫秒内的第 1 条消息。

消费者通过 XREAD 命令从消息队列中读取消息时，可以指定一个消息 ID，并从这个消息 ID 的下一条消息开始进行读取（注意是输入消息 ID 的下一条信息开始读取，不是查询输入ID的消息）。

```bash
# 从 ID 号为 1654254953807-0 的消息开始，读取后续的所有消息（示例中一共 1 条）。
> XREAD STREAMS mymq 1654254953807-0
1) 1) "mymq"
   2) 1) 1) "1654254953808-0"
         2) 1) "name"
            2) "xiaolin"
```

如果想要实现阻塞读（当没有数据时，阻塞住），可以调用 XRAED 时设定 BLOCK 配置项，实现类似于 BRPOP 的阻塞读取操作。

比如，下面这命令，设置了 BLOCK 10000 的配置项，10000 的单位是毫秒，表明 XREAD 在读取最新消息时，如果没有消息到来，XREAD 将阻塞 10000 毫秒（即 10 秒），然后再返回。

```bash
# 命令最后的“$”符号表示读取最新的消息
> XREAD BLOCK 10000 STREAMS mymq $
(nil)
(10.00s)
```

Stream 的基础方法，使用 xadd 存入消息和 xread 循环阻塞读取消息的方式可以实现简易版的消息队列。

> 前面介绍的这些操作 List 也支持的，接下来看看 Stream 特有的功能。

Stream 可以以使用 XGROUP 创建消费组，创建消费组之后，Stream 可以使用 XREADGROUP 命令让消费组内的消费者读取消息。

创建两个消费组，这两个消费组消费的消息队列是 mymq，都指定从第一条消息开始读取：

```bash
# 创建一个名为 group1 的消费组，0-0 表示从第一条消息开始读取。
> XGROUP CREATE mymq group1 0-0
OK
# 创建一个名为 group2 的消费组，0-0 表示从第一条消息开始读取。
> XGROUP CREATE mymq group2 0-0
OK
```

消费组 group1 内的消费者 consumer1 从 mymq 消息队列中读取所有消息的命令如下：

```bash
# 命令最后的参数“>”，表示从第一条尚未被消费的消息开始读取。
> XREADGROUP GROUP group1 consumer1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654254953808-0"
         2) 1) "name"
            2) "xiaolin"
```

消息队列中的消息一旦被消费组里的一个消费者读取了，就不能再被该消费组内的其他消费者读取了，即同一个消费组里的消费者不能消费同一条消息。

比如说，我们执行完刚才的 XREADGROUP 命令后，再执行一次同样的命令，此时读到的就是空值了：

```bash
> XREADGROUP GROUP group1 consumer1 STREAMS mymq >
(nil)
```

但是，不同消费组的消费者可以消费同一条消息（但是有前提条件，创建消息组的时候，不同消费组指定了相同位置开始读取消息）。

比如说，刚才 group1 消费组里的 consumer1 消费者消费了一条 id 为 1654254953808-0 的消息，现在用 group2 消费组里的 consumer1 消费者消费消息：

```bash
> XREADGROUP GROUP group2 consumer1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654254953808-0"
         2) 1) "name"
            2) "xiaolin"
```

因为我创建两组的消费组都是从第一条消息开始读取，所以可以看到第二组的消费者依然可以消费 id 为 1654254953808-0 的这一条消息。因此，不同的消费组的消费者可以消费同一条消息。

使用消费组的目的是让组内的多个消费者共同分担读取消息，所以，我们通常会让每个消费者读取部分消息，从而实现消息读取负载在多个消费者间是均衡分布的。

例如，我们执行下列命令，让 group2 中的 consumer1、2、3 各自读取一条消息：

```bash
# 让 group2 中的 consumer1 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer1 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654254953808-0"
         2) 1) "name"
            2) "xiaolin"
# 让 group2 中的 consumer2 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer2 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654256265584-0"
         2) 1) "name"
            2) "xiaolincoding"
# 让 group2 中的 consumer3 从 mymq 消息队列中消费一条消息
> XREADGROUP GROUP group2 consumer3 COUNT 1 STREAMS mymq >
1) 1) "mymq"
   2) 1) 1) "1654256271337-0"
         2) 1) "name"
            2) "Tom"
```

> 基于 Stream 实现的消息队列，如何保证消费者在发生故障或宕机再次重启后，仍然可以读取未处理完的消息？

Streams 会自动使用内部队列（也称为 PENDING List）留存消费组里每个消费者读取的消息，直到消费者使用 XACK 命令通知 Streams “消息已经处理完成”。

消费确认增加了消息的可靠性，一般在业务处理完成之后，需要执行 XACK 命令确认消息已经被消费完成。

如果消费者没有成功处理消息，它就不会给 Streams 发送 XACK 命令，消息仍然会留存。此时，消费者可以在重启后，用 XPENDING 命令查看已读取、但尚未确认处理完成的消息。

例如，我们来查看一下 group2 中各个消费者已读取、但尚未确认的消息个数，命令如下：

```bash
127.0.0.1:6379> XPENDING mymq group2
1) (integer) 3
2) "1654254953808-0"  # 表示 group2 中所有消费者读取的消息最小 ID
3) "1654256271337-0"  # 表示 group2 中所有消费者读取的消息最大 ID
4) 1) 1) "consumer1"
      2) "1"
   2) 1) "consumer2"
      2) "1"
   3) 1) "consumer3"
      2) "1"
```

如果想查看某个消费者具体读取了哪些数据，可以执行下面的命令：

```bash
# 查看 group2 里 consumer2 已从 mymq 消息队列中读取了哪些消息
> XPENDING mymq group2 - + 10 consumer2
1) 1) "1654256265584-0"
   2) "consumer2"
   3) (integer) 410700
   4) (integer) 1
```

可以看到，consumer2 已读取的消息的 ID 是 1654256265584-0。一旦消息 1654256265584-0 被 consumer2 处理了，consumer2 就可以使用 XACK 命令通知 Streams，然后这条消息就会被删除。

```bash
> XACK mymq group2 1654256265584-0
(integer) 1
```

当我们再使用 XPENDING 命令查看时，就可以看到，consumer2 已经没有已读取、但尚未确认处理的消息了。

```bash
> XPENDING mymq group2 - + 10 consumer2
(empty array)
```

总结一下：

- 消息保序：XADD/XREAD；
- 阻塞读取：XREAD block；
- 重复消息处理：Stream 在使用 XADD 命令，会自动生成全局唯一 ID；
- 消息可靠性：内部使用 PENDING List 自动保存消息，使用 XPENDING 命令查看消费组已经读取但是未被确认的消息，消费者使用 XACK 确认消息；
- 支持消费组形式消费数据。

> Redis 基于 Stream 消息队列与专业的消息队列有哪些差距？

一个专业的消息队列，必须要做到两大块：

- 消息不丢。
- 消息可堆积。

1. Redis Stream 消息会丢失吗？

使用一个消息队列，其实就分为三大块：生产者、队列中间件、消费者，所以要保证消息就是保证三个环节都不能丢失数据。

Redis Stream 消息队列能不能保证三个环节都不丢失数据？

- Redis 生产者会不会丢消息？生产者会不会丢消息，取决于生产者对于异常情况的处理是否合理。 从消息被生产出来，然后提交给 MQ 的过程中，只要能正常收到 （ MQ 中间件） 的 ack 确认响应，就表示发送成功，所以只要处理好返回值和异常，如果返回异常则进行消息重发，那么这个阶段是不会出现消息丢失的。
- Redis 消费者会不会丢消息？不会，因为 Stream （ MQ 中间件）会自动使用内部队列（也称为 PENDING List）留存消费组里每个消费者读取的消息，但是未被确认的消息。消费者可以在重启后，用 XPENDING 命令查看已读取、但尚未确认处理完成的消息。等到消费者执行完业务逻辑后，再发送消费确认 XACK 命令，也能保证消息的不丢失。
- Redis 消息中间件会不会丢消息？会，Redis 在以下 2 个场景下，都会导致数据丢失：
	- AOF 持久化配置为每秒写盘，但这个写盘过程是异步的，Redis 宕机时会存在数据丢失的可能；
	- 主从复制也是异步的，主从切换时，也存在丢失数据的可能。

可以看到，Redis 在队列中间件环节无法保证消息不丢。像 RabbitMQ 或 Kafka 这类专业的队列中间件，在使用时是部署一个集群，生产者在发布消息时，队列中间件通常会写「多个节点」，也就是有多个副本，这样一来，即便其中一个节点挂了，也能保证集群的数据不丢失。

2. Redis Stream 消息可堆积吗？

Redis 的数据都存储在内存中，这就意味着一旦发生消息积压，则会导致 Redis 的内存持续增长，如果超过机器内存上限，就会面临被 OOM 的风险。

所以 Redis 的 Stream 提供了可以指定队列最大长度的功能，就是为了避免这种情况发生。

当指定队列最大长度时，队列长度超过上限后，旧消息会被删除，只保留固定长度的新消息。这么来看，Stream 在消息积压时，如果指定了最大长度，还是有可能丢失消息的。

但 Kafka、RabbitMQ 专业的消息队列它们的数据都是存储在磁盘上，当消息积压时，无非就是多占用一些磁盘空间。

因此，把 Redis 当作队列来使用时，会面临的 2 个问题：

- Redis 本身可能会丢数据；
- 面对消息挤压，内存资源会紧张；

所以，能不能将 Redis 作为消息队列来使用，关键看你的业务场景：

- 如果你的业务场景足够简单，对于数据丢失不敏感，而且消息积压概率比较小的情况下，把 Redis 当作队列是完全可以的。
- 如果你的业务有海量消息，消息积压的概率比较大，并且不能接受数据丢失，那么还是用专业的消息队列中间件吧。

> 补充：Redis 发布/订阅机制为什么不可以作为消息队列？

发布订阅机制存在以下缺点，都是跟丢失数据有关：

1. 发布/订阅机制没有基于任何数据类型实现，所以不具备「数据持久化」的能力，也就是发布/订阅机制的相关操作，不会写入到 RDB 和 AOF 中，当 Redis 宕机重启，发布/订阅机制的数据也会全部丢失。
2. 发布订阅模式是“发后既忘”的工作模式，如果有订阅者离线重连之后不能消费之前的历史消息。
3. 当消费端有一定的消息积压时，也就是生产者发送的消息，消费者消费不过来时，如果超过 32M 或者是 60s 内持续保持在 8M 以上，消费端会被强行断开，这个参数是在配置文件中设置的，默认值是`client-output-buffer-limit pubsub 32mb 8mb 60`。

所以，发布/订阅机制只适合即时通讯的场景，比如构建哨兵集群的场景采用了发布/订阅机制。

### Redis 数据结构

#### SDS 

字符串在 Redis 中是很常用的，键值对中的键是字符串类型，值有时也是字符串类型。

Redis 是用 C 语言实现的，但是它没有直接使用 C 语言的 char * 字符数组来实现字符串，而是自己封装了一个名为简单动态字符串（simple dynamic string，SDS） 的数据结构来表示字符串，也就是 Redis 的 String 数据类型的底层数据结构是 SDS。

##### C 语言字符串的缺陷

C 语言的字符串其实就是一个字符数组，即数组中每个元素是字符串中的一个字符。在 C 语言里，对字符串操作时，char * 指针只是指向字符数组的起始位置，而字符数组的结尾位置就用 “\0” 表示，意思是指字符串的结束。

因此，C 语言标准库中的字符串操作函数就通过判断字符是不是 “\0” 来决定要不要停止操作，如果当前字符不是 “\0” ，说明字符串还没结束，可以继续操作，如果当前字符是 “\0” 是则说明字符串结束了，就要停止操作。

C 语言获取字符串长度的时间复杂度是 O（N），而且 C 语言字符串用 “\0” 字符作为结尾标记有个缺陷。假设有个字符串中有个 “\0” 字符，这时在操作这个字符串时就会提早结束，这个限制使得 C 语言的字符串只能保存文本数据，不能保存像图片、音频、视频文化这样的二进制数据。

另外， C 语言标准库中字符串的操作函数是很不安全的，对程序员很不友好，稍微一不注意，就会导致缓冲区溢出。

##### SDS 结构设计

下图就是 Redis 5.0 的 SDS 的数据结构：

![RedisSDS数据结构](../picture/RedisSDS数据结构.png)

结构中的每个成员变量分别介绍下：

- len，记录了字符串长度。这样获取字符串长度的时候，只需要返回这个成员变量值就行，时间复杂度只需要 O（1）。
- alloc，分配给字符数组的空间长度。这样在修改字符串的时候，可以通过 alloc - len 计算出剩余的空间大小，可以用来判断空间是否满足修改需求，如果不满足的话，就会自动将 SDS 的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用 SDS 既不需要手动修改 SDS 的空间大小，也不会出现前面所说的缓冲区溢出的问题。
- flags，用来表示不同类型的 SDS。一共设计了 5 种类型，分别是 sdshdr5、sdshdr8、sdshdr16、sdshdr32 和 sdshdr64，后面在说明区别之处。
- buf[]，字符数组，用来保存实际数据。不仅可以保存字符串，也可以保存二进制数据。

总的来说，Redis 的 SDS 结构在原本字符数组之上，增加了三个元数据：len、alloc、flags，用来解决 C 语言字符串的缺陷。

1. O（1）复杂度获取字符串长度

C 语言的字符串长度获取 strlen 函数，需要通过遍历的方式来统计字符串长度，时间复杂度是 O（N）。而 Redis 的 SDS 结构因为加入了 len 成员变量，那么获取字符串长度的时候，直接返回这个成员变量的值就行，所以复杂度只有 O（1）。

2. 二进制安全

因为 SDS 不需要用 “\0” 字符来标识字符串结尾了，而是有个专门的 len 成员变量来记录长度，所以可存储包含 “\0” 的数据。但是 SDS 为了兼容部分 C 语言标准库的函数， SDS 字符串结尾还是会加上 “\0” 字符。

因此， SDS 的 API 都是以处理二进制的方式来处理 SDS 存放在 buf[] 里的数据，程序不会对其中的数据做任何限制，数据写入的时候时什么样的，它被读取时就是什么样的。

通过使用二进制安全的 SDS，而不是 C 字符串，使得 Redis 不仅可以保存文本数据，也可以保存任意格式的二进制数据。

3. 不会发生缓冲区溢出

C 语言的字符串标准库提供的字符串操作函数，大多数都是不安全的，因为这些函数把缓冲区大小是否满足操作需求的工作交由开发者来保证，程序内部并不会判断缓冲区大小是否足够用，当发生了缓冲区溢出就有可能造成程序异常结束。

所以，Redis 的 SDS 结构里引入了 alloc 和 len 成员变量，这样 SDS API 通过 alloc - len 计算，可以算出剩余可用的空间大小，这样在对字符串做修改操作的时候，就可以由程序内部判断缓冲区大小是否足够用。

而且，当判断出缓冲区大小不够用时，Redis 会自动扩大 SDS 的空间大小，以满足修改所需的大小。

SDS 扩容的规则为：

- 如果所需的 sds 长度小于 1 MB，那么最后的扩容是按照翻倍扩容来执行的，即 2 倍的newlen；
- 如果所需的 sds 长度超过 1 MB，那么最后的扩容长度应该是 newlen + 1MB。

在扩容 SDS 空间之前，SDS API 会优先检查未使用空间是否足够，如果不够的话，API 不仅会为 SDS 分配修改所必须要的空间，还会给 SDS 分配额外的「未使用空间」。

这样的好处是，下次在操作 SDS 时，如果 SDS 空间够的话，API 就会直接使用「未使用空间」，而无须执行内存分配，有效的减少内存分配次数。

所以，使用 SDS 即不需要手动修改 SDS 的空间大小，也不会出现缓冲区溢出的问题。

4. 节省内存空间

SDS 结构中有个 flags 成员变量，表示的是 SDS 类型。Redis 一共设计了 5 种类型，分别是 sdshdr5、sdshdr8、sdshdr16、sdshdr32 和 sdshdr64。这 5 种类型的主要区别就在于，它们数据结构中的 len 和 alloc 成员变量的数据类型不同。

之所以 SDS 设计不同类型的结构体，是为了能灵活保存不同大小的字符串，从而有效节省内存空间。比如，在保存小字符串时，结构头占用空间也比较少。

除了设计不同类型的结构体，Redis 在编程上还使用了专门的编译优化来节省内存空间，即在 struct 声明了`__attribute__ ((packed))` ，它的作用是：告诉编译器取消结构体在编译过程中的优化对齐，按照实际占用字节数进行对齐。

#### 链表

先来看看「链表节点」结构的样子：

```c
typedef struct listNode {
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
} listNode;
```

Redis 在 listNode 结构体基础上又封装了 list 这个数据结构，这样操作起来会更方便，链表结构如下：

```c
typedef struct list {
    //链表头节点
    listNode *head;
    //链表尾节点
    listNode *tail;
    //节点值复制函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值比较函数
    int (*match)(void *ptr, void *key);
    //链表节点数量
    unsigned long len;
} list;
```

##### 链表的优势与缺陷

Redis 的链表实现优点如下：

- listNode 链表节点的结构里带有 prev 和 next 指针，获取某个节点的前置节点或后置节点的时间复杂度只需 O(1)，而且这两个指针都可以指向 NULL，所以链表是无环链表；
- list 结构因为提供了表头指针 head 和表尾节点 tail，所以获取链表的表头节点和表尾节点的时间复杂度只需 O(1)；
- list 结构因为提供了链表节点数量 len，所以获取链表中的节点数量的时间复杂度只需O(1)；
- listNode 链表节使用 void * 指针保存节点值，并且可以通过 list 结构的 dup、free、match 函数指针为节点设置该节点类型特定的函数，因此链表节点可以保存各种不同类型的值；

缺点如下：

- 链表每个节点之间的内存都是不连续的，意味着无法很好利用 CPU 缓存。能很好利用 CPU 缓存的数据结构就是数组，因为数组的内存是连续的，这样就可以充分利用 CPU 缓存来加速访问。
- 还有一点，保存一个链表节点的值都需要一个链表节点结构头的分配，内存开销较大。

因此，Redis 3.0 的 List 对象在数据量比较少的情况下，会采用「压缩列表」作为底层数据结构的实现，它的优势是节省内存空间，并且是内存紧凑型的数据结构。

不过，压缩列表存在性能问题，所以 Redis 在 3.2 版本设计了新的数据结构 quicklist，并将 List 对象的底层数据结构改由 quicklist 实现。

然后在 Redis 5.0 设计了新的数据结构 listpack，沿用了压缩列表紧凑型的内存布局，最终在最新的 Redis 版本，将 Hash 对象和 Zset 对象的底层数据结构实现之一的压缩列表，替换成由 listpack 实现。

#### 压缩列表 Ziplist

压缩列表的最大特点，就是它被设计成一种内存紧凑型的数据结构，占用一块连续的内存空间，不仅可以利用 CPU 缓存，而且会针对不同长度的数据，进行相应编码，这种方法可以有效地节省内存开销。

但是，压缩列表的缺陷也是有的：

- 不能保存过多的元素，否则查询效率就会降低；
- 新增或修改某个元素时，压缩列表占用的内存空间需要重新分配，甚至可能引发连锁更新的问题。

因此，Redis 对象（List 对象、Hash 对象、Zset 对象）包含的元素数量较少，或者元素值不大的情况才会使用压缩列表作为底层数据结构。

##### 压缩列表结构设计

压缩列表是 Redis 为了节约内存而开发的，它是由连续内存块组成的顺序型数据结构，有点类似于数组。

![Redis压缩列表结构](../picture/Redis压缩列表结构.png)

- zlbytes，记录整个压缩列表占用对内存字节数；
- zltail，记录压缩列表「尾部」节点距离起始地址多少字节，也就是列表尾的偏移量；
- zllen，记录压缩列表包含的节点数量；
- zlend，标记压缩列表的结束点，固定值 0xFF（十进制255）。

在压缩列表中，如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段（zllen）的长度直接定位，复杂度是 O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是 O(N) 了，因此压缩列表不适合保存过多的元素。

另外，压缩列表节点（entry）的构成如下：

![Redis压缩列表节点结构](../picture/Redis压缩列表节点结构.png)






































































## 单线程

## 多线程

## 多进程

## 缓存过期删除和缓存淘汰

## 持久化

## Redis 高可用

## 缓存

## 分布式锁

## 事务

## 面试题