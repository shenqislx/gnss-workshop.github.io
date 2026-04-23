---
title: "嵌入式无堆动态数据结构：定长数组、Slot Registry 与 CTZ"
layout: article
date: 2026-04-23
category: "嵌入式软件"
author: "Andy"
summary: "如何在嵌入式平台上用定长数组、slot registry 和 ctz/bitmap 机制实现有边界的动态数据结构，兼顾实时性、内存上界和遍历效率。"
---

最近在优化 GNSS 嵌入式软件，之前的项目里过于追求低存储占用，忽视了代码运行效率。

我为原本的数据结构中增加了双向查找表，稍微牺牲一点 memory，换取倍数级别的访问效率提升。

通过 bitmask 以及硬件平台支持的 CTZ 指令，实现数据成员的快速索引和更新。

在展开描述之前，我想先聊聊嵌入式软件的数据结构。

为什么很多底层嵌入式软件项目中严格禁止动态数据分配（Heap malloc）？

# 嵌入式无堆动态数据结构：定长数组、Slot Registry 与 CTZ

嵌入式平台并不是不能做“动态数据结构”。真正危险的不是动态，而是**无边界的动态**：运行期对象数量没有上限、内存碎片不可预测、分配耗时无法估计、失败路径没有被设计过，最后把一个看起来很自然的 `malloc/free` 或链表操作，变成实时系统里最难复现的问题。

在资源受限、长时间运行、调试窗口很小的系统里，数据结构设计的第一目标不是 API 好不好看，而是资源边界能不能证明：

- 最大占用多少 RAM？
- 分配和释放的最坏耗时是多少？
- 遍历活跃对象时会不会扫大量空洞？
- 失败时是返回错误、降级，还是悄悄破坏状态？
- 释放后的句柄还能不能被误用？

我的观点很简单：**嵌入式平台不是不能动态，而是要把动态性约束在固定资源模型内**。一种很实用的组合是：

1.  用定长数组承载对象存储；
2.  动态映射管理：建立一个轻量化的下标映射关系，通过维护有效成员的下标实现动态分配。也称为 slot registry，即管理哪些槽位正在使用；
3.  用 bitmap + ctz 快速索引和管理活跃槽位。

这套方法非常适合**最大容量已知、运行期活跃对象稀疏变化**的场景，比如 GNSS 接收机通道表、协议会话表、消息 buffer、驱动请求队列、订阅表、任务实例池等。

---

## 1. 嵌入式数据结构优化，优化的到底是什么？

很多人讨论数据结构时，第一反应是时间复杂度，比如 O(1)、O(logN)、O(N)。但在嵌入式系统里，仅有平均复杂度是不够的。尤其是实时路径，真正需要关心的是 **WCET（Worst-Case Execution Time，最坏情况执行时间）**。

TLSF 论文在讨论实时动态分配器时，把实时内存分配的关键要求说得很明确：

- 分配和释放的最坏执行时间必须能提前知道，并且不能依赖应用输入数据；
- 同时，长期运行系统还必须控制碎片问题。

也就是说，实时系统要的不是“通常很快”，而是 **“最坏也有界”**。

因此，嵌入式数据结构优化通常要同时看几个维度：

| 维度 | 需要回答的问题 |
| --- | --- |
| 内存上界 | 最多占用多少字节？是否随运行时间增长？ |
| 时间上界 | 分配、释放、查询、遍历的最坏耗时是多少？ |
| 碎片风险 | 是否存在外部碎片？长期运行后是否还能分配？ |
| 遍历成本 | 遍历活跃对象时是否会扫描大量空槽？ |
| 失败模式 | 满容量、重复释放、非法句柄如何处理？ |
| 局部性 | 数据是否连续？cache 和 DMA 访问是否友好？ |

这也是为什么很多嵌入式项目不愿意在实时热路径里直接使用通用 heap。

Embedded 社区对 `malloc()` 的讨论中也提到，嵌入式应用里常见顾虑包括 **不可重入、分配耗时不可预测、分配失败，以及碎片导致“总空闲内存足够但连续空间不够”** 的问题。

文章同时指出，如果应用所需内存块是固定大小或少量固定大小，固定块分配器可以消除碎片并更容易做到确定性。

所以问题不应该是“能不能用动态分配”，而应该是：

> 这个动态分配是否有容量上界、时间上界和可测试的失败路径？

如果答案是否定的，就不要把它放进实时路径。

---

## 2. 定长数组上的动态分配

假设我们在做一个 GNSS 接收机，最多支持 64 个跟踪通道。运行时通道会创建、释放、重新分配，但最大数量是明确的。这种场景没有必要向系统 heap 申请内存：

```c
#define MAX_CHANNELS 64

typedef struct {
    uint8_t  sat_id;
    uint8_t  signal_id;
    uint32_t state;
    float    cn0;
} channel_t;

static channel_t channels[MAX_CHANNELS];
```

这只是静态数组，还不能算“动态”。动态性来自 slot 管理：运行期从 `0..63` 这些槽位里找一个空槽，把它标记为已用；释放时再把槽位还回去。

最小实现可以用 free list：

```c
typedef struct {
    channel_t items[MAX_CHANNELS];
    uint8_t   next_free[MAX_CHANNELS];
    uint8_t   free_head;
    uint8_t   used_count;
} channel_pool_t;
```

通道较多时可以用 bitmap：

```c
typedef struct {
    channel_t items[MAX_CHANNELS];
    uint64_t  used_bits;
    uint8_t   used_count;
} channel_pool_t;
```

两者都不需要 heap。free list 的优势是分配空槽很直接；bitmap 的优势是占用状态紧凑，而且天然适合快速遍历。

实际工程里也可以组合使用：free list 负责快速分配，bitmap 负责快速查询和遍历。

定长数组的核心价值不是“省掉 malloc”这么简单，而是把系统资源边界显式写进类型和常量里：

```c
#define SLOT_INVALID 0xffu

typedef struct {
    uint8_t index;
    uint8_t generation;
} channel_handle_t;
```

这里的 `index` 是槽位编号，`generation` 用来识别过期句柄。释放后再次分配同一个 index 时，generation 增加。这样旧 handle 再拿来访问，就能被检测出来，而不会误操作新对象（在 GNSS 接收机中可以用通道号、PRN 等替代`generation`）。

---

## 3. Slot Registry：把对象存储和活跃状态拆开

slot registry 的核心思想是：**payload 数组只负责存数据，registry 负责说明哪些 slot 有效**。

```c
#define MAX_SLOTS 64

typedef struct {
    channel_t items[MAX_SLOTS];
    uint8_t   generation[MAX_SLOTS];
    uint64_t  used_bits;
    uint8_t   used_count;
} channel_registry_t;
```

这种结构和普通链表的思路不同：链表把“对象存储”和“遍历关系”绑在节点指针里；**slot registry 则把对象放在连续数组中**，用独立的 bitmap 表示活跃集合。

这么做有几个好处：

- **内存布局稳定**：对象数组连续，容量固定，没有外部碎片。
- **句柄轻量**：外部只需要保存 index + generation，不需要暴露裸指针。
- **活跃集合可计算**：`used_bits` 本身就是一个集合，可以快速判断、合并、遍历。
- **失败路径清晰**：当 `used_count == MAX_SLOTS` 时，分配失败是确定事件。

可以把 slot registry 理解成一个小型资源表。它不追求通用 allocator 的灵活性，而是用更窄的能力换取更确定的行为。

### 3.1 最小 API

一个实用的 registry 不需要复杂 API，最少只要这些能力（增改删查）：

```c
bool channel_alloc(channel_registry_t *r, channel_handle_t *out);
bool channel_free(channel_registry_t *r, channel_handle_t h);
channel_t *channel_get(channel_registry_t *r, channel_handle_t h);
uint8_t channel_capacity(const channel_registry_t *r);
uint8_t channel_used(const channel_registry_t *r);
```

关键是 `get()` 必须验证 handle：

```c
static bool channel_handle_valid(const channel_registry_t *r,
                                 channel_handle_t h)
{
    if (h.index >= MAX_SLOTS) {
        return false;
    }

    if ((r->used_bits & (1ull << h.index)) == 0) {
        return false;
    }

    return r->generation[h.index] == h.generation;
}
```

这段检查同时覆盖了三类错误：

1.  index 越界；
2.  slot 已释放；
3.  slot 被释放后又重新分配，旧 generation 失效。

---

## 4. 为什么需要 CTZ？

有了 bitmap，下一步是遍历。

最朴素的遍历方式是扫描整个数组：

```c
for (uint8_t i = 0; i < MAX_SLOTS; ++i) {
    if ((r->used_bits & (1ull << i)) != 0) {
        process(&r->items[i]);
    }
}
```

这段代码简单，但它的成本固定是 O(N)。如果 `MAX_SLOTS = 64`，问题不大；如果是 256、512，且每次只有少量活跃对象，就会浪费很多空槽检查。

更好的做法是只遍历被置位（set）的 bit。这里就用到 ctz：count trailing zeros，统计一个整数从最低位开始连续 0 的个数。

对于二进制数：

```text
mask = 0b00101000
ctz(mask) = 3
```

因为最低的 set bit 在 bit 3。

经典遍历模式如下：

```c
uint64_t word = r->used_bits;

while (word != 0) {
    uint8_t bit = ctz64(word);
    process(&r->items[bit]);
    word &= word - 1;  // clear lowest set bit
}
```

**`word &= word - 1` 会清掉最低的 set bit**。

例如：

```text
word     = 0b00101000
word - 1 = 0b00100111
result   = 0b00100000
```

下一轮 ctz 就会找到下一个活跃 slot。

这类模式在系统软件里非常常见。Linux kernel 有完整的 bitmap API，用于对位图执行置位、清零、查找等操作。

实时分配器 TLSF 也使用 bitmap 和位搜索机制来快速定位合适的空闲块类别。

也就是说，bitmap + bit scan 不是奇技淫巧，而是底层系统软件里成熟的集合表示方法。

---

## 5. 多 word bitmap：容量超过机器字长怎么办？

如果 slot 数量超过 64，就把 bitmap 拆成多个 word：

```c
#define SLOT_WORD_BITS 32u
#define MAX_SLOTS      128u
#define SLOT_WORDS     ((MAX_SLOTS + SLOT_WORD_BITS - 1u) / SLOT_WORD_BITS)

typedef struct {
    channel_t items[MAX_SLOTS];
    uint8_t   generation[MAX_SLOTS];
    uint32_t  used_words[SLOT_WORDS];
    uint16_t  used_count;
} channel_registry_t;
```

遍历时先扫 word，再扫 word 内部的 set bit：

```c
for (uint32_t w = 0; w < SLOT_WORDS; ++w) {
    uint32_t bits = r->used_words[w];

    while (bits != 0u) {
        uint32_t bit = ctz32(bits);
        uint32_t index = w * SLOT_WORD_BITS + bit;

        if (index < MAX_SLOTS) {
            process(&r->items[index]);
        }

        bits &= bits - 1u;
    }
}
```

这个遍历的成本可以粗略理解为：

`O(word_count + active_count)`

它仍然要检查每个 word 是否为 0，但不会逐个检查 word 内的每个 slot。对于稀疏活跃集合，这比全数组扫描更合适。

---

## 6. CTZ 的工程边界

ctz 很好用，但有一个重要陷阱：**不要对 0 调用没有定义 0 行为的 ctz builtin**。

GCC 文档里，新的 type-generic `__builtin_ctzg(x, fallback)` 可以在 `x == 0` 时返回 fallback；但如果只传一个参数且参数为 0，结果未定义。传统 `__builtin_ctz()` / `__builtin_ctzl()` / `__builtin_ctzll()` 也应该按“输入不能为 0”来封装。

因此建议永远写一个薄封装：

```c
static inline uint32_t ctz32(uint32_t x)
{
    // caller must guarantee x != 0
    return (uint32_t)__builtin_ctz(x);
}

static inline uint32_t ctz64(uint64_t x)
{
    // caller must guarantee x != 0
    return (uint32_t)__builtin_ctzll(x);
}
```

然后在调用点保证：

```c
while (bits != 0u) {
    uint32_t bit = ctz32(bits);
    bits &= bits - 1u;
}
```

如果是 C++20，可以考虑 `std::countr_zero()`。标准库 `<bit>` 提供了 `countl_zero`、`countr_zero`、`popcount` 等位操作函数，表达上比编译器 builtin 更可移植。

如果目标是 Arm Cortex-M，一些核上没有直接的 CTZ 指令，但可以通过 `RBIT + CLZ` 组合实现：先反转 bit 顺序，再统计 leading zeros。

Arm 指令文档中 CLZ 的语义是统计最高有效位之前的 0 个数；结合 RBIT，就能得到 trailing zeros。是否值得手写 intrinsic，要看编译器对 `__builtin_ctz` 的代码生成质量，通常应优先让编译器处理。

---

## 7. 分配、释放和遍历的示例骨架

下面给一个 64 slot 的简化版本，用单个 `uint64_t` 作为 bitmap：

```c
#include <stdbool.h>
#include <stdint.h>
#include <string.h>

#define MAX_CHANNELS 64u

typedef struct {
    uint8_t  sat_id;
    uint8_t  signal_id;
    uint32_t state;
    float    cn0;
} channel_t;

typedef struct {
    uint8_t index;
    uint8_t generation;
} channel_handle_t;

typedef struct {
    channel_t items[MAX_CHANNELS];
    uint8_t   generation[MAX_CHANNELS];
    uint64_t  used_bits;
    uint8_t   used_count;
} channel_registry_t;

static inline uint32_t ctz64_nonzero(uint64_t x)
{
    return (uint32_t)__builtin_ctzll(x);
}

static bool channel_alloc(channel_registry_t *r, channel_handle_t *out)
{
    if (r->used_count >= MAX_CHANNELS) {
        return false;
    }

    uint64_t free_bits = ~r->used_bits;
    uint8_t index = (uint8_t)ctz64_nonzero(free_bits);

    r->used_bits |= (1ull << index);
    r->used_count++;

    memset(&r->items[index], 0, sizeof(r->items[index]));
    out->index = index;
    out->generation = r->generation[index];
    return true;
}

static bool channel_free(channel_registry_t *r, channel_handle_t h)
{
    if (h.index >= MAX_CHANNELS) {
        return false;
    }

    uint64_t bit = 1ull << h.index;
    if ((r->used_bits & bit) == 0) {
        return false;
    }

    if (r->generation[h.index] != h.generation) {
        return false;
    }

    r->used_bits &= ~bit;
    r->used_count--;
    r->generation[h.index]++;
    return true;
}

static channel_t *channel_get(channel_registry_t *r, channel_handle_t h)
{
    if (h.index >= MAX_CHANNELS) {
        return 0;
    }

    uint64_t bit = 1ull << h.index;
    if ((r->used_bits & bit) == 0) {
        return 0;
    }

    if (r->generation[h.index] != h.generation) {
        return 0;
    }

    return &r->items[h.index];
}
```

这里的分配逻辑用了 `free_bits = ~used_bits`。因为 `MAX_CHANNELS` 正好是 64，所以不需要屏蔽高位。

**如果容量不是 word bit 数的整数倍，一定要 mask 掉超出容量的 bit**：

```c
uint32_t valid_mask = (1u << MAX_SLOTS) - 1u;  // only valid when MAX_SLOTS < 32
uint32_t free_bits = (~used_bits) & valid_mask;
```

工程实现里不要把这行照搬到 `MAX_SLOTS == 32` 的情况，因为 `1u << 32` 本身就是未定义行为。更稳妥的做法是按 word 处理最后一个 word 的 valid mask。

遍历 API 可以写成 callback：

```c
typedef void (*channel_visitor_t)(channel_t *ch, void *ctx);

static void channel_for_each(channel_registry_t *r,
                             channel_visitor_t visitor,
                             void *ctx)
{
    uint64_t bits = r->used_bits;

    while (bits != 0ull) {
        uint8_t index = (uint8_t)ctz64_nonzero(bits);
        visitor(&r->items[index], ctx);
        bits &= bits - 1ull;
    }
}
```

这段遍历不会被释放操作破坏，因为它遍历的是进入函数时复制出来的 `bits`。但这也意味着：如果 visitor 里释放或新增 slot，当前遍历是否应该看到这些变化，在设计接口时需要考虑清楚这个问题。

此外，裸机或 RTOS 环境下，如果 registry 会被中断或多个任务同时访问，还需要用关中断、临界区、mutex 或 lock-free 机制**保护 `used_bits` 和 `used_count` 的一致性**。

---

## 8. 与几种常见方案的对比

| 方案 | 优点 | 风险 |
| --- | --- | --- |
| 全局静态数组 + 全量扫描 | 最简单，内存上界明确 | 遍历固定 O(N)，活跃对象稀疏时浪费 |
| 链表活跃表 | 遍历 O(k)，只访问活跃对象 | 指针开销、删除一致性、节点损坏难排查，局部性差 |
| `malloc/free` + 指针容器 | 灵活，适合复杂变长对象 | 碎片、失败路径、最坏耗时和长期运行风险 |
| 固定块对象池 | 无外部碎片，容量确定 | 需要自己设计句柄、重复释放和满池处理 |
| slot registry + bitmap/ctz | 容量确定，状态紧凑，遍历高效 | 需要处理 ctz(0)、跨 word、并发一致性 |

再次重申，不是要在嵌入式软件中绝对禁止 heap。

通用 heap 可以出现在初始化阶段、非实时路径、调试工具、上位机程序，或者使用 TLSF 这类经过实时约束设计的分配器。

但在 MCU 实时热路径里，如果对象大小固定、容量可估，**固定资源模型通常更容易证明、更容易测试，也更容易在出问题时定位**。

---

## 9. 测试应该覆盖哪些边界？

slot registry 这种结构看起来简单，但边界条件非常多。至少要覆盖：

- 连续分配直到满池，最后一次分配必须失败；
- 释放一个 slot 后再次分配，容量计数恢复正确；
- 重复释放必须失败，不能破坏 `used_count`；
- 释放后旧 handle 调用 `get()` 必须失败；
- generation 回绕时是否可接受，是否需要更宽的 generation；
- bitmap 跨 word 遍历是否正确；
- 最后一个 word 的无效 bit 是否被屏蔽；
- `ctz` 调用点是否保证输入非 0；
- visitor 中释放当前 slot 时行为是否符合设计；
- 中断和任务并发访问时是否有临界区保护。

如果要做性能对比，不一定要追求复杂 benchmark。可以准备三组数据：

1.  `MAX_SLOTS = 64/256/1024`；
2.  活跃对象比例为 5%、25%、75%；
3.  分别对比全数组扫描、链表活跃表、bitmap + ctz。

最终重点不是证明某个数字永远最快，而是证明设计的资源边界：内存占用固定、失败路径可控、遍历成本和活跃集合相关。

---

## 10. 结论：不是反对动态，而是反对无边界动态

嵌入式软件里的很多问题，不是因为用了动态数据结构，而是因为动态性没有被约束。`malloc/free` 给了很大的表达自由，但也把碎片、失败路径、最坏耗时和长期运行风险带进了系统。

对于最大容量已知的对象集合，更稳的做法是把动态性收敛到固定模型里：

- 用定长数组定义内存上界；
- 用 slot registry 定义对象生命周期；
- 用 handle + generation 防止悬空访问；
- 用 bitmap 表示活跃集合；
- 用 ctz 快速索引和遍历 set bit；
- 用清晰的失败返回替代隐式崩溃。

这不是一个复杂框架，而是一种嵌入式设计习惯：**先把资源边界写下来，再谈抽象**。

只要边界清楚，动态数据结构也可以是确定的、可测试的、适合实时系统的。

---

## 参考资料

- Colin Walls, [When To Use Malloc In Dynamic Memory Allocation](https://www.embedded.com/use-malloc-why-not/), Embedded.com.
- Miguel Masmano, Ismael Ripoll, Patricia Balbastre, Alfons Crespo, [A constant-time dynamic storage allocator for real-time systems](https://www.wide-dot.com/tlsf/paper/jrts2008.pdf), Real-Time Systems, 2008.
- Ben Kenwright, [Fast Efficient Fixed-Size Memory Pool: No Loops and No Overhead](https://arxiv.org/abs/2210.16471), arXiv, 2022.
- [Memory allocation using Pool](https://embedded-code-patterns.readthedocs.io/en/latest/pool/), embedded-code-patterns.
- [Linux Kernel API: Bitmap Operations](https://www.kernel.org/doc/html/next/core-api/kernel-api.html), Linux kernel documentation.
- GCC, [Bit Operation Builtins](https://gcc.gnu.org/onlinedocs/gcc-15.1.0/gcc/Bit-Operation-Builtins.html).
- cppreference, [Standard library header `<bit>`](https://en.cppreference.com/w/cpp/header/bit).
- Arm, [CLZ, Count leading zeros](https://developer.arm.com/documentation/ddi0602/2026-03/Base-Instructions/CLZ--Count-leading-zeros-).
