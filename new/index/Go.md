# Go

## 逃逸分析

### 什么是逃逸

逃逸分析是编译器用于决定变量分配到堆上还是栈上的一种行为。

函数的运行都是在栈上面运行的，在栈上面声明临时变量，分配内存，函数运行完毕之后，回收内存，每个函数的栈空间都是独立的，其他函数是无法进行访问的，但是在某些情况下栈上面的数据需要在函数结束之后还能被访问，这时候就会涉及到内存逃逸了，什么是逃逸，就是抓不住。

如果数据从栈上面逃逸，会跑到堆上面，栈上面的数据在函数结束的时候会自动回收，回收代价比较小，栈的内存分配和使用一般只需要两个 CPU 指令 “PUSH” 和 “RELEASE”，分配和释放，而堆分配内存，则是首先需要找到一块合适的内存，之后通过 GC 回收才能释放，对于这种情况，频繁的使用垃圾回收，则会占用比较大的系统开销，所以尽量分配内存到栈上面，减少 GC 的压力，提高程序运行速度。

### 逃逸分析过程

Go 语言最基本的逃逸分析原则：如果一个函数返回一个对变量的引用，那么它就会发生逃逸。

在任何情况下，如果一个值被分配到了栈之外的地方，那么一定是到了堆上面。简而概之：编译器会分析代码的特征和代码声明周期，Go 中的变量只有在编译器可以证明在函数返回后不会再被引用的，才分配到栈上，其他情况都是分配到堆上。

Go 语言里面没有一个关键字或者函数可以直接让变量被编译器分配到堆上，相反，编译器是通过分析代码来决定将变量分配到何处。简单来说，编译器会根据变量是否被外部引用来决定是否逃逸：

- 如果函数外部没有引用，则优先放到栈中；
- 如果函数外部存在引用，则必定放到堆中；

### 指针逃逸

我们知道传递指针可以减少底层值的拷贝，可以提高效率，但是如果拷贝的数据量小，由于指针传递会发生逃逸，可能会使用堆，也可能增加 GC 的负担，所以传递指针不一定是高效的。

### 动态类型逃逸

很多函数参数为 interface 类型，编译期间很难确定其参数的具体类型，也能产生逃逸。

### 逃逸常见情况

1. 发送指针的指针或值包含了指针到 Channel 中，由于在编译阶段无法确定其作用域与传递的路径，所以一般都会逃逸到堆上分配。
2. slice 中的值是指针的指针或包含指针字段。一个例子是类似`[]*string`的类型。这总是导致 slice 的逃逸。即使切片的底层仍可能位于栈上，数据的引用也会转移到堆中。
3. slice 由于 append 操作超出其容量，因此会导致 slice 重新分配。这种情况下，由于在编译时 slice 的初始大小的已知情况下，将会在栈上分配。如果 slice 的底层存储必须基于仅在运行时数据进行扩展，则它将分配在堆上。
4. 调用接口类型的方法。接口类型的方法调用是动态调度，实际使用的具体实现只能在运行时确定。考虑一个接口类型为`io.Reader`的变量 r，对`r.Read(b)`的调用将导致 r 的值和字节片 b 的后续转义并因此分配到堆上。
5. 尽管能够符合分配到栈的场景，但是其大小不能够在编译时确定，也会分配到堆上。

### 如何避免

1. Go 中的接口类型的方法调用是动态调度，因此不能够在编译阶段确定，所有类型结构转换成接口的过程涉及到内存逃逸的情况发生。如果对于性能要求比较高且访问频次比较高的函数调用，应该尽量避免使用接口类型 。
2. 由于切片一般是使用在函数传递的场景下，而且切片在 append 的时候可能会涉及到重新分配内存，如果切片在编译期间的大小不能够确认或者大小超出栈的限制，多数情况下都会分配到堆上。

### 总结

1. 堆上动态分配内存比栈上静态分配内存，开销大很多。
2. 变量分配在栈上需要能在编译期确定它的作用域，否则会分配到堆上。
3. Go 编译器会在编译期考察变量的作用域，并做一系列检查，如果他的作用域在运行期间对编译器是一直可知的，那么就会分配到栈上。简单来说，编译器会根据变量是否被外部引用来决定是否逃逸。
4. 对 Go 程序员来说，编译器的这些逃逸分析规则不需要掌握，只需要通过`go build -gcflags -m`的命令来观察变量逃逸情况就行。
5. 不要盲目使用变量的指针作为函数参数，虽然它会减少复制操作。但其实当参数为变量自身的时候，复制是在栈上完成的操作，开销远比变量逃逸后动态地在堆上分配内存少的多。

## String 面试与分析

### String 的底层数据结构是怎样的

go 语言中字符串的底层实现是一个结构类型，包含两个字段，一个只想字节数组的指针，另一个是字符串的字节长度。

### 字符串可以被修改吗

字符串不可以不修改

### []byte 转化为 string 互相转换会发生内存拷贝吗

会发生内存拷贝，所以程序中应避免出现大量的长字符串的这种转换。

### 字符串拼接有哪几种方式，各自的性能怎么样

| 方法            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| +               | + 拼接 2 个字符串时，会生成一个新的字符串，开辟一段新的内存空间，新空间的大小是原来两个字符串的大小之和，所以每拼接一次就要开辟一段空间，性能很差。 |
| Sprintf         | Sprintf 会从临时对象池中获取一个对象，然后格式化操作，最后转化为 string，释放对象，实现很复杂，性能也很差。 |
| strings.Builder | 底层存储使用 []byte，转化为字符串时可复用，每次分配内存的时候，支持预分配内存并且自动扩容，所以总体来说，开辟内存的次数越少，性能相对就高。 |
| bytes.Buffer    | 底层存储使用 []byte，转化为字符串时不可复用，底层实现和 strings.Builder 差不多，性能比 strings.Builder 略差一点，区别是 bytes.Buffer 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 strings.Builder 直接将底层的 []byte 转换成了字符串类型返回了。 |
| append          | 直接使用 []byte 扩容机制，可复用，支持预分配内存和自动扩容，性能最好。 |

## Slice 面试与分析

### Slice 的底层数据结构是怎样的

slice 的底层实现是一个结构类型，有三个字段，真想一个数组的指针 Pointer、切片的长度 len、切片的容量 cap。

### 从一个切片截取出另一个切片，修改新切片的值会影响原来的切片内容吗

在截取完之后，如果新切片没有触发扩容，则修改切片元素会影响原切片，如果触发了扩容则不会。

### 切片的扩容策略是怎样的

Go 1.17 版本以前：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果当前切片的长度小于 1024 就会将容量翻倍；
3. 如果当前切片的长度大于 1024 就会将每次增加 25% 的容量，直到新容量大于期望容量。

Go 1.18 之后：

1. 如果期望容量大于当前容量的两倍就会使用期望容量；
2. 如果扩容前容量小于 256 就会将容量翻倍；
3. 按照公式扩容，扩容公式为`newcap = oldcap + (oldcap + 3 * 256)/4`。

## Map 面试与分析

### Map 的底层实现原理

Map 的底层实现数据结构实际上是一个哈希表。在运行时表现为指向 hmap 结构的指针，hmap 中记录了桶数组指针、溢出桶指针以及元素个数等字段。每个桶是一个 bmap 的数据结构，可以存储 8 个键值对和 8 个 tophash 以及指向下一个溢出桶的指针。为了内存紧凑，采用的是先存 8 个 key 后再存 value。

### 为什么遍历 Map 是无序的？

Map 在遍历的时候会随机一个桶号和槽位，从这个随机的桶号开始，在每个桶中从这个随机的槽位开始遍历完所有的桶，至于为什么要随机开始？因为 Map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 的位置就会发生改变。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的恶变化，有些 key 位置不动，有些迁移到了新位置。这样，遍历 Map 的结果就不可能按原来的顺序。所以，Go 语言强制每次遍历都是随机开始。

### 如何实现有序遍历 Map？

可以在遍历的时候将结果保存到一个 Slice 里，对 Slice 进行排序。

### 为什么 Go Map 是非线程安全的？

Go 官方给出的原因是：Map 适配的场景应该是简单的（不需要从多个 goroutine 中进行安全访问的），而不是为了小部分情况（并发访问），导致大部分程序付出锁的代价，因此决定了不支持。

### 线程安全的 Map 如何实现？

加锁或者使用 sync.Map

### Go sync.Map 和 原生 Map 谁的性能好，为什么？

原生 Map 的性能好，因为 Go sync.Map 为了保证线程安全，以空间换时间，采用了 read 和 dirty 两个 Map，用了原子操作和锁两种方式来实现线程安全，只是在操作 read 的时候用原子操作，当 read 中不存在某个 key 的时候就要加锁操作 dirty，过程中还是会有加锁操作，所以性能上有损耗。

### 为什么 Go Map 的负载因子是 6.5 ？

负载因子 = 哈希表存储的元素个数 / 桶个数

装载因子越大，填入的元素越多，空间利用率就越高，但发生哈希冲突的几率就越大。

装载因子越小，填入的元素越少，冲突发生的几率减小，但空间浪费也会变得更多，而且还会提高扩容操作的次数。

源码里对负载因子的定义是 6.5，是经过测试后取出的一个比较合理的值。每个 bucket 有 8 个空位，假设 Map 里所有的数组桶都装满元素，没有一个数组桶有溢出桶，那么这时的负载因子刚好是 8。而负载因子是 6.5 的时候，说明数组桶快要用完了，存在溢出的情况下，查找一个 key 很可能要去遍历溢出桶，会造成查找性能下降，所以有必要扩容了。

### Map 扩容策略是什么？

当负载因子大于 6.5 发生双倍扩容。

当溢出桶过多，发生等量扩容，溢出桶过多的标志：

1. 当正常桶数量大于 2^15 的时候，溢出桶多于正常桶，溢出桶数量就过多了；
2. 当正常桶数量大于 2^15 的时候，溢出桶一旦多于 2^15，溢出桶数量就过多了。

## sync.Map 面试与分析

### read map 什么时候更新？

delete 和 update 的时候 key 存在于 read，就会更新，当 misses 次数大于等于 dirty 长度时，会将 dirty 提升为 read。

### dirty map 什么时候会更新

- 插入一个新的 key 的时候，会直接插入到 dirty；
- 执行 store 操作的时候，当 read 中存在这个 key，dirty 中不存在这个 key 的时候，会更新，将这个 key/value 插入到 dirty；
- delete 的时候，当 read 中没有这个 key，而 dirty 中存在这个 key 的时候，会更新，将这个 key/value 直接从 dirty map 删除掉；
- 当 misses 数量大于等于 dirty 长度时，会将 dirty 提升为 read，将 dirty 中的所有 key/value 拷贝到 read 中，然后将 dirty 置为 nil；
- 当 dirty 为 nil 的时候，此时插入一个新的 key，会重塑 dirty，新建一个 dirty map 将 read 中非 expunged 状态的 key/value 拷贝到 dirty。

### read map 和 dirty map 的删除逻辑有什么区别？

read 删除是标记删除，并没有在 map 中实际删除这个 key，而是只是将这个 key 对应的 value 置为 nil，等到 misses 大于等于 dirty 长度的时候，将用 dirty 覆盖 read 的时候才会真正删除这个 key，是延迟删除。

dirty 中删除时直接将这个 key 从 dirty 这个 map 中删除掉，是直接删除。

### 那既然在删除 read 的时候没有删除这个 key，而 dirty 覆盖的时候又只覆盖了 read，那么假如 dirty 中也存在这个 key，这个 key 是不是会被遗漏，没有删掉，导致内存泄漏？

不会，因为在将 dirty 提升为 read，覆盖 read 完了之后，会将 dirty 置为 nil，这个 key 会被垃圾回收掉。

### sync.Map 中的 read 和 dirty 有什么关系？

read 可以看作是 dirty 的一个快照，在 dirty 不为空的时候，dirty 包含 map 中的所有有效 key，在 dirty 为空的时候，read 包含 map 中的所有有效 key。

在 read 的 misses 达到 dirty 的长度的时候，会将 dirty 提升为 read，用 dirty 中的所有 key/value 覆盖 read，之后 dirty 置为 nil，当 dirty 为 nil 的时候，插入一个新 key，此时会根据 read 来重塑 dirty，将 read 中非标记删除的 key/value 都 copy 到 dirty。

### sync.Map 中的值是否一定是有效的

不一定，sync.Map 中的值其实是由 entry 中的一个 p 指针指向的，p 可能有三种状态，nil、正常值、还有 expunged。当 p 的状态为 expunged 和 nil 时表示这个值是被删除了的，并不一定都是有效的。

### sync.Map 应用场景

1. 读多写少，因为写入新 key，是要加锁操作 dirty 的，所以写入操作太多性能不会太高。
2. 写操作也多，但是修改的 key 和读取的 key 特别不重合。

### sync.Map 是怎样提升性能的

1. 空间换时间。通过 read map 和 dirty map，优先操作 read map，减少频繁加锁对性能的影响；
2. 延迟删除。删除 key 并不是立即删除这个 key/value，而是先标记为 expunged，然后在 store 的过程中删除。
3. 读写分离设计。大部分读操作都在 read map 中，无需加锁。

## Channel 面试与分析

### channel 是否线程安全？锁用在什么地方？

是线程安全的，hchan 的底层实现中，hchan 结构体中采用 Mutex 锁来保证数据读写安全。在对循环数组 buf 中的数据进行入队和出队操作时，必须先获取互斥锁，才能操作 channel 数据。

### Go channel 的底层实现原理（数据结构）

channel 的底层实现是一个 hchan 的结构

```go
type hchan struct {
	qcount		uint						// 循环队列中的数据总数
	dataqsiz	uint						// 循环队列大小
	buf				unsafe.Pointer	// 指向循环队列的指针
	elemsize	uint16					// 循环队列中的每个元素的大小
	closed		uint32					// 标记位，标记channel是否关闭
	elemtype	*_type					// 循环队列中的元素类型
	sendx			uint						// 已发送元素在循环队列中的索引位置
	recvx			uint						// 已接收元素在循环队列中的索引位置
	recvq			waitq						// 等待从channel接收消息的sudog队列
	sendq			waitq						// 等待向channel写入消息的sudog队列
	lock			mutex						// 互斥锁，对channel的数据读写操作加锁，保证并发安全
}
```

### nil、关闭的 channel、有数据的 channel，再进行读、写、关闭会怎么样？（各类变种题型）例如：go channel close 后读的问题

向为 nil 的 channel 发送数据会怎么样？

会死锁，永久阻塞

### 向 channel 发送数据和从 channel 读数据的流程是什么样的？

### 哪些操作会使 channel 发生 panic ？

1. 往一个已经关闭的 channel 写数据
2. 关闭一个 nil 的 channel
3. 关闭一个已经关闭 的 channel

### 当一个 channel 关闭后，我们是否还能从 channel 读到数据？

可以，如果缓冲区中还有数据会取完数据，如果缓冲区内没有数据，会收到对应数据的零值。

### channel 是创建在堆上还是栈上

堆。

### channel 发送和接收元素的本质是什么？

值的拷贝。

### 有 4 个 goroutine，编号为 1、2、3、4，每秒钟会有一个 goroutine 打印出自己的编号，要求写一个程序，让输出的编号总是按照 1、2、3、4、1、2、3、4...的顺序打印出来

```go
func main() {
	c := make([]chan int, 4)
	for i := range c {
		c[i] = make(chan int)
		go func(i int) {
			for {
				v := <-c[i]
				fmt.Println(v + 1)
				time.Sleep(time.Second)
				c[(i+1)%4] <- (v + 1) % 4
			}
		}(i)
	}
	c[0] <- 0
	time.Sleep(time.Second * 10)
}
```

### 用 channel 实现一个限流器

```go
func main() {
	c := make(chan struct{}, 3)
	for i := 0; i < 20; i++ {
		c <- struct{}{}
		go func(i int) {
			defer func() {
				<-c
			}()
			// do something
		}(i)
	}
	
	time.Sleep(time.Second * 10)
}
```

### 用 channel 实现一个互斥锁

```go
type Mutex struct {
	ch chan struct{}
}

func NewMutex() *Mutex {
	mu := &Mutex{make(chan struct{}, 1)}
	mu.ch <- struct{}{}
	return mu
}

func (mu *Mutex) Lock() {
	<-mu.ch
}

func (mu *Mutex) Unlock() {
	select {
	case mu.ch <- struct{}{}:
	default:
		panic("unlock of unlocked mutex")
	}
}

func (mu *Mutex) TryLock() bool {
	select {
	case <-mu.ch:
		return true
	default:
	}
	return false
}

func (mu *Mutex) LockTimeout(timeout time.Duration) bool {
	timer := time.NewTimer(timeout)
	select {
	case <-mu.ch:
		timer.Stop()
		return true
	case <-timer.C:
	}
	return false
}

func (mu *Mutex) IsLocked() bool {
	return len(mu.ch) == 0
}

func main() {
	mu := NewMutex()
	ok := mu.TryLock()
	fmt.Println("TryLock:", ok)
	ok = mu.TryLock()
	fmt.Println("TryLock:", ok)

	go func() {
		time.Sleep(5 * time.Second)
		mu.Unlock()
	}()

	ok = mu.LockTimeout(3 * time.Second)
	fmt.Println("LockTimeout:", ok)
}
```






