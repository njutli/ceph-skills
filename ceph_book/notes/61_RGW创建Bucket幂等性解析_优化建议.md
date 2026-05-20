# Note 61 优化建议

## 1. HTTP 状态码的精确返回
- **原文问题**：原文提到返回 200 OK 或 `ERR_BUCKET_EXISTS`。
- **优化建议**：根据 S3 协议规范，如果 Bucket 已存在且策略一致，RGW 应返回 **`409 Conflict`** 并附带 `BucketAlreadyOwnedByYou` 错误码，而不是 200 OK。这明确告知客户端 Bucket 已存在，但操作是安全的（因为参数一致）。如果策略不同，则返回 `409 Conflict` 附带 `BucketAlreadyExists`。建议修正状态码描述以符合 S3 标准。

## 2. 并发创建的处理
- **原文问题**：原文未提及并发创建场景。
- **优化建议**：补充说明 RGW 使用 **RADOS 的乐观锁（CAS, Compare-And-Swap）** 机制来处理并发创建。当多个网关同时收到创建同名 Bucket 的请求时，只有一个能成功写入 `domain_root`，其他会收到 `EEXIST` 错误，从而保证全局唯一性。

## 3. 存储策略变更的限制
- **原文问题**：原文提到策略不同会返回冲突。
- **优化建议**：明确指出 RGW **不支持**在 Bucket 创建后修改 `placement_rule`（存储策略）。如果需要将数据迁移到不同 Pool，必须创建新 Bucket 并使用 `radosgw-admin bucket sync` 或客户端工具进行数据拷贝。这是 RGW 架构设计的限制。

## 4. 源码位置
- **优化建议**：Bucket 创建逻辑位于 `src/rgw/rgw_bucket.cc` 的 `RGWCreateBucket::execute` 函数。幂等性检查和策略比对位于该函数的开头部分。
