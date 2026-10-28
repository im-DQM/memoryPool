# v3 测试用例与结果

## 测试环境

- Linux（WSL2 Ubuntu），GCC，C++17
- 编译选项：`-O2`
- 性能数据为单机实测，多次运行取代表值

## 构建与运行

使用 CMake：

```bash
cd v3
mkdir build && cd build
cmake ..
make
./unit_test   # 功能测试
./perf_test   # 性能测试
```

不使用 CMake 时可直接编译：

```bash
cd v3
g++ -std=c++17 -O2 -Iinclude src/*.cpp tests/UnitTest.cpp -o unit_test -pthread
g++ -std=c++17 -O2 -Iinclude src/*.cpp tests/PerformanceTest.cpp -o perf_test -pthread
```

## 功能测试（UnitTest.cpp）

| 用例 | 覆盖内容 | 结果 |
| --- | --- | --- |
| `testBasicAllocation` | 8B、1KB、1MB（超过 `MAX_BYTES`）的分配与释放 | 通过 |
| `testMemoryWriting` | 分配 128B 后写入并逐字节校验 | 通过 |
| `testMultiThreading` | 4 线程 × 1000 次，8B–2KB 随机分配与随机释放 | 通过 |
| `testEdgeCases` | 0 字节、1 字节对齐、`MAX_BYTES`、`MAX_BYTES + 1` | 通过 |
| `testStress` | 10000 次随机大小分配，随机顺序释放 | 通过 |

运行结果：**全部通过**（`All tests passed successfully!`）。

## 性能测试（PerformanceTest.cpp）

运行前先用 1000 轮 × 5 种尺寸（32B–512B）预热分配器。加速比 = `new/delete` 耗时 ÷ 内存池耗时。

| 场景 | 操作规模 | new/delete 耗时 | 内存池耗时 | 加速比 |
| --- | --- | --- | --- | --- |
| 固定大小小对象分配 | 单线程，100000 次，每次 32B | — | — | **2.89×** |
| 多线程混合分配 | 4 线程 × 25000 次，随机 8B–256B | — | — | **1.53×** |
| 混合大小分配 | 单线程，100000 次，随机 16B–2KB（8 种尺寸） | — | — | **1.27×** |

说明：

- 固定 32B 场景收益最大，因为分配/释放几乎全部命中线程本地自由链表，没有系统调用。
- 多线程场景下 `new/delete` 存在 glibc arena 竞争，但内存池仍需线程缓存与中心缓存之间的批量搬运，收益相对缩小。
- 混合大小场景涉及多个尺寸类的批量获取，且大块最终仍由页缓存管理，加速比最低。
- 性能数据受 CPU、内存带宽、系统负载影响，不同环境结果会有波动，建议在本机重新运行 `perf_test` 取多次结果。
