# 作用

删除过期key（通过命令设置过ttl的key）。

# 原理

起一个线程定时（默认10s）检查是否有过期key需要删除。所有设置ttl的key都会hash到一个槽位上（默认1000个槽位），每个操作缓存一定数量的key（默认100个），每隔10s检查这些槽位，看是否有超时的key，有则删除。

# 源码

### set_ttl

```c++
int ExpirationHandler::set_ttl(const Bytes &key, int64_t ttl){
	uint32_t idx = string_hash(key) >> (32-EXPIR_CON_DEGREE);
	Locking l(&mutexs[idx]);

	int64_t expired = time_ms() + ttl * 1000;
	char data[30];
	int size = snprintf(data, sizeof(data), "%" PRId64, expired);
	if(size <= 0){
		log_error("snprintf return error!");
		return -1;
	}

	int ret = 0;
	{
		// no row lock, pervent from dead lock
		Transaction trans(ssdb, Bytes());
    // list_name 槽位的前缀（leveldb的前缀）
		ret = ssdb->zset(this->list_name[idx], key, Bytes(data, size), trans, 0);
	}
	if(ret == -1){
		return -1;
	}

  // 槽位第一个超时的key
	if(expired < first_timeout[idx]){
		first_timeout[idx] = expired;
	}

	std::string s_key = key.String();
	if(!fast_keys[idx].empty() && expired <= fast_keys[idx].max_score()){
		fast_keys[idx].add(s_key, expired);
		if(fast_keys[idx].size() > BATCH_SIZE){
			log_debug("pop_back");
			fast_keys[idx].pop_back();
		}
	}else{
		fast_keys[idx].del(s_key);
		//log_debug("don't put in fast_keys");
	}

	return 0;
}
```



### thread_func

```
void* ExpirationHandler::thread_func(void *arg){
	SET_PROC_NAME("expiration_handler");
	ExpirationHandler *handler = (ExpirationHandler *)arg;

	while(!handler->thread_quit){
		bool immediately = false;
		int64_t now = time_ms();
		
		// 判断是否有槽位超时
		for (int i = 0; i < EXPIR_CONCURRENT; i++) {
			if (handler->first_timeout[i] <= now) {
				immediately = true;
				break;
			}
		}

		if(!immediately){
			usleep(10 * 1000);
			continue;
		}

		handler->expire_loop();
	}

	log_debug("ExpirationHandler thread quit");
	return (void *)NULL;
}
```



### expire_loop

```c++
void ExpirationHandler::expire_loop(){

	// for each partition
	for (int i = 0; i < EXPIR_CONCURRENT; i++) {

		Locking l(&this->mutexs[i]);
		if(!this->ssdb){
			return;
		}

		// 如果槽位空了从leveldb中读取BATCH_SIZE（100）个
		if(this->fast_keys[i].empty()){
			this->load_expiration_keys_from_db(i, BATCH_SIZE);
			if(this->fast_keys[i].empty()){
				this->first_timeout[i] = INT64_MAX;

				continue;
			}
		}

		int64_t now = time_ms();
		int64_t score;
		std::string key;
		int max = 5; // prevent from block set_ttl && del_ttl too long
		while (max-- > 0 && this->fast_keys[i].front(&key, &score)) {
			this->first_timeout[i] = score;

			// 有序列表存储，超时
			if(score <= now){
				log_debug("expired %s", key.c_str());
				{
					// no row lock, prevent from dead lock
					Transaction trans(ssdb, Bytes());
					
					// 删除数据
					ssdb->del(key, trans);
					if (binlog) {
						// 同步给从
						binlog->write(BinlogType::SYNC, BinlogCommand::K_DEL, key);
					}
				}
				{
					// no row lock, prevent from dead lock
					Transaction trans(ssdb, Bytes());
					
					// 删除元数据
					ssdb->zdel(this->list_name[i], key, trans, 0);
					this->fast_keys[i].pop_front();
				}

			} else {
				break;
			}
		}

	}/* end of for */
}
```



