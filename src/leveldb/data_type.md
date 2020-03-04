# LevelDB 元信息

DB在每次compact之后都有sst等元信息变更（增加、删除等）。snapshot针对某一时刻的db数据快照，每次compact之后又不能删除之前snapshot的数据，需要一个机制保证DB的变更清理无用数据，保留必要数据。

在leveldb中，VersionEdit记录了每次compact的变更，VersionEdit作用在version上形成当前可用的Version，多个Version组成了VersionSet保证了多版本的共存，调用db的open之后会通过mainfest文件恢复相关信息。

下面介绍VersionEdit、Version、VersionSet 类的成员变量，后续流程会介绍其成员函数的细节。

###### VersionEdit

```c++
class VersionEdit {
  typedef std::set< std::pair<int, uint64_t> > DeletedFileSet;		// 删除文件的map，level-> file_number

  std::string comparator_;			// 比较器，可以用户自定义，默认字典序
  uint64_t log_number_;				// 当前binlog的file_number，compact level 0时用到
  uint64_t prev_log_number_;			// 前一个binlog的file_number
  uint64_t next_file_number_;			// 下一个binlog的file_number
  SequenceNumber last_sequence_;		// binlog的seq
  bool has_comparator_;							
  bool has_log_number_;							
  bool has_prev_log_number_;				
  bool has_next_file_number_;				
  bool has_last_sequence_;					

  std::vector< std::pair<int, InternalKey> > compact_pointers_;		// 记录压缩的进度，level -> key
  DeletedFileSet deleted_files_;					// 压缩后需要删除的文件
  std::vector< std::pair<int, FileMetaData> > new_files_;		// 压缩后新生成的文件
}
```



###### Version

```C++
class Version {
	VersionSet* vset_;      // VersionSet to which this Version belongs
  Version* next_;               // Next version in linked list
  Version* prev_;               // Previous version in linked list
  int refs_;                    // Number of live refs to this version

  // List of files per level
  std::vector<FileMetaData*> files_[config::kNumLevels];

  // Next file to compact based on seek stats. 
  // 1. 压缩条件有两个一个score， 根据level的文件大小 和 预期大小
  //    level = 0, 8M
  //    levle = 1, 10M,  size = 10M的1次幂
  //    levle = 2, 100M, size = 10M的2次幂
  //    一次类推，默认7层，10T
  // 2. 另一个是seek的次数，每次读算一个seek，默认1 << 30次
  FileMetaData* file_to_compact_;
  int file_to_compact_level_;

  // Level that should be compacted next and its compaction score.
  // Score < 1 means compaction is not strictly needed.  These fields
  // are initialized by Finalize().
  double compaction_score_;
  int compaction_level_;
}

// sst文件的原信息
struct FileMetaData {
  int refs;
  int allowed_seeks;          // Seeks allowed until compaction
  uint64_t number;
  uint64_t file_size;         // File size in bytes
  InternalKey smallest;       // Smallest internal key served by table
  InternalKey largest;        // Largest internal key served by table

  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) { }
};
```



###### VersionSet

```c++
class VersionSet {
  Env* const env_;				// leveldb 考虑到跨平台兼容性对系统调用做了一层封装
  const std::string dbname_;			// open时指定
  const Options* const options_;		// 用户传入的自定义选项
  TableCache* const table_cache_;		// 可以通过Options自定义，默认lru
  const InternalKeyComparator icmp_;		// 可以通过Options自定义，默认字典序
  uint64_t next_file_number_;			// sst文件号，递增
  uint64_t manifest_file_number_;		// 自增
  uint64_t last_sequence_;			// 
  uint64_t log_number_;				// 
  uint64_t prev_log_number_;  			// 0 or backing store for memtable being compacted

  // Opened lazily
  WritableFile* descriptor_file_;		// manifest
  log::Writer* descriptor_log_;         	// 对descriptor_file_写入的封装
  Version dummy_versions_;  			// Head of circular doubly-linked list of versions.
  Version* current_;        			// == dummy_versions_.prev_

  // Per-level key at which the next compaction at that level should start.
  // Either an empty string, or a valid InternalKey.
  std::string compact_pointer_[config::kNumLevels];
}
```





