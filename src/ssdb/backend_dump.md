# 作用

dump到磁盘文件，分两种模式，一种是pattern维度，一种是slot维度。

# 原理

起一个新线程，按slot粒度扫描key，过滤（pattern模式）或者不过滤（slot模式）dump数据到文件内。

# 源码

```c++
/* dump之前先确认当前是否有dump任务 */
int BackendDump::running() {
	return tid == 0 ? 0 : 1;
}

int BackendDump::proc(const Bytes &pattern){
	log_info("start dump keys, pattern=%s", pattern.String().c_str());
  
  /* 线程参数 */
	struct run_arg *arg = new run_arg();
	arg->pattern = pattern.String();
	arg->backend = this;

	int err = pthread_create(&tid, NULL, &BackendDump::_dump_regex_thread, arg);
	if(err != 0){
		log_error("can't create thread: %s", strerror(err));
		tid = 0;
	}
	return tid == 0 ? 0 : 1;
}

int BackendDump::proc(int16_t slot) {
	log_info("start dump slot %d", slot);
  
  /* 线程参数 */
	struct run_arg *arg = new run_arg();
	arg->slot = slot;
	arg->backend = this;

	int err = pthread_create(&tid, NULL, &BackendDump::_dump_slot_thread, arg);
	if(err != 0) {
		log_error("can't create thread: %s", strerror(err));
		tid = 0;
	}
	return tid == 0 ? 0 : 1;
}

/* dump 文件名 */
#define KEYS_FILE_NAME "keys"

void* BackendDump::_dump_regex_thread(void *arg){
	pthread_detach(pthread_self());
	SET_PROC_NAME("backend_dump_regex");
	struct run_arg *p = (struct run_arg*)arg;
	BackendDump *backend = p->backend;
	std::string pattern = p->pattern;
	delete p;

	int unixtime = time(NULL);
	char tmpfile[256];
	snprintf(tmpfile, 256, "temp-%d.%ld.keys", unixtime, (long)(pthread_self()));
	int fd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
	if(fd == -1) {
		log_error("create temp file for keys failed");
		backend->tid = 0;
		return NULL;
	}

  /* 按slot粒度扫描S */
	for (int i = 0; i < CLUSTER_SLOTS; ++i) {
		Iterator *it = backend->ssdb->keys(i);
		if(it == NULL) {
			log_error("create iterator for keys in slot %d failed", i);
			backend->tid = 0;
			return NULL;
		}

    /* dump * */
		int allkeys = (pattern[0] == '*' && pattern[1] == '\0');
		while(it->next()) {
			std::string k;
			if (decode_version_key(it->key(), &k) != 0) {
				continue;
			}

      /* 前缀匹配 */
			if(allkeys ||  stringmatchlen(pattern.data(), pattern.size(), k.data(), k.size(),0)) {
				log_debug("write key %s", k.c_str());
				write(fd, k.data(), k.size());
				write(fd, "\n", 1);
			}
		}

		SAFE_DELETE(it);
	}
	close(fd);
	if(rename(tmpfile, KEYS_FILE_NAME) == -1) {
		log_error("failed to rename tmpfile to keys file");
	}
	backend->tid = 0;
	log_info("dump keys done");
	return NULL;
}


void* BackendDump::_dump_slot_thread(void *arg){
	pthread_detach(pthread_self());
	SET_PROC_NAME("backend_dump_slot");
	struct run_arg *p = (struct run_arg*)arg;
	BackendDump *backend = p->backend;
	int16_t slot = p->slot;
	delete p;

	int unixtime = time(NULL);
	char tmpfile[256];
	char targetfile[256];
	snprintf(tmpfile, 256, "temp-%d.%ld.slot-%d", unixtime, (long)(pthread_self()), slot);
	snprintf(targetfile, 256, "slot-%d", slot);
	int fd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
	if(fd == -1) {
		log_error("create temp file for keys failed");
		backend->tid = 0;
		return NULL;
	}

	Iterator *it = backend->ssdb->keys(slot);
	if(it == NULL) {
		log_error("create iterator for keys failed");
		backend->tid = 0;
		return NULL;
	}

	while(it->next()) {
		std::string k;
		if (decode_version_key(it->key(), &k) != 0) {
			continue;
		}

		log_debug("write key %s", k.c_str());
		write(fd, k.data(), k.size());
		write(fd, "\n", 1);
	}

	SAFE_DELETE(it);
	close(fd);
	if(rename(tmpfile, targetfile) == -1) {
		log_error("failed to rename tmpfile to keys file");
	}
	backend->tid = 0;
	log_info("dump slot %d done", slot);
	return NULL;
 }
```

