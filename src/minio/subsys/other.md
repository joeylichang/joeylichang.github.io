##### Sign

1. 签名的目的是防止请求被篡改

2. S3 签名 V4 的主要元素（根据请求做适当组合）

   1. 秘钥（access keys —— access key ID, secret access key）

   2. 选取请求中部分元素进行签名，S3 收到请求没有相应部分或者签名不对，则拒绝请求

      1. 被签名的部分会作为 Head 的部分信息

   3. 签名 15min 内有效，有权限的为认证的用户可以在 15min 内修改未签名的部分，进而影响请求，建议最大化的对请求 Head 和 Body 进行签名。

      

##### ACL

1. 基于资源的访问策略（Policy）
   1. 资源分类：Bucket、Object
      1. Bucket子资源：lifecycle、website、versioning、policy、acl、cors、logging
      2. Object子资源：acl、restore
   2. Bucket子资源
      1. lifecycle
         1. 过期操作：过期删除
         2. 转换操作：到达时间后转换存储类（类似冷热存储吧），将次存储成
      2. website：托管静态网站，展示静态信息
      3. versioning：版本控制
      4. cors：跨资源共享（与 website 相关）
      5. logging：访问日志
      6. policy、acl（object部分介绍）：版本控制
   3. Object子资源
      1. acl：哪些账户或组将被授予访问权限以及访问的类型
         1. 访问策略：s3:ListBucket、GetObject等
         2. 权限集：READ、WRITE、READ_ACP、WRITE_CAP、FULL_CONTROL，每个权限对应若干访问策略，如 READ 对应 s3:ListBucket、GetObject 等
         3. 标准 acl：用户或者用户组 与 权限集进行绑定
         4. 通过 Bucket 的权限控制 Object 的 访问权限：
            1. s3:x-amz-grant-read ‐ 需要 Bucket 读取访问权限才能读 Obejct
            2. s3:x-amz-grant-write ‐ 需要 Bucket 写入访问权限
            3. s3:x-amz-grant-read-acp ‐ 需要对存储桶 ACL 的读取访问权限
            4. s3:x-amz-grant-write-acp ‐ 需要对存储桶 ACL 的写入访问权限
            5. s3:x-amz-grant-full-control ‐ 需要完全控制权限
            6. s3:x-amz-acl ‐ 需要一个标准 ACL
      2. restore
         1. S3 Glacier 存储类中的对象是存档对象。要访问这类对象，必须首先启动还原请求。
2. 基于用户的访问策略（IAM）
   1. AWS 通用的基于用户的权限系统



##### 用户级对象锁

UserDefine 支持对象保持功能，加上保持的对象只读不写，拒绝覆盖和删除。对象保持针对的是 version，既针对某一个版本进行加保持。

1. 分两种模式（AmzObjectLockMode）

   1. governance（X-Amz-Bypass-Governance-Retention）：只有有相应权限的账户可以更新、删除保持，或者被该用户授予相应权限的用户。
   2. Compliance：必须与一个过期保持周期绑定，过期之前任何账户都不能删除，并且保持周期只能延长不能缩短。

2. 保持周期（AmzObjectLockRetainUntilDate）

   1. 可以针对 Object 单独设置，也可以使用 Bucket 的默认配置。
   2. 作为元数据存储在 metadata（UserDefine）

3. 合法持有

   1. 只要有 s3:PutObjectLegalHold 权限的账户都可以对一个 Object 进行合法持有，持有期间不会被删除和覆盖。
   2. 直到被任何账户（具有上述权限的账户）删除合法持有。

4. **注意：**

   1. 合法持有 与 保持周期 是互相平行没有交集的两种机制。

   2. 删除合法保持，如果保持周期未到，仍不会被任何用户删除。

   3. 同样合法保持设置和删除与 保持周期无关。

      

##### 存储类

S3 支持如下：

1. S3 Standard：默认
2. RRS：减少冗余度，可能是 EC 编码
3. S3 Intelligent-Tiering：类似冷热分离存储，智能数据迁移
4. S3 Standard-IA：访问延时和 Standard 差不多，应该是省钱吧
5. S3 One Zone-IA ：同上，但是只在一个 zone 内
6. S3 Glacier：仅保存 90天，restore 1-5 min 之后被检索出来，也是降低成本吧？
7. S3 Glacier Deep Archive：180天、restore 12小时之后被检索
   Glacier 是一种归档，需要先 restore，之后才能访问，restore 的时间是 1-5min 和 12 小时