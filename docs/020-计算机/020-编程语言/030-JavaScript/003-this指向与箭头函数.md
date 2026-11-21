---
id: 95e89b5d-fa25-4499-90ca-d0c66cc2b3a9
anki: true
---

# this 由什么决定, 箭头函数又特殊在哪?

## 常见问题

- 简述 Javascript 中 this 的指向有哪些
- 箭头函数和普通函数的区别是什么？

## 概念索引

- 调用点: 函数被调用的那一处表达式, 它决定 this;
- 默认绑定: 裸调用时落到全局对象或 undefined;
- 隐式绑定: 写成 `obj.fn()` 时 this 就是 obj;
- 显式绑定: call、apply、bind 直接指定 this;
- new 绑定: 构造调用时 this 是新创建的对象;
- 词法 this: 箭头函数不参与上面四条, 直接取外层;

## this 在什么时候确定?

- 时机: 调用时写入执行上下文的 ThisBinding, 定义时不看;
- 依据: 只看调用表达式, 也就是括号左边是什么;
- 严格模式: 只影响默认绑定的落点, 不影响其它三条;
- 推论: 同一个函数换一种调用方式, this 就换一个;

```js
function who() { console.log(this.name); }
const obj = { name: "obj", who };

obj.who();          // "obj"; 隐式绑定
const f = obj.who;  // 方法被摘出来
f();                // 严格模式抛错; 非严格模式看全局 name
```

## 四条绑定规则分别是什么?

| 规则 | 调用形式 | this |
| --- | --- | --- |
| 默认绑定 | `f()` | 严格模式 undefined, 否则全局对象 |
| 隐式绑定 | `obj.f()` | obj |
| 显式绑定 | `f.call(o)` | o |
| new 绑定 | `new F()` | 新创建的对象 |

- 优先级: new 高于显式, 显式高于隐式, 隐式高于默认;
- 箭头函数: 不参与这套规则, call 与 bind 都改不动它;

```js
const o = { name: "o", f() { return this.name; } };
o.f.call({ name: "p" }); // "p"; 显式绑定赢过隐式

function F() { this.tag = "new"; }
new F().tag;             // "new"; new 绑定赢过一切
```

## 箭头函数的 this 从哪来?

- 本质: 箭头函数没有自己的 ThisBinding, 沿词法环境向外找最近的 this;
- 结果: 取定义时所在作用域的 this, 之后怎么调用都不变;
- 对比 bind: bind 是运行时造一个绑定版新函数, 箭头函数是定义时就捕获;
- 典型用途: 回调里要沿用外层 this 时, 不必再写 `const that = this`;

```js
const group = {
  title: "前端",
  list: ["a", "b"],
  show() {
    this.list.forEach((x) => console.log(this.title + x));
  },
};
group.show(); // 前端a 前端b; 箭头函数拿到 show 的 this
```

## 箭头函数和普通函数的区别?

| 项 | 普通函数 | 箭头函数 |
| --- | --- | --- |
| this | 调用时绑定 | 取外层, 固定不变 |
| arguments | 有自己的 | 取外层的, 没有自己的 |
| new | 可作构造函数 | 不能 new |
| prototype | 有 | 没有 |
| super | 可用 | 继承外层 |
| 常见用途 | 对象方法、构造函数 | 短回调、需要外层 this |

- 不能 new: 没有 [[Construct]], 调用即抛 TypeError;
- 没有 prototype: 函数对象上根本不存在这个属性;
- 不适合: 对象方法、构造函数、需要转发 arguments 的包装器;

```js
const f = () => {};
new f();     // TypeError; f is not a constructor
f.prototype; // undefined
```

## 方法被借出去后 this 怎么丢?

- 场景一: `const f = obj.method` 之后再裸调 f(), 隐式绑定消失;
- 场景二: 把方法直接传给 forEach、setTimeout, 同样是裸调用;
- 修法一: f.bind(obj) 造一个 this 固定的新函数;
- 修法二: 定义处改用箭头函数, 让 this 跟随外层;
- 修法三: 调用处写 obj.method.call(obj) 或外包一层箭头;

```js
const obj = { name: "obj", getName() { return this.name; } };
const f = obj.getName;
f();         // undefined; 丢了隐式绑定
f.call(obj); // "obj"
```
