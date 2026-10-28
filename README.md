# memoryPool · C++ 高并发内存池

一个用 C++17 写的线程安全内存池，目标是高频小对象分配场景：把 `new/delete` 的开销从"每次都要走分配器内部逻辑、必要时进内核"
变成"在线程本地链表上摘一个节点"。仓库按 v1 → v3 三个版本递进，每一版都保留完整代码，方便对照阅读。

## 为什么做这个

- `new/delete`、`malloc/free` 每次都要走分配器的查找与合并逻辑，小对象高频分配时这部分开销占比很高；
- 多线程下 glibc 的 arena 还会出现锁竞争；
- 内存池预取大块、按需切分，把管理逻辑搬到应用层，分配/释放变成链表操作，同时减少碎片。

## 三个版本

| 版本 | 内容 | 规模 |
|---|---|---|
| **v1** | 单层内存池：`MemoryPool` 直接管理固定块，验证"预分配 + 复用"的基本收益，也是后两版的对照组 | ~165 行 |
| **v2** | 拆成三层：`ThreadCache`（线程本地）、`CentralCache`（按大小类共享）、`PageCache`（按页管理），补上功能与性能测试 | ~600 行 |
| **v3** | 三层结构完善版：慢开始增长与延迟归还阈值、按页数组织的 Span 链表与「页 → Span」映射、更完整的用例与测试说明 | ~700 行 |

## 架构与关键设计

```
应用 ──► ThreadCache（每线程一份，thread_local，无锁）
              │ 批量搬运（不够时）
              ▼
        CentralCache（按大小类分桶，原子链表 + 每桶自旋锁）
              │ 按页申请 / 归还
              ▼
        PageCache（Span 链表，按页数管理，可向系统要页）
```

- **ThreadCache**：`thread_local` 单例，每个尺寸类一条自由链表，分配/释放只动本线程链表，天然无锁；
  一次向 CentralCache 批量要一批（`getBatchNum`，慢开始策略，避免一上来搬一大块），
  链表长度超过阈值再整批还回（`shouldReturnToCentralCache`），避免线程囤积内存。
- **CentralCache**：按大小类分桶，桶内是原子链表 + 每桶自旋锁；跨线程搬运在这一层完成，
  锁粒度只到「某个尺寸类」，不锁全局。
- **PageCache**：以页为单位管理 Span，按页数挂到不同链表；维护 `spanMap_`（地址 → Span）用于归还时定位，
  并合并相邻 Span，减少外部碎片。
- **对齐与边界**：8 字节对齐；小于 `MAX_BYTES` 走池子，更大的直接交给系统；Span 延迟归还，避免频繁向系统要页/还页。

## 实测数据

环境：Intel i7-13650HX（14 核 20 线程），WSL2 Ubuntu，GCC，`-O2`。
加速比 = `new/delete` 耗时 ÷ 内存池耗时，多次运行取代表值（完整用例见 `v3/tests/README.md`）。

| 场景 | 规模 | 加速比 |
|---|---|---|
| 固定大小小对象 | 单线程 100000 次，每次 32B | **2.89×** |
| 多线程混合 | 4 线程 × 25000 次，随机 8B–256B | **1.53×** |
| 混合大小 | 单线程 100000 次，随机 16B–2KB（8 种尺寸） | **1.27×** |

解读：

- 32B 固定大小收益最大——几乎全部命中线程本地链表，没有系统调用、没有锁；
- 多线程场景要在「线程缓存 ↔ 中心缓存」之间批量搬运，收益收窄；
- 混合大小涉及多个尺寸类，较大块最终仍由页缓存管理，加速比最低。

## 构建与运行

以 v3 为例（v1 / v2 目录结构相同）：

```bash
cd v3
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/unit_test     # 功能测试
./build/perf_test     # 性能测试
```

不用 CMake 也可以直接编：

```bash
cd v3
g++ -std=c++17 -O2 -Iinclude src/*.cpp tests/UnitTest.cpp -o unit_test -pthread
g++ -std=c++17 -O2 -Iinclude src/*.cpp tests/PerformanceTest.cpp -o perf_test -pthread
```

## 目录结构

```
.
├── v1/            单层内存池（基线）
├── v2/            三层缓存版
├── v3/            三层完善版 + 测试说明（推荐从这里读）
│   ├── include/   ThreadCache / CentralCache / PageCache / Common
│   ├── src/
│   └── tests/     UnitTest.cpp / PerformanceTest.cpp / README.md
└── images/        各版本的测试与性能截图
```

## 测试覆盖（v3）

| 用例 | 覆盖 |
|---|---|
| `testBasicAllocation` | 8B / 1KB / 1MB（超 `MAX_BYTES`）分配与释放 |
| `testMemoryWriting` | 128B 分配后写入并逐字节校验 |
| `testMultiThreading` | 4 线程 × 1000 次随机分配释放 |
| `testEdgeCases` | 0 字节、1 字节、`MAX_BYTES`、`MAX_BYTES + 1` |
| `testStress` | 10000 次随机大小分配，随机顺序释放 |

## 还可以继续做

- 加 `-fsanitize=address` 的 Debug 构建配置，把越界与悬垂指针扫一遍；
- 与 `tcmalloc` / `jemalloc` 同场景对比，而不只是和 `new/delete` 比；
- 中心缓存尝试无锁结构，量化锁粒度对 4 线程以上场景的影响；
- 补更贴近真实负载的测试（对象大小分布不均、生命周期参差）。
