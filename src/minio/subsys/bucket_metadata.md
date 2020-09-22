### BucketMetadata

```go
type BucketMetadata struct {
	Name                    string 			// Bucket Name
	Created                 time.Time 	// 创建时间
	LockEnabled             bool 				// 历史遗留，不再使用
  
  /* 
   * 1. []byte 与下面的配置一一对应
   * 2. 下面的配置在之前的版本中都对应一个配置文件，在最新的版本中都集中在
   * 	  /diskpath/.minio.sys/buckets/${bucket_name}/.metadata.bin 中
   * 3. []byte 用于序列化反序列落盘或者load，下面的配置用于内存操作
   */
	PolicyConfigJSON        []byte 				
	NotificationConfigXML   []byte
	LifecycleConfigXML      []byte
	ObjectLockConfigXML     []byte
	VersioningConfigXML     []byte
	EncryptionConfigXML     []byte
	TaggingConfigXML        []byte
	QuotaConfigJSON         []byte
	ReplicationConfigXML    []byte
	BucketTargetsConfigJSON []byte

	// Unexported fields. Must be updated atomically.
	policyConfig       *policy.Policy 			 	
	notificationConfig *event.Config 				
	lifecycleConfig    *lifecycle.Lifecycle 		
	objectLockConfig   *objectlock.Config 			// "object-lock-enabled.json"	
	versioningConfig   *versioning.Versioning 		
	sseConfig          *bucketsse.BucketSSEConfig 	
	taggingConfig      *tags.Tags 					
	quotaConfig        *madmin.BucketQuota 			
	replicationConfig  *replication.Config 			
	bucketTargetConfig *madmin.BucketTarget 		
}
```



程序初始阶段在 SubSys 初始化步骤会加载所有的 Bucket 的上述 Meta 信息（注意一个 bucket 对应一个 meta数据），所有的 Bucket 并发去 Load，并发度最大是 100。

上述绝大部分的配置，支撑各自的子系统，其主要目的是兼容 S3 的语义，在请求到来时进行约束校验，只有检验通过了才能进行请求的操作，否则拒绝。下面看一下所有的子系统的介绍。



##### policyConfig

```go
type Policy struct {
	ID         ID `json:"ID,omitempty"`
	Version    string
	Statements []Statement `json:"Statement"`
}
```

S3 的权限管理，分两部分，既 基于资源的访问策略（Bucket、Object 级别）、基于用户的访问策略（IAM子系统），在 BucketMetadata 中主要是基于 Bucket 的访问控制策略，在所有关于 Bucket 或者 涉及到 Bucket 的请求到来时，都会去根据这个策略去判断是否被允许，其中 Statements 就是表示一个一个的约束，min.io 中 pkg 详细的实现了 Statement 的约束判断逻辑。

例如：如下配置表示允许所有用户对 Bucket 进行读。详细信息可以查看 S3 官网。

```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Sid":"PublicRead",
      "Effect":"Allow",
      "Principal": "*",
      "Action":["s3:GetObject","s3:GetObjectVersion"],
      "Resource":["arn:aws:s3:::awsexamplebucket1/*"]
    }
  ]
}
```



##### notificationConfig

Notification 主要用于兼容 S3 的通知机制，针对 Bucket 的一些关注事件进行监听。

例如：如下是配置监听关于bucket 内部有前缀为 images/，后缀为 jpg 的 Object 被创建之后，将消息发送到 AWS 的 queue（SQS）中。

```xml
<NotificationConfiguration>
  <QueueConfiguration>
      <Id>1</Id>
      <Filter>
          <S3Key>
              <FilterRule>
                  <Name>prefix</Name>
                  <Value>images/</Value>
              </FilterRule>
              <FilterRule>
                  <Name>suffix</Name>
                  <Value>jpg</Value>
              </FilterRule>
          </S3Key>
     </Filter>
     <Queue>arn:aws:sqs:us-west-2:444455556666:s3notificationqueue</Queue>
     <Event>s3:ObjectCreated:Put</Event>
  </QueueConfiguration>
</NotificationConfiguration>
```



##### lifecycleConfig

Lifecycle 可以针对 Bucket 内部的 Object 进行生命周期配置，如下配置表示，在bucket 内 前缀为 documents 且 enable 状态的 存储在 GLACIER（S3 分位很多种存储类型，主要是针对场景进行优化和成本折中考虑）的对象保存 365天。同样 Lifecycle 的实现也在 功能的 pkg 部分封装成单独的库，可用于辅助程序的开发（例如 Cli 等运维工具）。

```xml
<LifecycleConfiguration>
    <Rule>
        <ID>ExampleRule</ID>
        <Filter>
           <Prefix>documents/</Prefix>
        </Filter>
        <Status>Enabled</Status>
        <Transition>        
           <Days>365</Days>        
           <StorageClass>GLACIER</StorageClass>
        </Transition>    
        <Expiration>
             <Days>3650</Days>
        </Expiration>
    </Rule>
</LifecycleConfiguration>
```



##### objectLockConfig

对象锁的是否开启的配置。S3 支持的对象是用户级别的锁，目的是支持对象的 worm，既一次写多读，只有用户解锁了才能删除，对象锁分位两种一种锁，一种是合法持有，前者又分位两种模式 Governance 和 Compliance。

1. 对象锁
   1. Governance：只有有相应权限的账户可以更新、删除保持，或者被该用户授予相应权限的用户。
   2. Compliance：必须与一个过期保持周期绑定，过期之前任何账户都不能删除，并且保持周期只能延长不能缩短。
   3. 锁周期：可以针对 Object 单独设置，也可以使用 Bucket 的默认配置。
2. 合法持有
   1. 只要有 s3:PutObjectLegalHold 权限的账户都可以对一个 Object 进行合法持有，持有期间不会被删除和覆盖，只读状态。
3. 注意：合法持有 与 对象锁互相不干扰，删除合法保持，如果保持周期未到，仍不会被任何用户删除。
   		同样，合法持有设置和删除 与 保持周期无关。

用户针对对象锁的操作在用于的元数据（UserDefine）中需要制定管管的参数：AmzObjectLockMode、AmzObjectLockRetainUntilDate、X-Amz-Bypass-Governance-Retention等等。



##### versioningConfig

Versioning 支持对象保存多个版本，Config 中配置主要包含两项，一个是是否启用，另一个是是否开启多重验证删除。

1. 如果是删除对象，则插入一个删除标记在最新的版本中，并且可以随时恢复之前的任意版本。
2. 如果筛覆盖对象，也是插入新的版本，而不是真正的覆盖。
3. 开启多重验证，在永久删除对象和更改版本配置时，会验证 IAM 信息和序列号、密码（S3控制台可操作）等。



##### sseConfig

配置是否启动 server-side-encryption，S3 的加密分为两类既客户端加密 和 服务端加密，前者用户自己管理秘钥。服务端加密分为一下几种：

1. SSE-S3：所有对象的秘钥一样，加密秘钥的主密钥会定期轮换重新加密对象的秘钥，使用AES256进行加密。
2. SSE-KMS：CMK（S3 提供的服务）服务存储用户的主密钥，可以进行审计查看（时间、那个用户做了什么操作）。
3. SSE-C：用户自己提供加密秘钥，在请求中作为参数带进来，也是 AES256进行加密。

目前 min.io server 仅支持 SSE-C，gateway（不是本部分的重点） 方式支持 SSE-KMS。在所有请求到来，在 http handler 层进行加密认证和加解密逻辑。

注意：S3中仅对数据进行加密，对象元数据不进行加密，min.io 特进行了加密，并且默认都开启了 SSE-C。



##### taggingConfig

给 Bucket 设置 tag （可以是多个），标记Bucket 。在整个 Minio 中只有查询接口，在请求逻辑中并没有结合其做任何校验约束等。



##### quotaConfig

配置中，quota 的配置分位两种类型，一种是 HardQuota 只硬配额，另一种 FIFOQuota 从bucket中删除旧文件的配额限制。

例如：在 PutObject 流程中会获取配额的配置，然后判断如果是HardQuota（FIFOQuota 不做校验直接返回），根据写入的 size 和 DataUsageInfo（定期收集的信息）合并 check 是否超过 HardQuota 限制。



##### replicationConfig && bucketTargetConfig

Replicate 的作用是将写入的写对象复制到另一个存储桶，可以同区域复制（SRR）、跨区域复制（CRR）。前者使用场景包括：日志汇总、配置生产环境与测试环境数据一致等，后者可以减少不同地域上游服务的访问延迟等。二者共同的一个场景是分级存储，既复制到另一个低成本的存储桶中，用于性能要求不高的访问服务使用。

S3 的语义中 Replicate 保证副本的元数据与源副本一致（创建时间、版本号等）、时效性在 15min 以内、两个桶各自维护账户所有权（相同数据，不同副本，账户权限不一致）、分级存储（如上所述，存储介质的不同控制性能与成本的平衡）。

在配置中必要配置包含源与目标桶、IAM 权限：

```xml
<ReplicationConfiguration>
    <Role>IAM-role-ARN</Role>
    <Rule>
        <ID>Rule-1</ID>
        <Status>rule-Enabled-or-Disabled</Status>
        <Priority>integer</Priority>
        <DeleteMarkerReplication>
           <Status>Disabled</Status>
        </DeleteMarkerReplication>
        <Destination>        
           <Bucket>arn:aws:s3:::bucket-name</Bucket> 
        </Destination>    
    </Rule>
    <Rule>
         ...
    </Rule>
</ReplicationConfiguration>
```

bucketTargetConfig 是获取赋值 target  bucket 的信息。