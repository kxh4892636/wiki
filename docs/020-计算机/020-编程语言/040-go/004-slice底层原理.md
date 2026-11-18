---
id: 210d79ad-aabb-4c35-871e-f369349fdb26
anki: true
---

# Go 的 slice 底层原理是什么？

## 常见问题

- 简述 slice 的底层原理，slice 和数组的区别是什么？

## 概念索引

- 描述符: 指针、长度、容量三部分;
- 底层数组: slice 实际指向的连续内存;
- len: 当前可访问元素数量;
- cap: 从起点可扩展的最大长度;
- 共享: 多个 slice 可指向同一底层数组;

## slice 的描述符是什么？

```go
// 运行时结构示意，不是用户可定义的内置类型
type slice struct {
    array unsafe.Pointer // 底层数组指针
    len   int            // 长度
    cap   int            // 容量
}
```

- slice 是值类型，赋值只复制描述符;
- 描述符里的指针仍指向同一底层数组;
- 修改共享元素，所有引用该数组的 slice 都能看到;

## slice 与数组有什么区别？

| 项 | 数组 | slice |
| --- | --- | --- |
| 长度 | 类型的一部分，固定 | 运行时可变 |
| 赋值 | 复制整个数组 | 复制描述符，共享底层数组 |
| 传参 | 复制整个数组 | 复制描述符 |
| 扩容 | 不支持 | append 可扩容 |
| 比较 | 同长度数组可比较 | 只能与 nil 比较 |

```go
arr := [3]int{1, 2, 3}
s := arr[:]      // 共享 arr 的底层数组
s[0] = 10        // arr[0] 也变成 10
fmt.Println(arr) // [10 2 3]
```

## nil slice 与空 slice 有什么区别？

| 状态 | 是否为 nil | len | cap | append |
| --- | --- | --- | --- | --- |
| var s []int | 是 | 0 | 0 | 可用 |
| `s := []int{}` | 否 | 0 | 0 | 可用 |
| make([]int, 0, 8) | 否 | 0 | 8 | 可用 |

- len == 0 只表示没有元素;
- 与 nil 比较才表示是不是零值 slice;
- JSON 序列化时，nil slice 常输出 null，空 slice 输出 [];

## append 与 copy 的行为是什么？

```go
s := []int{1, 2}
s = append(s, 3) // 容量足够时原地追加
s = append(s, 4, 5)

dst := make([]int, len(s))
n := copy(dst, s) // 复制 min(len(dst), len(src)) 个元素
```

- append 返回新的描述符，必须接收返回值;
- 容量不足时分配新数组并复制旧元素;
- copy 不增加目标长度，允许源和目标重叠;
- 三下标切片 a[low:high:max] 可限制新 slice 的容量;

## 截取为什么会互相影响？

```go
a := []int{0, 1, 2, 3, 4}
b := a[1:4] // b 与 a 共享底层数组
b[0] = 10   // a[1] 同步变化

c := append(b, 99) // 容量足够时可能写回 a 的底层数组
```

- 截取不复制元素，只新建描述符;
- append 在容量足够时复用底层数组，可能覆盖原数据;
- 需要隔离时用 copy 或三下标切片限制容量;
