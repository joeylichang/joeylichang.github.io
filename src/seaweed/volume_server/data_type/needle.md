# Needle

```go
/* 上传数据时http参数执行下面的变量值
 * Cookie 是一个校验值，防止共计
 */
type Needle struct {
	Cookie Cookie   `comment:"random number to mitigate brute force lookups"`
	Id     NeedleId `comment:"needle id"`
	Size   uint32   `comment:"sum of DataSize,Data,NameSize,Name,MimeSize,Mime"`

	DataSize     uint32 `comment:"Data size"` //version2
	Data         []byte `comment:"The actual file data"`
	Flags        byte   `comment:"boolean flags"` //version2
	NameSize     uint8  //version2
	Name         []byte `comment:"maximum 256 characters"` //version2
	MimeSize     uint8  //version2
	Mime         []byte `comment:"maximum 256 characters"` //version2
	PairsSize    uint16 //version2
	Pairs        []byte `comment:"additional name value pairs, json format, maximum 64kB"`
	LastModified uint64 //only store LastModifiedBytesLength bytes, which is 5 bytes to disk
	Ttl          *TTL

	Checksum   CRC    `comment:"CRC32 to check integrity"`
	AppendAtNs uint64 `comment:"append timestamp in nano seconds"` //version3
	Padding    []byte `comment:"Aligned to 8 bytes"`
}
```

