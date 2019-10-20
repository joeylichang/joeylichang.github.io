# 作用

负责数据搬迁。数据搬迁流程如下：

1. target：slot_preimporting
2. source：slot_premigrating
3. target：migrate ip port slot
4. source：slot_postmigrating
5. target：slot_postimporting

# 原理

SSDBCluster只负责slot的标记（migrating、importing）与管理（当前节点负责的slot），搬迁的核心逻辑在RangeMigrate内部实现，通过搬迁的slot可以定位到需要搬迁数据的范围，逐个key进行搬迁，每搬迁一次都需要一个ACK确认。

# 源码

source核心逻辑：

```
void RangeMigrate::Client::proc() {
	owner->migrate_ret = 1;
	std::string msg;
	while(true) {
		switch(status) {
			case CLIENT_DISCONNECT:
				connect();
				break;
			case CLIENT_RECONNECT:
				reconnect();
				break;
			case CLIENT_CONNECTED:
				init();
				break;
			case CLIENT_INITIALIZED:
			case CLIENT_COPY:
				copy();
				break;
			case CLIENT_ACK:
				confirm_ack();		/* 内部调用key_migrate_init获取下一个需要搬迁的key */
				break;
			case CLIENT_EOF:
				confirm_eof();
				break;
			case CLIENT_ABORT:
				owner->migrate_ret = -1;
				msg = "abort";
				goto finish;
			case CLIENT_DONE:
				owner->migrate_ret = 0;
				msg = "done";
				goto finish;
			default:
				log_warn("unknown status");
				owner->migrate_ret = -1;
				msg = "abort";
				goto finish;
		}

		/* proc timeout only after full key migration */
		if(status == CLIENT_INITIALIZED && proc_timeout > 0 && time_ms() - proc_start > proc_timeout) {
			log_info("range migrate timeout");
			owner->migrate_ret = 0;
			msg = "continue";
			goto finish;
		}
	}

finish:
	/* block process, do not flush link here */
	resp->push_back(msg);
	owner->processing = 0;
}
```

target核心逻辑：

```
void RangeMigrate::Server::proc() {
	log_info("start range importing form %s:%d", link->remote_ip, link->remote_port);
	if (init() != 0) {
		log_error("range importing alignment failed");
		goto err;
	}
	while(true) {
		const std::vector<Bytes> *req = sync_read(&select, link, RANGE_MIGRATE_RECV_TIMEOUT);
		if(req == NULL) {
			log_error("link recv error: %s, importing abort", strerror(errno));
			goto err;
		} else {
			/* 各种同步数据的解析和处理逻辑 */
			int ret = proc_req(*req);
			if(ret == 1) {
				log_info("range importe done");
				return;
			} else if(ret != 0) {
				log_error("process request failed");
				goto err;
			}
		}
	}
err:
	log_error("range import failed");
}
```

