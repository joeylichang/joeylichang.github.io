|__diskpath

​				|_.minio.sys

​						|_format.json（集群拓扑元数据）

​						|_Buckets

​								|_${bucket_name}

​											|_.metadata.bin (BucketMetadataSys)

​											|_.usage-cache.bin（dataUsageCache），每个 bucket 也有一个 dataUsageCache

​								|_.usage-cache.bin（dataUsageCache），一个 set 一个 dataUsageCache，按 bucket 维度统计

​								|_.usage-cache.bin（dataUsageCache），每个 Set 也有一个 dataUsageCache

​								|_.usage.json （DataUsageInfo），最终的汇总结果

​								|_.tracker.bin（dataUpdateTracker）boolmfilter 的数据



|_cachepath

​				|_.minio.sys

​						|_format.json（当前节点 cache 磁盘的元数据）

​				|_sha256hash(object/object)

​						|_cache.json（ObjectInfo）

​						|_part.1（单数据）

​						|_uuid（rang 分块上传数据）

