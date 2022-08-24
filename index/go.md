# Go语言

## 函数 & 关键字

### 1. make()、new()、var的区别？

&emsp;&emsp;new的特点

- 分配内存。内存里存的值是对应类型的零值。
- 只有一个参数。参数是分配的内存空间所存储的数据类型，Go语言里的任何类型都可以是new的参数，比如int， 数组，结构体，甚至函数类型都可以。
- 返回的是指针。

注: new在项目中很少见，可以被多种方法替代。

&emsp;&emsp;make的特点

- 分配和初始化内存。
- 只能用于slice, map和chan这3个类型，不能用于其它类型。
- 如果是用于slice类型，make函数的第2个参数表示slice的长度，这个参数必须给值。
- 返回的是原始类型，也就是slice, map和chan，不是返回指向slice, map和chan的指针。

&emsp;&emsp;为什么针对slice, map和chan类型专门定义一个make函数？这是因为slice, map和chan的底层结构上要求在使用slice，map和chan的时候必须初始化，如果不初始化，那slice，map和chan的值就是零值，也就是nil。我们知道：map如果是nil，是不能往map插入元素的，插入元素会引发panic。chan如果是nil，往chan发送数据或者从chan接收数据会引发panic。slice会有点特殊，理论上slice如果是nil，也是没法用的。但是append函数处理了nil slice的情况，可以调用append函数对nil slice做扩容。但是我们使用slice，总是会希望可以自定义长度或者容量，这个时候就需要用到make。

&emsp;&emsp;new来创建slice, map和chan的都是nil，并没有什么用。

&emsp;&emsp;var的特点

- 声明一个type类型的变量，分配内存空间给type类型的零值。
- 声明一个type类型的指针变量，不会分配内存空间，零值为nil。
- 声明一个type类型的变量，并赋值。

## 切片

### 1. 切片的扩容策略？

&emsp;&emsp;切片在扩容时会进行内存对齐，这个和内存分配策略相关。进行内存对齐之后，新 slice 的容量是要 大于等于老 slice 容量的 2倍或者1.25倍，当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；原 slice 容量超过 1024，新 slice 容量变成原来的1.25倍。

## 内存

### 1. 逃逸分析

### 2. 内存分配

### 3. 垃圾回收

#### 3.1 垃圾回收相关。进行内存对齐之后，新 slice 的容量是要 大于等于老 slice 容量的 2倍或者1.25倍，当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；原 slice 容量超过 1024，新 slice 容量变成原来的1.25倍。