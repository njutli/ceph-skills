# Q&A: CRUSH 可调参数模板 (Profile) 详解与源码实现

## 问题
在 1.3 节中提到系统自定义了 CRUSH 可调参数模板（Profile），如 `argonaut`、`jewel` 等。
**疑问**：
1. 这些模板包含哪些具体内容？不同类型模板包含的内容类型是否一样？
2. 能否以一个模板为例，列举其具体内容？
3. 这些模板的设置在 Ceph 源码中有具体体现吗？

## 解答

### 1. 模板包含的内容
所有模板包含的**参数类型（字段）是完全一致的**，区别仅在于**参数值**的不同。这些模板代表了 Ceph 在不同历史时期（Release 版本）推荐的配置集合。
核心配置参数包括：
*   **重试限制**: `choose_total_tries` (全局重试次数), `choose_local_tries` (局部重试, 已废弃).
*   **容灾域控制**: `chooseleaf_descend_once` (递归下降次数), `chooseleaf_vary_r` (随机种子微调), `chooseleaf_stable` (稳定性控制).
*   **算法版本**: `straw_calc_version` (Straw 权重计算版本).
*   **算法白名单**: `allowed_bucket_algs` (允许使用的 Bucket 算法，如 straw, straw2).

### 2. 模板示例：`jewel` Profile
`jewel` 模板代表了 Jewel 版本（及以后）的标准配置，修复了许多早期版本的映射不均问题。
在源码 `src/crush/CrushWrapper.h` 中，设置 `jewel` 模板的逻辑如下：

```cpp
void set_tunables_jewel() {
  crush->choose_local_tries = 0;              // 废弃局部重试
  crush->choose_local_fallback_tries = 0;     // 废弃局部后备重试
  crush->choose_total_tries = 50;             // 全局重试次数设为50，提高选路成功率
  crush->chooseleaf_descend_once = 1;         // 开启：容灾模式递归限制只下降一次
  crush->chooseleaf_vary_r = 1;               // 开启：递归时微调随机因子，降低哈希冲突
  crush->chooseleaf_stable = 1;               // 开启：确保 OSD 失效时数据迁移量最小化
  // 开启 straw2 等算法支持
  crush->allowed_bucket_algs = ...; 
}
```

相比之下，旧模板（如 `argonaut`）的 `choose_total_tries` 仅为 19，且未开启 `stable` 和 `vary_r`。

### 3. 代码实现位置
这些模板不仅是概念，而是实打实的代码实现。当你执行 `ceph osd crush tunables <profile>` 时：
1.  **入口处理**: 位于 `src/mon/OSDMonitor.cc`，Monitor 节点接收命令并解析具体是哪个 profile (如 `jewel`, `firefly`)。
2.  **逻辑执行**: 根据解析结果，调用 `crush->set_tunables_jewel()` 等方法修改内存中的 `crush_map` 结构体。
3.  **参数定义**: 所有的可调参数都定义在 `src/crush/crush.h` 的 `struct crush_map` 中。

**总结：**
Profile 就是一组“开箱即用”的参数预设集。生产环境中建议使用 `optimal`（等同于最新版 Jewel 配置），以获得最佳的数据均衡性和稳定性。
