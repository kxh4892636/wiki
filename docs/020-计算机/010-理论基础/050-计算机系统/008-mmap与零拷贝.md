---
id: a63f064f-1436-4ad7-9781-03e4ac615c5d
anki: true
---

# mmap 与零拷贝分别解决什么问题？

## 常见问题

- 简述 mmap 的使用场景以及原理
- 简述 Linux 零拷贝的原理

## 概念索引

- mmap: 把文件或匿名内存映射进进程地址空间;
- VMA: 内核描述一段虚拟地址区间的结构;
- page cache: 内核缓存文件内容的页;
- 零拷贝: 减少 CPU 参与的数据复制;
- sendfile: 内核内直接把文件页送到 socket;
- DMA: 设备直接读写内存，不占 CPU;

## mmap 的原理是什么？

- 惰性: `mmap` 只在进程里登记一段 VMA，既不分配物理页也不读文件;
- 缺页: 首次访问触发缺页，内核把文件对应页读进 page cache 并填页表;
- 共享: `MAP_SHARED` 的写会落到 page cache，最终回写磁盘并有 `msync`;
- 私有: `MAP_PRIVATE` 靠写时复制隔离，改动不落到磁盘;
- 回收: 映射页属于文件页，内存紧张时可直接丢弃，重读即可;

```c
int fd = open("data.bin", O_RDWR);
char *p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
p[0] = 'x';                  // 写 page cache，由内核回写
msync(p, len, MS_SYNC);      // 需要立刻落盘时调用
munmap(p, len);
```

- 对照: `read` 要把数据复制进用户缓冲区，`mmap` 只改页表，按页访问;

## mmap 适合哪些场景？

- 大文件随机读写: 省掉用户缓冲区，按页按需载入;
- 进程间共享: `MAP_SHARED` 加同一文件，或 `MAP_ANONYMOUS` 加 `fork`;
- 动态库加载: 多个进程共享同一份只读代码页;
- 内存分配: `malloc` 对大块内存直接用匿名映射;
- 设备映射: 把设备寄存器映射进用户态，减少系统调用;
- 限制: 小文件频繁映射、需要顺序流式处理时不如 `read` 简单;

## 传统读写为什么慢？

```text
read + write 一次文件转发:
  磁盘 -> 内核缓冲区 (DMA) -> 用户缓冲区 (CPU) -> socket 缓冲区 (CPU) -> 网卡 (DMA)
  4 次复制, 其中 2 次占 CPU; 4 次用户态与内核态切换
```

- 复制: 数据在磁盘、内核页、用户缓冲、socket 缓冲之间来回搬;
- 切换: `read` 与 `write` 各两次切换，小包场景开销明显;
- 结论: 转发类场景里，用户缓冲区几乎没被真正使用，属于白搬;

## 零拷贝有哪几种实现？

| 方式 | 复制次数 | 切换次数 | 适用 |
| --- | --- | --- | --- |
| `read` + `write` | 4 | 4 | 需要处理数据 |
| `mmap` + `write` | 3 | 4 | 用户态要读内容 |
| `sendfile` | 2 | 2 | 文件到 socket 直传 |
| `splice` | 2 | 2 | 经管道在两类 fd 间搬 |
| `sendfile` + SG-DMA | 0 次 CPU 复制 | 2 | 网卡支持分散聚集 |

- 原理: 内核只传页的引用或描述符，不让 CPU 逐字节搬;
- `sendfile`: 内核把 page cache 的页直接挂到 socket 发送队列，见 [sendfile(2)](https://man7.org/linux/man-pages/man2/sendfile.2.html);
- 限制: 传输中不能修改数据，所以加密与压缩仍需回用户态;
- 应用: Nginx 静态文件、Kafka 日志发送都默认走 `sendfile`;

```mermaid
flowchart LR
  A["磁盘"] -->|DMA| B["page cache"]
  B -->|只传页引用| C["socket 缓冲"]
  C -->|DMA| D["网卡"]
```

- 取舍: 零拷贝换来吞吐，代价是灵活性与错误处理复杂度;
