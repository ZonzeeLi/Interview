# Map & Sync.Map

## Map

### 哈希冲突

哈希表的原理是多个 k-v 键值对散列的存储在 buckets 中，buckets 可以理解为一个连续的数组，所以给定一个 key/value 键值对，需要存储到合适的位置需要经过两步骤：

1. 计算 hash 值：`hash = hashFunc(key)`
2. 计算索引位置：`index = hash % len(buckets)`

第一步是根据 hash 函数将 key 转化为一个 hash 值；
第二步是用 hash 值对桶的数量取模得到一个索引值，这样就得到了要插入的键值对的位置。

但是这里会出现一个问题，比如有两个键值对 key1/value1 和 key2/value2，经过哈希函数 hashFunc 的计算得到的哈希值 hash1 和 hash2 相同，那么索引值就会想通，所以会产生哈希冲突。

解决哈希冲突的方法一般有两种：拉链法和开放寻址法。

### 拉链法

