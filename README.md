# linux-system-programming

用 C++ 从零练习 Linux 系统编程，覆盖**进程控制、管道与 IPC、信号、线程与同步、生产者消费者模型、进程池与线程池**。每个目录是一个独立的实验，代码配合详细中文注释，记录从用户态接口到并发编程的学习过程。

## 目录结构

```
.
├── MiniShell/      # 迷你 shell：fork / exec / 重定向 / 内建命令
├── testpipe/       # 匿名管道：父子进程通信
├── fifo/           # 命名管道 (FIFO)：服务器 / 客户端
├── shm/            # System V 共享内存 + 命名管道
├── signal/         # 信号：阻塞、未决、自定义 kill 工具
├── thread/         # 线程封装类 + 线程基础示例
├── testmutex/      # 互斥锁：多线程抢票
├── testcond/       # 条件变量：线程唤醒
├── Cond/           # Mutex / LockGuard / Cond 封装（头文件库）
├── BlockQueue/     # 阻塞队列：互斥锁 + 条件变量（生产者消费者）
├── RingQueue/      # 环形队列：信号量（生产者消费者）
├── Process_Pool/   # 进程池：管道 + fork 任务分发
└── ThreadPool/     # 线程池：单例 + 任务队列 + 日志
```

## 模块说明

### MiniShell —— 进程控制与进程替换
一个可交互的最小 shell，是整套代码的起点。

- 命令解析：`strtok` 按空格切分，存储到 `gargv[]`
- 内建命令：`cd`（支持 `~`）、`echo`（支持 `$?`、`$VAR` 环境变量展开）
- 外部命令：`fork` + `execvp` + `waitpid`，并记录退出码 `lastcode`
- 重定向：`<` 输入重定向、`>` 输出覆盖、`>>` 追加（`dup2` 实现）
- 环境变量：从 `environ` 拷贝并 `putenv` 管理

### testpipe —— 匿名管道
`pipe()` 创建管道，`fork()` 后子进程关闭读端负责写，父进程关闭写端负责读，演示匿名管道在父子进程间的单向通信。

### fifo —— 命名管道 (FIFO)
`mkfifo` 创建命名管道文件，实现无亲缘关系的进程间通信：

- `server.cc`：以只读方式打开管道，循环读取客户端数据
- `client.cc`：以只写方式打开管道，从标准输入读入并写入
- `comm.hpp`：`NamedFifo`（创建/销毁管道）与 `FileOp`（读写封装）

### shm —— System V 共享内存 + 命名管道
共享内存是最快的 IPC 方式：

- `shm.hpp`：`ftok` + `shmget` + `shmat` + `shmctl`，区分 `CREATOR`（`IPC_CREAT | IPC_EXCL`）与 `USER`（`IPC_CREAT`）两种角色
- `server.cc`：创建者，通过命名管道收到通知后读取共享内存
- `client.cc`：使用者，向共享内存写数据并用命名管道通知服务器

### signal —— 信号
- `testsig.cc`：`sigprocmask` 屏蔽 2 号信号（SIGINT），观察 `sigpending` 未决位图，10 秒后解除屏蔽触发自定义 `handler`
- `mykill.cc`：命令行工具 `./mykill <信号> <pid>`，封装 `kill()` 发送信号

### thread —— 线程封装
`Thread.hpp` 用 C++ 类封装 `pthread`：`Start`（`pthread_create`）、`Join`（`pthread_join`）、`Detach`（`pthread_detach`）、`Stop`（`pthread_cancel`），通过 `std::function` 传入线程体。`main.cc` 演示线程与主线程并发递增共享变量。

### testmutex —— 互斥锁
`pthread_mutex` 保护共享的 `ticket` 变量，4 个线程并发抢票，演示互斥锁解决数据竞争。

### Cond —— 同步原语封装（头文件库）
- `Mutex.hpp`：`Mutex`（RAII 封装 `pthread_mutex`）+ `LockGuard`（构造加锁、析构解锁）
- `Cond.hpp`：`Cond`（封装 `pthread_cond_wait/signal/broadcast`）

供 `ThreadPool` 等模块复用。

### testcond —— 条件变量
5 个线程阻塞在 `pthread_cond_wait` 上，主线程每秒 `pthread_cond_signal` 唤醒一个，演示条件变量的等待与唤醒。

### BlockQueue —— 阻塞队列
`std::queue` + 互斥锁 + 两个条件变量（生产者/消费者）实现有界阻塞队列：

- 队列满时生产者阻塞，队列空时消费者阻塞
- 用 `_psleep_cnt` / `_csleep_cnt` 记录睡眠线程数，按需精准唤醒
- `main.cc`：20 个生产者 + 30 个消费者并发读写

### RingQueue —— 环形队列（信号量）
`std::vector` 实现环形缓冲区，用两个信号量（`_blank_sem` 空格数 / `_data_sem` 数据数）控制生产者消费者节奏，`TaskManager` 提供下载/上传/SQL 三类任务，2 生产 + 3 消费。

### Process_Pool —— 进程池
`fork` 创建 N 个子进程，每个子进程通过独立管道与父进程相连：

- `Channel` 封装管道写端与子进程 pid，`ChannelManager` 轮询（`Select`）分发
- 父进程写任务码，子进程 `read` 后由 `TaskManager::Execute` 执行
- `Stop` 关闭所有写端、`waitpid` 回收全部子进程

### ThreadPool —— 线程池
最完整的实现，综合运用了前面的全部知识：

- **线程池**：固定数量线程 + `std::queue` 任务队列，条件变量实现线程睡眠/唤醒
- **单例**：`GetInstance` 双重检查锁定（`LockGuard` + 静态成员）
- **日志系统** `log.hpp`：策略模式（控制台 / 文件两种 `LogStretage`），`LOG(level) << ...` 流式输出，含时间/等级/pid/文件/行号
- `TaskManager`：随机分发下载/上传/SQL 任务

## 编译运行

所有模块依赖 **Linux 环境**（`pthread`、`unistd.h`、`sys/*` 头文件），请使用 `g++` 在 Linux 下编译。

| 目录 | 编译 | 运行 | 说明 |
|------|------|------|------|
| MiniShell | `make` | `./myshell` | 交互式 shell |
| testpipe | `make` | `./testpipe` | 父子进程通信 |
| fifo | `make` | 终端1 `./server`，终端2 `./client` | 命名管道 |
| shm | `make` | 终端1 `./server`，终端2 `./client` | 共享内存 |
| signal | `make` | `./testsig`、`./mykill <信号> <pid>` | 信号测试 |
| thread | `make thread` | `./thread` | 线程封装 |
| testmutex | `make testmutex` | `./testmutex` | 抢票 |
| testcond | `make test` | `./test` | 条件变量 |
| BlockQueue | `make cp` | `./cp` | 阻塞队列 |
| RingQueue | `make rq` | `./rq` | 环形队列 |
| Process_Pool | `make` | `./ProcessPool` | 进程池 |
| ThreadPool | `make` | `./main` | 线程池（需 C++17） |

> 注：`Cond/` 为头文件库，无独立可执行程序；`ThreadPool` 使用 `std::filesystem`，需 `-std=c++17`（已在 Makefile 中配置）。清理产物用 `make clean`。

## 学习路线

按依赖关系建议的阅读顺序：

1. **进程**：`MiniShell` → `testpipe` → `fifo` → `shm`
2. **信号**：`signal`
3. **线程与同步**：`thread` → `testmutex` → `Cond` → `testcond`
4. **生产者消费者**：`BlockQueue`（互斥锁+条件变量）→ `RingQueue`（信号量）
5. **综合应用**：`Process_Pool`（进程维度）→ `ThreadPool`（线程维度）

## 环境要求

- Linux（Ubuntu / CentOS 等）
- `g++`（支持 C++11，ThreadPool 需 C++17）
- `make`
