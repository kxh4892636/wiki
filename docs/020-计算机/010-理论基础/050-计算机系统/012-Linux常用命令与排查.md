---
id: ca8ad5d8-1cef-45a0-be3e-7a5c887fae70
anki: true
---

# 线上排查该用哪些 Linux 命令？

## 常见问题

- 简述几个常用的 Linux 命令以及他们的功能
- Linux 下如何查看端口被哪个进程占用？
- Linux 下如何排查 CPU 以及 内存占用过多？
- Linux 下如何查看 CPU 荷载，正在运行的进程，某个端口对应的进程？
- Linux 如何查看实时的滚动日志？

## 概念索引

- 进程视图: `ps` 与 `top` 给出瞬时快照;
- 端口归属: 从 socket 反查持有它的进程;
- 负载: 运行队列与不可中断任务的平均长度;
- 采样: `pidstat` 与 `perf` 按时间窗口观察;
- 日志流: 跟随文件尾部持续输出;
- 排查顺序: 先定资源类型，再定位进程，最后看线程与调用栈;

## 常用命令各自负责什么？

| 命令 | 作用 | 常用参数 |
| --- | --- | --- |
| `top` | 实时进程与整体资源 | `-H` 看线程，`-p` 看指定进程 |
| `ps` | 进程快照 | `aux`、`-eo pid,ppid,stat,pcpu,comm` |
| `ss` | socket 统计与连接 | `-lntp` 监听端口加进程 |
| `lsof` | 打开的文件与端口 | `-i :8080`、`-p <pid>` |
| `df` / `du` | 磁盘容量与目录占用 | `-h`、`--max-depth=1` |
| `free` | 内存与交换区 | `-m`、`-h` |
| `vmstat` | 采样 CPU、内存、IO、切换 | `vmstat 1 5` |
| `iostat` | 磁盘吞吐与等待 | `-x 1` |
| `journalctl` | systemd 日志 | `-u 服务名 -f` |

- 组合: `top` 找大头，`ps` 精确过滤，`ss` 与 `lsof` 定位连接;
- 注意: 容器里 `top` 看到的是宿主机数据，要进对应 namespace;

## 如何查看端口被哪个进程占用？

- 首选: `ss -lntp` 列出监听端口与进程，`-p` 需要 root;
- 查连接: `ss -tnp state established '( sport = :8080 )'`;
- 备选: `lsof -i :8080` 或 `fuser -n tcp 8080` 直接给出进程;
- 内核: `/proc/net/tcp` 里端口是十六进制，再按 inode 反查 `/proc/*/fd`;
- 老系统: `netstat -tunlp` 与 `ss` 同源，但已不推荐;

```bash
ss -lntp | grep 8080
lsof -i :8080
fuser -n tcp 8080
```

- 找不到: 端口属于别的 namespace 或已被 TIME_WAIT 占用，用 `ss -tan` 看状态;

## 如何排查 CPU 占用过高？

- 定位进程: `top` 按 P 排序，或 `ps -eo pid,pcpu,comm --sort=-pcpu | head`;
- 定位线程: `top -H -p <pid>`，或 `ps -L -p <pid> -o tid,pcpu`;
- 看调用栈: `perf top -p <pid>` 抽样，`gdb -p <pid>` 或 `jstack` 抓栈;
- 看趋势: `pidstat -u -t -p <pid> 1` 区分用户态与内核态;
- 判内核态高: `perf top` 里多为 `sys` 或 `softirq`，要查系统调用与网卡;
- 常见原因: 死循环、正则回溯、GC 频繁、锁自旋、软中断打满;

```mermaid
flowchart LR
  A["top 找高 CPU 进程"] --> B["top -H 找热点线程"]
  B --> C["perf top 或抓栈"]
  C --> D["区分用户态与内核态"]
  D --> E["回到代码或内核路径"]
```

## 如何排查内存占用过高？

- 全局: `free -m` 看 `available`，别只看 `free`;
- 进程: `ps -eo pid,rss,vsz,comm --sort=-rss | head`，`RSS` 含共享库，多进程相加会重复;
- 明细: `pmap -x <pid>` 或 `/proc/<pid>/smaps` 按段看匿名页与文件页;
- 换页: `vmstat 1` 的 `si/so` 持续非零说明在换页;
- 缓存: page cache 占满是正常现象，可回收但不算泄漏;
- 泄漏: 业务平稳而 RSS 单调上涨，用堆分析工具比对快照;

## 如何查看 CPU 荷载与实时日志？

- 荷载: `uptime` 或 `cat /proc/loadavg`，给出 1、5、15 分钟平均值;
- 判读: 荷载约等于可运行加不可中断任务数，除核数超过 1 才算排队，CPU 空而荷载高多是 D 状态;
- 核数: `nproc` 或 `lscpu`，容器里要看 cgroup 的 quota;
- 运行进程: `ps aux` 全量，`ps -ef` 带父进程，`pstree -p` 看层级;
- 滚动日志: `tail -f app.log`，`less +F app.log` 可暂停回看;
- 系统日志: `journalctl -u 服务名 -f`，`dmesg -w` 跟内核日志，容器 OOM 记录也在这里;

```bash
uptime                      # load average: 0.52, 0.58, 0.59
nproc                       # 核数
tail -f /var/log/app.log    # 实时滚动
less +F /var/log/app.log    # F 跟随, Ctrl+C 暂停后可搜索
```

