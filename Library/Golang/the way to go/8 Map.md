# 8.1 声明、初始化和 make
`map` 是 **引用类型** 的： 内存用 `make()` 方法来分配。

`map` 的初始化：`var map1 = make(map[keytype]valuetype)`。或者简写为：`map1 := make(map[keytype]valuetype)`。

`mapCreated` 就是用这种方式创建的：`mapCreated := make(map[string]float32)`。

相当于：`mapCreated := map[string]float32{}`。

**不要使用 `new()`，永远用 `make()` 来构造 `map`**

**注意** 如果你错误地使用 `new()` 分配了一个引用对象，你会获得一个空引用的指针，相当于声明了一个未初始化的变量并且取了它的地址：

```go
mapCreated := new(map[string]float32)
```

接下来当我们调用：`mapCreated["key1"] = 4.5` 的时候，编译器会报错：

```
invalid operation: mapCreated["key1"] (index of type *map[string]float32).
```

值可以是任意类型的，这里给出了一个使用 `func() int` 作为值的 `map`，也可以用切片作为 map 的值。
```go
mp1 := make(map[int][]int)
mp2 := make(map[int]*[]int)
```

# 8.2 测试键值对是否存在及删除元素
为了解决这个问题，我们可以这么用：`val1, isPresent = map1[key1]`

`isPresent` 返回一个 `bool` 值：如果 `key1` 存在于 `map1`，`val1` 就是 `key1` 对应的 `value` 值，并且 `isPresent` 为 `true`；如果 `key1` 不存在，`val1` 就是一个空值，并且 `isPresent` 会返回 `false`。

如果你只是想判断某个 `key` 是否存在而不关心它对应的值到底是多少，你可以这么做：

```go
_, ok := map1[key1] // 如果key1存在则ok == true，否则ok为false
```

或者和 `if` 混合使用：

```go
if _, ok := map1[key1]; ok {
	// ...
}
```

从 `map1` 中删除 `key1`：

直接 `delete(map1, key1)` 就可以。

如果 `key1` 不存在，该操作不会产生错误。

译者注：map 的本质是散列表，而 map 的增长扩容会导致重新进行散列，这就可能使 map 的遍历结果在扩容前后变得不可靠，Go 设计者为了让大家不依赖遍历的顺序，每次遍历的起点--即起始 bucket 的位置不一样，即不让遍历都从某个固定的 bucket0 开始，所以即使未扩容时我们遍历出来的 map 也总是无序的。

# 8.5 map 的排序
`map` 默认是无序的，不管是按照 key 还是按照 value 默认都不排序

如果你想为 `map` 排序，需要将 key（或者 value）拷贝到一个切片，再对切片排序（使用 `sort` 包）。

# 8.6 将 map 的键值对调
如果 `map` 的值类型可以作为 key 且所有的 value 是唯一的，那么通过下面的方法可以简单的做到键值对调。