---
name: crush-algorithm
description: CRUSH数据分布算法专家。当用户询问CRUSH算法原理、bucket类型(straw2/list/tree/uniform)、数据分布计算、规则(rule)配置、对象到OSD映射、权重调整时调用此技能。
---

# CRUSH 数据分布算法

## 〇、为什么需要 CRUSH？

为什么需要 CRUSH 算法？在传统的分布式存储系统中，数据位置通常依赖中心化的查找表：

```
传统方案:
  客户端 → 元数据服务器 → 查询 "对象X在哪个OSD？" → 返回 [osd.0, osd.1, osd.2]
  
问题:
  - 元数据服务器成为性能瓶颈（所有 I/O 都需要查询）
  - 元数据服务器故障 = 整个集群不可用
  - 查找表随集群规模线性增长，无法扩展
```

CRUSH 的核心创新是：**客户端自行计算数据位置，无需中心查找表**。

```
CRUSH 方案:
  客户端 → 本地计算: CRUSH(object_id, pool_id, OSDMap) → [osd.0, osd.1, osd.2]
  
优势:
  - 零查询延迟
  - 无单点故障
  - 无限扩展（计算复杂度 O(log n)）
```

CRUSH 全称是 **C**ontrolled **R**eplication **U**nder **S**calable **H**ashing，由 Sage Weil 在 2006 年 SC（Supercomputing）论文中提出（Ceph 整体设计发表于同年的 OSDI 论文）。

---

## 一、核心数据结构

### 1.1 crush_map (src/crush/crush.h:355-468)

```c
struct crush_map {
    struct crush_bucket **buckets;    // 桶数组
    struct crush_rule **rules;        // 规则数组
    __s32 max_buckets;
    __u32 max_rules;
    __s32 max_devices;

    // 可调参数 (tunables):
    __u32 choose_total_tries;         // 默认 50
    __u8  chooseleaf_vary_r;          // 默认 1
    __u8  chooseleaf_stable;          // 默认 1 (避免不必要的数据迁移)
    __u32 allowed_bucket_algs;        // 允许的桶算法位掩码
};
```

### 1.2 crush_bucket (src/crush/crush.h:230-240)

```c
struct crush_bucket {
    __s32 id;         // 桶 ID (< 0, 唯一)
    __u16 type;       // 桶类型 (host=1, rack=2, datacenter=3, root=10)
    __u8  alg;        // 选择算法
    __u8  hash;       // 哈希函数
    __u32 weight;     // 16.16 定点权重
    __u32 size;       // 子项数量
    __s32 *items;     // 子项: < 0 = 桶, >= 0 = OSD
};
```

### 1.3 五种 Bucket 类型

| 类型 | 算法 | 时间复杂度 | 添加 | 删除 | 推荐度 |
|------|------|-----------|------|------|--------|
| **Uniform** | 排列选择 | O(1) | 差 | 差 | 不推荐 |
| **List** | 链表遍历 | O(n) | 最优 | 差 | 不推荐 |
| **Tree** | 二叉树下降 | O(log n) | 次优 | 次优 | 不推荐 |
| **Straw** | 抽签 (旧) | O(n) | 最优 | 最优 | 已废弃 |
| **Straw2** | 抽签 (新) | O(n) | 最优 | 最优 | **默认推荐** |

### 1.4 Straw2 Bucket (src/crush/crush.h:341-344)

```c
struct crush_bucket_straw2 {
    struct crush_bucket h;
    __u32 *item_weights;  // 16.16 定点权重
};
```

---

## 二、规则系统

### 2.1 crush_rule (src/crush/crush.h:84-91)

```c
struct crush_rule {
    __u32 len;
    __u8 type;  // REPLICATED=1, ERASURE=3, MSR_FIRSTN=4, MSR_INDEP=5
    struct crush_rule_step steps[0];
};
```

### 2.2 规则步骤 (src/crush/crush.h:51-75)

| 操作码 | 值 | 说明 |
|--------|-----|------|
| `CRUSH_RULE_TAKE` | 1 | 从指定桶开始 |
| `CRUSH_RULE_CHOOSELEAF_FIRSTN` | 6 | 选择 N 个叶子（顺序依赖） |
| `CRUSH_RULE_CHOOSELEAF_INDEP` | 7 | 选择 N 个叶子（位置独立） |
| `CRUSH_RULE_EMIT` | 4 | 输出结果 |

### 2.3 典型规则示例

```
rule replicated_rule {
    id 0
    type replicated
    min_size 1
    max_size 10
    step take default                    # 从 root=default 开始
    step chooseleaf firstn 0 type host   # 每个 host 选 1 个 OSD
    step emit                            # 输出结果
}
```

---

## 三、Straw2 选择算法

### 3.1 数学原理

Straw2 基于指数分布：每个 item 生成一个指数随机变量，值最大的获胜。

```c
// src/crush/mapper.c:315-340
static inline __s64 generate_exponential_distribution(int type, int x, int y, int z, int weight)
{
    unsigned int u = crush_hash32_3(type, x, y, z);  // 均匀分布 [0, 0xffff]
    u &= 0xffff;
    __s64 ln = crush_ln(u) - 0x1000000000000ll;      // 自然对数 (查表)
    return div64_s64(ln, weight);                     // 除以权重
}
```

### 3.2 Straw2 选择流程

```c
// src/crush/mapper.c:342-365
static int bucket_straw2_choose(const struct crush_bucket_straw2 *bucket,
                                int x, int r, ...)
{
    for (i = 0; i < bucket->h.size; i++) {
        if (weights[i]) {
            draw = generate_exponential_distribution(..., weights[i]);
        } else {
            draw = S64_MIN;  // 权重为 0 的 item 永远不会赢
        }
        if (i == 0 || draw > high_draw) {
            high = i;
            high_draw = draw;
        }
    }
    return bucket->h.items[high];
}
```

---

## 四、crush_do_rule() 完整流程

### 4.1 入口点 (src/crush/mapper.c:2017-2053)

```c
int crush_do_rule(const struct crush_map *map,
                  int ruleno, int x, int *result, int result_max,
                  const __u32 *weight, int weight_max,
                  void *cwin, const struct crush_choose_arg *choose_args)
```

### 4.2 执行流程

```
输入: x=12345 (PG ID), numrep=3, target_type=host

Step 1: TAKE root
  working_set = [root_bucket_id]

Step 2: CHOOSELEAF_FIRSTN 0 type host
  对 working_set 中每个桶:
    对每个副本 (rep=0,1,2):
      r = rep + parent_r + ftotal
      item = crush_bucket_choose(bucket, x, r)
      如果 item 不是目标类型 → 递归下降
      如果 recurse_to_leaf → 继续下降到 OSD
      检查冲突和 OSD 是否 out
      如果失败 → 重试 (最多 choose_total_tries 次)

Step 3: EMIT
  result = [osd.0, osd.1, osd.2]
```

### 4.3 完整示例

```
层级结构:
         root (id=-1, type=10)
        /    |    \
    host0  host1  host2  (type=1)
     |      |      |
   osd.0  osd.1  osd.2  (type=0)

计算 x=12345, numrep=3:

1. TAKE root: w=[-1]

2. CHOOSELEAF_FIRSTN:
   副本 0: r=0, hash(root, x=12345, r=0) → host0 → osd.0
   副本 1: r=1, hash(root, x=12345, r=1) → host1 → osd.1
   副本 2: r=2, hash(root, x=12345, r=2) → host2 → osd.2

3. EMIT: result=[0, 1, 2]
```

---

## 五、哈希函数

### 5.1 RJENKINS1 (src/crush/hash.c:12-22)

```c
#define crush_hashmix(a, b, c) do {             \
    a = a-b;  a = a-c;  a = a^(c>>13);          \
    b = b-c;  b = b-a;  b = b^(a<<8);           \
    c = c-a;  c = c-b;  c = c^(b>>13);          \
    ... (多轮混合)                               \
} while (0)

#define crush_hash_seed 1315423911
```

### 5.2 自然对数查表 (src/crush/mapper.c:229-271)

`crush_ln()` 使用预计算表计算 `2^44 * log2(input+1)`，用于 Straw2 的指数分布生成。

---

## 六、is_out() 函数

### 6.1 OSD 状态判断 (src/crush/mapper.c:405-419)

```c
static int is_out(const struct crush_map *map,
                  const __u32 *weight, int weight_max,
                  int item, int x)
{
    if (item >= weight_max) return 1;           // 超出范围 = out
    if (weight[item] >= 0x10000) return 0;      // 满权重 = in
    if (weight[item] == 0) return 1;            // 零权重 = out
    // 部分权重: 概率性判断 (用于渐进式 reweight)
    if ((crush_hash32_2(CRUSH_HASH_RJENKINS1, x, item) & 0xffff) < weight[item])
        return 0;  // in
    return 1;  // out
}
```

---

## 七、内核 CRUSH 实现

### 7.1 工作空间管理 (linux/net/ceph/osdmap.c:981-1109)

内核使用工作空间池模式（借鉴自 Btrfs 压缩），为每个 CPU 缓存 CRUSH 工作空间：

```c
// 工作空间池管理
get_workspace()   // 获取空闲工作空间 (或等待)
put_workspace()   // 归还工作空间到池
```

### 7.2 关键代码位置

| 功能 | 用户态 | 内核态 |
|------|--------|--------|
| 数据结构 | src/crush/crush.h | linux/net/ceph/osdmap.c |
| 映射函数 | src/crush/mapper.c | linux/net/ceph/osdmap.c |
| 哈希函数 | src/crush/hash.c | linux/net/ceph/ceph_hash.c |
| 工作空间 | crush_init_workspace() | alloc_workspace() |

---

## 八、关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| crush_map 结构 | src/crush/crush.h | 355-468 |
| crush_bucket 结构 | src/crush/crush.h | 230-240 |
| crush_rule 结构 | src/crush/crush.h | 84-91 |
| crush_do_rule | src/crush/mapper.c | 2017-2053 |
| crush_do_rule_no_retry | src/crush/mapper.c | 826-1032 |
| crush_choose_firstn | src/crush/mapper.c | 441-629 |
| crush_choose_indep | src/crush/mapper.c | 636-824 |
| bucket_straw2_choose | src/crush/mapper.c | 342-365 |
| generate_exponential_distribution | src/crush/mapper.c | 315-340 |
| crush_bucket_choose (dispatch) | src/crush/mapper.c | 368-399 |
| is_out | src/crush/mapper.c | 405-419 |
| crush_hash32 | src/crush/hash.c | 93-141 |
| crush_ln | src/crush/mapper.c | 229-271 |
| C++ 封装 | src/crush/CrushWrapper.h | 1-1673 |
| 内核 CRUSH 解码 | linux/net/ceph/osdmap.c | 71-678 |
| 内核工作空间管理 | linux/net/ceph/osdmap.c | 981-1109 |

---

## 九、参考文献与资源

### 学术论文
1. **"Ceph: A Scalable, High-Performance Distributed File System"** - Weil et al., OSDI 2006
2. **"The CRUSH Algorithm"** - Weil et al., SC 2006
   - CRUSH 算法完整论文

### 官方文档
3. **CRUSH Map 文档**: https://docs.ceph.com/en/latest/rados/operations/crush-map/
4. **CRUSH 调优指南**: https://docs.ceph.com/en/latest/rados/operations/crush-map/#crush-map-tunables

### 调试工具
5. **crushtool**: 测试和调试 CRUSH 映射
   ```bash
   # 查看当前 CRUSH 映射
   crushtool -d /tmp/crush.map -t
   
   # 测试数据分布
   crushtool -i /tmp/crush.map --test --min-x 0 --max-x 100 --num-rep 3
   
   # 编译文本规则
   crushtool -c crush.txt -o crush.map
   ```

---

## 十、常见问题与陷阱

### Q1: 为什么 Straw2 是推荐的 bucket 类型？

**A**: Straw2 相比其他类型:
- 添加/删除 OSD 时数据迁移量最小（最优）
- 分布均匀性最好
- 支持 per-position 权重调整（choose_args）

### Q2: `chooseleaf_stable` 调优参数的作用？

**A**: 启用后，CRUSH 在 descend 过程中会对 r 值进行位偏移，避免因单个 bucket 变化导致大量数据迁移。默认值为 1（启用）。

### Q3: FIRSTN 和 INDEP 有什么区别？

**A**:
- **FIRSTN**: 顺序依赖，副本 1 的计算依赖副本 0 的结果。改变副本数量会影响所有后续副本的位置。
- **INDEP**: 位置独立，每个副本独立计算。改变副本数量不影响已有副本的位置。适用于 EC（纠删码）池。

### 相关技能

- [Objecter](../../client/objecter/SKILL.md) — CRUSH 的主要调用者，客户端在 Objecter 中执行 PG→OSD 映射
- [Ceph 整体概述](../../overview/SKILL.md) — CRUSH 在 Ceph 架构中的位置
