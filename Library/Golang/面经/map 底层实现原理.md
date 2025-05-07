Go 语言中的 `map` 是一种哈希表（Hash Table），它的底层实现融合了哈希表 + 拉链法 + 开放寻址（内联桶）+ 动态扩容等机制，保证在大多数情况下都能提供 **平均 O(1)** 的访问效率。

下面是 Go map 底层的核心原理和实现机制：

---

## 一、基本结构概述

Go map 的核心结构定义在 `runtime/map.go`，关键数据结构是：

### 1. `hmap`：map 的整体结构

```go
type hmap struct {
    count     int           // 当前键值对的个数
    flags     uint8
    B         uint8         // 2^B 是桶的数量（bucket 数）
    noverflow uint16        // 溢出桶个数（用于GC优化）
    hash0     uint32        // 哈希种子（防哈希冲突攻击）

    buckets    unsafe.Pointer // 指向 bucket 数组的指针
    oldbuckets unsafe.Pointer // 用于扩容时，保存旧的 buckets
    nevacuate  uintptr        // 扩容时记录迁移的进度

    extra *mapextra           // 仅在需要时使用，如溢出桶等
}
```

### 2. `bmap`：桶结构

每个桶 bucket 存储最多 **8 个 key/value** 对，结构大致如下：

```go
type bmap struct {
    tophash [8]uint8        // 哈希高位 (前8位)，用于快速判断是否相等
    // 后面紧跟的是 keys 和 values（使用偏移访问）
}
```

---

## 二、哈希和定位逻辑

1. **计算哈希值**：
    
    - 对 key 使用哈希函数（带 `hash0` 种子）生成 hash 值。
        
    - 高 8 位存在 `tophash[]`，后面用于快速比较。
        
    - 低 B 位用于定位桶位置（`hash & ((1 << B) - 1)`）。
        
2. **桶内查找**：
    
    - 每个桶最多 8 个元素。
        
    - 若未找到，则沿着溢出桶链继续查找。
        

---

## 三、冲突解决方案

Go 使用“**桶 + 溢出桶链表**”的方式解决冲突：

- 每个桶最多 8 个 kv。
    
- 超出后自动链入溢出桶（overflow bucket）。
    
- bucket 本身带有一个 `tophash` 数组，可加快 key 匹配。
    

---

## 四、扩容机制

扩容条件：

- 当负载因子过高（大约 load factor > 6.5）时，map 自动扩容。
    

扩容方式：

- **渐进式扩容**（incremental rehash）：
    
    - 新建一组更大的桶。
        
    - 插入或访问时，按需将旧桶逐渐迁移（从 oldbuckets -> buckets）。
        
    - 避免一次性 rehash 导致性能抖动。
        

---

## 五、遍历的随机性

为了防止用户依赖遍历顺序（避免构建依赖行为），Go 遍历 `map` 时是随机顺序：

- 每次 `range map` 使用不同的起始桶位置。
    
- 同时桶内遍历顺序也是乱序。
    

---

## 六、map 是非线程安全的

- 多个 goroutine 并发读写 `map` 会发生竞态。
    
- 解决方式：使用 `sync.Map` 或加锁（`sync.RWMutex`）。
    

---

## 七、性能总结

|操作|平均时间复杂度|
|---|---|
|插入|O(1)|
|查找|O(1)|
|删除|O(1)|
|遍历|O(n)|

---

## 示例说明

```go
m := make(map[string]int)
m["a"] = 1  // 哈希后进入某个桶
m["b"] = 2  // 冲突时进同桶或溢出桶
```

你可以通过如下方式查看 map 的桶数量：

```go
fmt.Printf("map size: %d\n", len(m)) // 逻辑长度
```

但底层桶的容量只能通过 `reflect` 或 `unsafe` 查看（不建议）。

# 定义
在计算机科学里，被称为相关数组、map、符号表或者字典，是由一组 `<key, value>` 对组成的抽象数据结构，，并且同一个 key 只会出现一次。

和 map 相关的操作主要是：

1. 增加一个 k-v 对 —— Add or insert；
2. 删除一个 k-v 对 —— Remove or delete；
3. 修改某个 k 对应的 v —— Reassign；
4. 查询某个 k 对应的 v —— Lookup；

map 的设计也被称为 “The dictionary problem”，它的任务是设计一种数据结构用来维护一个集合的数据，并且可以同时对集合进行增删查改的操作。最主要的数据结构有两种：`哈希查找表（Hash table）`、`搜索树（Search tree）`。

哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。在很多场景下，哈希查找表的性能很高。

哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：`链表法`和`开放地址法`。`链表法`将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。`开放地址法`则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树。

自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。还有一点，遍历自平衡搜索树，返回的 key 序列，一般会按照从小到大的顺序；而哈希查找表则是乱序的。

# map 底层实现
前面说了 map 实现的几种方案，Go 语言采用的是哈希查找表，并且使用链表解决哈希冲突。

## map 内存模型
在源码中，表示 map 的结构体是 hmap，它是 hashmap 的“缩写”：
```golang
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
	count     int
	flags     uint8
	// buckets 的对数 log_2
	B         uint8
	// overflow 的 bucket 近似数
	noverflow uint16
	// 计算 key 的哈希的时候会传入哈希函数
	hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer
	// 等量扩容的时候，buckets 长度和 oldbuckets 相等
	// 双倍扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成
	nevacuate  uintptr
	extra *mapextra // optional fields
}
```

说明一下，`B` 是 buckets 数组的长度的对数，也就是说 buckets 数组的长度就是 2^B。bucket 里面存储了 key 和 value，后面会再讲。

buckets 是一个指针，最终它指向的是一个结构体：
```
type bmap struct{
	tophash [bucketCnt]uint8
}
```
但这只是表面(src/runtime/hashmap.go)的结构，编译期间会给它加料，动态地创建一个新的结构：
```golang
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```
`bmap` 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。

来一个整体的图：
![[Pasted image 20250506120650.png]]
bmap 是存放 k-v 的地方，我们把视角拉近，仔细看 bmap 的内部组成。
![[Pasted image 20250506120823.png]]
上图就是 bucket 的内存模型，`HOB Hash` 指的就是 top hash。 注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 `overflow` 指针连接起来。

## 哈希函数
map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 `alginit()` 中完成，位于路径：`src/runtime/alg.go` 下。

hash 函数，有加密型和非加密型。 加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种； 非加密型的一般就是查找。在 map 的应用场景中，用的是查找。 选择 hash 函数主要考察的是两点：性能、碰撞概率。

## key 为什么是无序的
map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 2^B）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

当然，如果我就一个 hard code 的 map，我也不会向 map 进行插入删除的操作，按理说每次遍历这样的 map 都会返回一个固定顺序的 key/value 序列吧。的确是这样，但是 Go 杜绝了这种做法，因为这样会给新手程序员带来误解，以为这是一定会发生的事情，在某些情况下，可能会酿成大错。

当然，Go 做得更绝，当我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。

多说一句，“迭代 map 的结果是无序的”这个特性是从 go 1.0 开始加入的。


