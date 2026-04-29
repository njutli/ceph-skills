---
name: rgw
description: RGW对象存储网关专家。当用户询问RGW架构、S3/Swift API、RADOS后端存储、多站点复制(multisite)、生命周期(lifecycle)、桶索引(bucket index)、IAM策略、PubSub通知时调用此技能。
---

# RGW (RADOS Gateway) 对象存储网关

## 〇、为什么需要 RGW？

Ceph 底层 RADOS 提供原生对象存储，但 REST API 是 HTTP 协议的，客户端无法直接与 OSD 通信。RGW 作为 HTTP 网关层：

```
        S3 客户端                Swift 客户端
           │                        │
           ▼                        ▼
  ┌─────────────────────────────────────────┐
  │              RGW (radosgw)               │
  │  ┌──────────────┐  ┌──────────────────┐  │
  │  │  S3 API 层   │  │  Swift API 层    │  │
  │  │ (rgw_rest_s3)│  │(rgw_rest_swift) │  │
  │  └──────┬───────┘  └────────┬─────────┘  │
  │         └─────────┬─────────┘             │
  │                   ▼                       │
  │          RGW Op (rgw_op)                  │
  │                   │                       │
  │                   ▼                       │
  │  ┌─────────────────────────────────────┐  │
  │  │       SAL (Storage Abstraction)     │  │
  │  └─────────────────────────────────────┘  │
  └────────────────────┬──────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  librados + Ceph │
              └─────────────────┘
```

HTPP 前端 (Beast / Civetweb) 接收 HTTP 请求 → RGW 解析 S3/Swift 语义 → 转换为 RADOS 对象操作 → 写入 Ceph 集群。

## 一、核心数据结构

### 1.1 SAL (Storage Abstraction Layer) - src/rgw/rgw_sal.h

RGW 最重要的抽象层，将存储后端与 HTTP 前端解耦：

```cpp
// SAL 层核心接口 (简化)
class Store {
    virtual unique_ptr<Bucket> get_bucket(const rgw_bucket& b) = 0;
    virtual unique_ptr<Object> get_object(const rgw_obj_key& k) = 0;
    virtual int list_objects(...) = 0;
    virtual int create_user(const RGWUserInfo& u, ...) = 0;
};

class Bucket {
    virtual int list(dpp, params, ...) = 0;   // 列举桶内对象
    virtual int get_owner(rgw_user& o) = 0;    // 获取桶属主
    virtual int remove(dpp, ...) = 0;           // 删除桶
    virtual int set_acl(const RGWAccessControlPolicy& a) = 0;
};

class Object {
    virtual int read(dpp, bufferlist& bl, ...) = 0;
    virtual int write(dpp, bufferlist& bl, ...) = 0;
    virtual int get_attrs(map<string,bufferlist>& a) = 0;
    virtual int set_attrs(dpp, map<string,bufferlist>& a) = 0;
};

class RadosStore : public Store {   // 基于 RADOS 的实现
    librados::Rados rados;
    ...
};
```

新增存储后端（如 DBStore、CORTX Motr）只需实现 SAL 接口。

### 1.2 RGW 对象映射

S3 语义映射到 RADOS 对象：

```
S3 请求: GET /my-bucket/photo.jpg HTTP/1.1

RGW 内部映射:
  用户 (UID) → 通过 radosgw-admin 创建, 存为 "user.<uid>" 对象
  桶 (Bucket) → 存为 "bucket.<bucket_id>" 对象
         │
         ├─ 桶索引 (Bucket Index): "index.<bucket_id>" 对象 (omap 存储)
         │    key = object_name (photo.jpg)
         │    value = {size, etag, mtime, storage_class, acl, ...}
         │
         └─ 数据对象: "default.<bucket_instance_id>_photo.jpg"
               attrs: x-amz-meta-*, content-type, etag, ...
               data: 文件内容
```

### 1.3 桶索引 (Bucket Index) - src/rgw/rgw_bucket.h

桶索引是 RGW 最关键的数据结构之一，支持 O(1) 的对象元数据查找：

```
索引存储方式:
  - 默认: 单个 RADOS omap 对象
  - 大桶: 动态分片 (sharding), 每片一个 omap 对象
    "index.<bucket_id>.0", "index.<bucket_id>.1", ...

分片策略:
  - 对 object_name 取 hash
  - hash % num_shards → 确定落在哪个分片
  - 分片数可根据桶大小动态扩展 (reshard)

索引内容 (每个对象一条):
{
  "name": "photo.jpg",
  "instance": "",  // 版本号 (versioning)
  "locator": "",
  "exists": true,
  "meta": {
    "category": 0,
    "size": 1048576,
    "mtime": "2024-01-01T00:00:00Z",
    "etag": "d41d8cd98f00b204e9800998ecf8427e",
    "storage_class": "STANDARD",
    "owner": "user_uid",
    "owner_display_name": "John Doe",
    "content_type": "image/jpeg",
    "accounted_size": 1048576,
    "user_data": "...",
    "appendable": false
  },
  "tag": "_Dv...==",  // ACL 标签
  "ver": {...}        // versioning 信息
}
```

## 二、核心流程

### 2.1 HTTP 请求处理流程

```
HTTP 请求 GET /bucket/object
  │
  ▼
1. RGWFCGX / RGWBeastHandler 接收连接
   src/rgw/rgw_main.cc:rgw_process_request()
  │
  ▼
2. RGWHandler_REST 解析 HTTP 头
   src/rgw/rgw_rest.cc:authorize()
  │
  ├─ RGW_Auth_S3::authorize()  → 验证 AWS Signature V4
  │  或 RGW_Auth_Swift::authorize() → 验证 Swift TempAuth/Keystone
  ├─ RGWUser 查询 (权限、配额、ACL)
  │
  ▼
3. RGWHandler_REST_S3::get_op()  (或 RGWHandler_REST_Swift)
   src/rgw/rgw_rest_s3.cc
  │
  ├─ 根据 HTTP method + URL 路由到具体 Op:
  │    GET /bucket → RGWListBucket
  │    GET /bucket/obj → RGWGetObj
  │    PUT /bucket/obj → RGWPutObj
  │    DELETE /bucket/obj → RGWDeleteObj
  │    ...
  │
  ▼
4. RGWOp::execute()
   src/rgw/rgw_op.cc
  │
  ├─ verify_permission()  → IAM 策略 + Bucket Policy 鉴权
  ├─ pre_exec()           → 参数校验
  ├─ execute()            → 核心逻辑: 读写 RADOS
  ├─ complete()           → 写后处理 (更新索引、触发通知)
  │
  ▼
5. 返回 HTTP 响应
```

### 2.2 PUT 对象流程 (简化)

```
HTTP PUT /bucket/photo.jpg → RGWPutObj::execute()
  │
  ├─ 1. 从 Bucket 索引找到桶实例
  │
  ├─ 2. 分配 RadosWriter
  │     - 计算对象名: "default.<bucket_id>_photo.jpg"
  │     - 根据 storage_class 选择数据 pool
  │
  ├─ 3. 流式写入 RADOS (支持分块上传)
  │     rgw::sal::Object::write()
  │       → librados::IoCtx::write_full()  (小对象)
  │       或 librados::IoCtx::write() + append (大对象/分块)
  │
  ├─ 4. 上传完成后:
  │     - 计算 ETag (MD5/XXHash)
  │     - 设置 attrs (x-amz-meta-*, content-type, ...)
  │     - 更新 Bucket Index (omap_set)
  │     - 更新桶统计 (num_objects, size_utilized)
  │     - 触发 PubSub 通知 (s3:ObjectCreated:Put)
  │
  └─ 5. 返回 200 OK + ETag
```

### 2.3 多站点复制 (Multisite)

跨地域桶同步：

```
                    Zone A (primary)              Zone B (secondary)
              ┌──────────────────┐          ┌───────────────────┐
用户写入 ──→  │  RGW (zone-a)    │          │  RGW (zone-b)      │
              │      │          │          │       │            │
              │      ▼          │          │       ▼            │
              │  写本地 RADOS   │   sync   │  从 zone-a pull    │
              │      │          │ ──────→  │  数据 + 索引       │
              │      ▼          │          │       │            │
              │  写 Meta Log    │          │  写本地 RADOS     │
              │  写 Data Log    │          │       │            │
              └──────────────────┘          └───────────────────┘
```

关键数据结构：
```
Meta Log (元数据日志):
  - 记录桶、用户、策略的变更
  - 存储在 "meta.log:..." RADOS 对象中

Data Log (数据日志):
  - 记录对象级别的增删改
  - 存储在 "data.log:..." 分片对象中
  - 包含: op_type (write/delete), bucket, object, etag, mtime, ...

Sync 过程:
  zone-b 的 RGW 定期:
    1. 读取 zone-a 的 meta log 增量 → 同步桶/用户变更
    2. 读取 zone-a 的 data log 增量 → 逐对象同步数据
    3. 更新本地 Bucket Index, 设置 local marker
```

代码位置:
- `src/rgw/rgw_sync.cc` - 同步引擎
- `src/rgw/rgw_data_sync.cc` - 数据日志同步
- `src/rgw/rgw_meta_sync.cc` - 元数据日志同步

### 2.4 生命周期 (Lifecycle)

自动过期或转储对象：

```xml
<!-- S3 Lifecycle 规则示例 -->
<LifecycleConfiguration>
  <Rule>
    <ID>expire-old-logs</ID>
    <Filter>
      <Prefix>logs/</Prefix>
    </Filter>
    <Status>Enabled</Status>
    <Expiration>
      <Days>90</Days>          <!-- 90天后删除 -->
    </Expiration>
    <Transition>
      <Days>30</Days>           <!-- 30天后转为低频存储 -->
      <StorageClass>GLACIER</StorageClass>
    </Transition>
  </Rule>
</LifecycleConfiguration>
```

RGW 实现 (`src/rgw/rgw_lc.cc`):
- 每个桶的 Lifecycle 规则存储在桶对象的 attrs 中
- 后台线程定期遍历桶索引，检查每个对象的 mtime
- 对符合条件的对象执行删除或 storage_class 变更

### 2.5 IAM 权限控制

RGW 支持 AWS IAM 风格的细粒度权限控制：

```
权限检查链:
  user → user policy → bucket policy → bucket ACL → object ACL
  
IAM 策略结构 (src/rgw/rgw_iam_policy.cc):
  {
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": ["arn:aws:iam::user_id"]},
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": ["arn:aws:s3:::bucket/*"]
    }]
  }
```

## 三、关键设计特性

| 特性 | 说明 | 代码位置 |
|------|------|---------|
| **S3 API 兼容** | 99% S3 API 实现 | `src/rgw/rgw_rest_s3.cc` |
| **Swift API 兼容** | OpenStack Swift API 支持 | `src/rgw/rgw_rest_swift.cc` |
| **Multisite** | 跨地域异步复制 | `src/rgw/rgw_sync.cc` |
| **Bucket Sharding** | 桶索引水平分片，支持百亿对象级大桶 | `src/rgw/rgw_reshard.cc` |
| **Versioning** | 对象多版本保留 | `src/rgw/rgw_op.cc:RGWBulkUploadOp` |
| **Lifecycle** | 自动过期/转储 | `src/rgw/rgw_lc.cc` |
| **SSE (加密)** | 服务端加密 SSE-S3/SSE-KMS | `src/rgw/rgw_crypt.cc` |
| **IAM/Policy** | AWS IAM 兼容权限策略 | `src/rgw/rgw_iam_policy.cc` |
| **PubSub** | 对象事件通知 (Kafka/AMQP/HTTP) | `src/rgw/rgw_pubsub.cc` |
| **Compression** | 存储压缩 | `rgw_compression_type` 配置 |
| **CORS** | 跨域资源共享 | `src/rgw/rgw_cors.cc` |
| **Static Website** | 静态网站托管 | `src/rgw/rgw_rest_s3.cc (website)` |
| **STS** | 临时安全凭证 (AssumeRole) | `src/rgw/rgw_rest_sts.cc` |
| **Usage/Logging** | 使用统计与访问日志 | `src/rgw/rgw_usage.cc`, `rgw_log.cc` |
| **Lua Scripting** | 请求过滤与变换 | `src/rgw/rgw_lua.cc` |

## 四、关键代码位置

```
src/rgw/
├── rgw_main.cc                      # RGW 入口, Beast/Civetweb 启动
├── rgw_process.cc                   # HTTP 请求处理调度
│
├── rgw_rest.cc/h                    # REST 基础框架
├── rgw_rest_s3.cc                   # S3 API 路由 (10000+ 行)
├── rgw_rest_swift.cc                # Swift API 路由
├── rgw_rest_sts.cc/h                # STS (临时凭证) API
├── rgw_rest_iam_*.cc                # IAM 用户/角色/策略 API
├── rgw_rest_pubsub.cc               # PubSub 通知 API
│
├── rgw_op.cc/h                      # 核心操作实现 (10000+ 行)
│   ├── RGWGetObj                     # GET 对象
│   ├── RGWPutObj                     # PUT 对象
│   ├── RGWDeleteObj                  # DELETE 对象
│   ├── RGWListBucket                # LIST 桶
│   ├── RGWCreateBucket              # 创建桶
│   └── RGWBulkUploadOp              # 分块上传 / 多版本
│
├── rgw_sal.h                        # 存储抽象层接口
├── rgw_sal_rados.cc/h               # RADOS SAL 实现
├── rgw_sal_dbstore.cc/h             # DBStore SAL 实现 (开发/测试)
├── rgw_sal_filter.cc/h              # SAL 拦截器
│
├── rgw_bucket.*                     # 桶操作
├── rgw_user.cc/h                    # 用户管理
├── rgw_acl_*.cc                     # ACL 解析与评估
│
├── rgw_sync.cc/h                    # 多站点同步引擎
├── rgw_data_sync.cc/h               # 数据日志同步
├── rgw_meta_sync.cc/h               # 元数据日志同步
├── rgw_zone.cc/h                    # Zone/ZoneGroup 配置
│
├── rgw_lc.cc/h                      # 生命周期管理
├── rgw_reshard.cc/h                 # 桶分片动态调整
├── rgw_iam_policy.cc/h              # IAM 策略引擎
├── rgw_crypt.cc/h                   # 加密 (SSE)
├── rgw_pubsub.cc/h                  # PubSub 通知
├── rgw_compression.cc/h             # 压缩
├── rgw_cors.cc/h                    # CORS 支持
├── rgw_usage.cc/h                   # 使用量统计
├── rgw_log.cc/h                     # 访问日志
├── rgw_lua.cc/h                     # Lua 脚本
│
├── rgw_auth_s3.cc                   # AWS Signature V4 认证
├── rgw_auth_keystone.cc             # Keystone 认证
├── rgw_auth_registry.cc             # 认证插件注册
│
├── rgw_common.cc/h                  # 工具函数、配置
├── rgw_rados.cc/h                   # RADOS 底层操作封装
│
└── driver/                          # 存储后端驱动
    └── rados/                       # RADOS 驱动具体实现
```

```
src/rgw/lua/                         # Lua 脚本集成
├── rgw_lua_background.cc            # 后台请求处理
└── rgw_lua_request.cc               # 请求上下文
```

## 五、常见问题与陷阱

1. **桶索引过大导致 LIST 超时**
   - 大桶 (百万+ 对象) 需要预分片。创建桶时指定：`radosgw-admin bucket reshard --bucket=bkt --num-shards=11`
   - 动态 reshard 由 `rgw_dynamic_resharding=true` 控制，但大桶 reshard 非常耗时。

2. **Multisite 数据延迟**
   - 多站点复制是异步的，延迟受日志轮转间隔和网络带宽影响。`rgw_sync_log_trim_interval` 和 `rgw_sync_data_inject_err_probability` 影响日志清理和传输。
   - 监控：`radosgw-admin sync status`

3. **S3 签名版本不匹配**
   - 默认只支持 Signature V4。旧客户端使用 Signature V2 会收到 `InvalidRequest` 拒绝。
   - 解决方案：确保客户端使用 V4 签名，或临时在少数旧客户端场景禁用 V4 强制要求。

4. **`NoSuchBucketPolicy` 错误**
   - IAM 策略语法严格，空格的差异都会导致策略解析失败。
   - 调试：`radosgw-admin policy` 命令验证策略语法。

5. **大文件上传失败**
   - S3 默认分块上传 (multipart upload) 最大支持 5GB 每块。超过需分段。
   - `rgw_max_chunk_size` 控制内部 RADOS 写入粒度，过小导致过多 RADOS 操作。

6. **Beast vs Civetweb**
   - Ceph Nautilus+ 默认使用 Beast (Boost.Beast) 作为 HTTP 前端，替代旧版 Civetweb。
   - 区别：Beast 性能更好 (异步 I/O)，Civetweb 更成熟但社区不再维护。
   - 配置：`rgw_frontends = beast port=7480`

7. **存储类 (Storage Class) 与压缩**
   - STANDARD_IA / GLACIER 仅作为元数据标签存在，RGW 不自动降冷。
   - 压缩策略通过 `rgw compression type` 全局控制，不是 per-object 的。

## 六、参考文献

1. **RGW 官方文档**: https://docs.ceph.com/en/latest/radosgw/
2. **Multisite 配置**: https://docs.ceph.com/en/latest/radosgw/multisite/
3. **S3 API 兼容性**: https://docs.ceph.com/en/latest/radosgw/s3/
4. **AWS S3 REST API**: https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
5. **OpenStack Swift API**: https://docs.openstack.org/api-ref/object-store/
