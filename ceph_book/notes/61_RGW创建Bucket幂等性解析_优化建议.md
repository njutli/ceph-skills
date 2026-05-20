# Note 61 优化建议

## 1. HTTP 状态码的精确返回
- **原文错误**：原文提到如果 Bucket 已存在且策略一致，返回 200 OK。
- **改正**：根据 S3 协议规范，如果 Bucket 已存在且策略一致，RGW 应返回 **`409 Conflict`** 并附带 `BucketAlreadyOwnedByYou` 错误码，而不是 200 OK。这明确告知客户端 Bucket 已存在。建议修正状态码描述以符合 S3 标准。
