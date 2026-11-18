---
id: 027b79ce-a72a-4a81-9f3a-2cb025024d62
anki: true
---

# Go 的 defer 执行顺序是什么？

## 概念索引

- 注册: defer 语句把调用压入延迟栈;
- 执行: 当前函数返回前按后进先出执行;
- 求值: 参数在注册时求值;
- 闭包: 函数体读取外层变量，值在调用时确定;
- 返回值: 具名返回值可被 defer 修改;

## defer 按什么顺序执行？

```go
func order() {
    defer fmt.Println("first registered")
    defer fmt.Println("second registered")
    defer fmt.Println("third registered")
    fmt.Println("body")
}
// 输出：
// body
// third registered
// second registered
// first registered
```

| 步骤 | 行为 |
| --- | --- |
| 注册 | 依次压入延迟栈 |
| 函数返回前 | 从栈顶弹出执行 |
| 顺序 | LIFO，后注册先执行 |

## 参数在什么时候求值？

```go
func main() {
    n := 1
    defer fmt.Println(n) // 注册时立即求值为 1
    n = 2
}
// 输出 1，不是 2
```

- 普通参数: 注册 defer 时求值;
- 闭包: 函数体在真正执行时才读取外层变量;
- 方法值: 接收者在注册时求值;
- 需要延迟读取时，用闭包包一层;

## defer 如何影响返回值？

```go
func count() (n int) {
    defer func() { n++ }()
    return 5
}
// 返回 6
```

- 具名返回值是函数作用域变量;
- defer 在 return 赋值之后、函数真正返回之前执行;
- 可以读取并修改具名返回值;
- 匿名返回值没有可修改的名字，defer 改不到;

## 循环里使用 defer 要注意什么？

```go
for _, f := range files {
    defer f.Close() // 全部堆到函数返回才关闭
}
```

- defer 绑定整个函数，不绑定循环体;
- 循环里反复注册会堆积资源;
- 修正: 把循环体抽成函数，或显式关闭;
- panic 场景: defer 仍会执行，可用于 recover 与清理;
