# 作用

slave同步master的数据。

# 原理

slave主动向master发送sync140命令（附带last_seq、last_key），master向其同步数据，如果差距过大可能同步snapshot。

单独线程运行backend_sync（一个slave对应一个线程），并且还有一个定时线程清理snapshot（默认1小时）。

# 源码

```
void* BackendSync::_run_thread(void *arg){
	pthread_detach(pthread_self());
	SET_PROC_NAME("backend_sync2");
	struct run_arg *p = (struct run_arg*)arg;
	BackendSync *backend = (BackendSync *)p->backend;
	Link *link = (Link *)p->link;
	delete p;
	SSDBServer *server = backend->owner;
	int idle = 0;
	struct timeval now;
	struct timespec ts;

	// block IO
	link->noblock(false);

	Client client(backend);
	client.link = link;
	client.init();

	log_info("accept slave(%s:%d)", link->remote_ip, link->remote_port);

	{
		pthread_t tid = pthread_self();
		Locking l(&backend->mutex);
		backend->workers[tid] = &client;
	}

	if (client.status == Client::SYNC)
		goto binlog;

snapshot:
	client.status = Client::COPY;

	// pre snapshot
	if (client.pre_snapshot() != 0) {
		log_error("pre snapshot failed.");
		goto finished;
	}

	log_info("begin transfer snapshot seq(%" PRIu64 ")", client.last_seq);

	// transfer snapshot
	while (!backend->thread_quit) {

		if (client.status == Client::SYNC) {
			log_info("snapshot finished, ready to get binlog.");
			break;
		}

		if (client.copy() <= 0) continue;

		float data_size_mb = link->output->size() / 1024.0 / 1024.0;
		if (link->flush() == -1) {
			log_error("%s:%d, fd(%d) send error.", link->remote_ip,
						link->remote_port, link->fd());

			// mark snapshot abort, for breakpoint
			backend->mark_snapshot(client.host, CopySnapshot::ABORT);
			goto finished;
		}

		if (backend->sync_speed > 0) {
			usleep((data_size_mb/backend->sync_speed) * 1000 * 1000);
		}
	} // end while

	if (backend->thread_quit) {
		goto finished;
	}

	log_info("finished transfer snapshot seq(%" PRIu64 ")", client.last_seq);

	// post snapshot
	client.post_snapshot();

binlog:
	// pre binlog
	if (client.pre_binlog() != 0) {
		log_error("pre binlog failed, reset.");
		client.reset();
		goto snapshot;
	}

	// binlog
	while (!backend->thread_quit) {
		if (client.status != Client::SYNC) {
			log_error("out of sync, reset");
			client.reset();
			goto snapshot;
		}

		LogEvent event;
		unsigned char v;
		int ret = client.read(&event);
		if (ret == 0) { // EOF
			if (!server->binlog->is_active_log(client.logfile.filename)) {
				if (client.next_binlog("") != 0) {
					log_error("walk to next binlog failed. current binlog(%s).",
							client.logfile.filename.c_str());
					goto finished;
				}
				continue;
			}

			pthread_mutex_lock(&server->binlog->mutex);
			gettimeofday(&now, NULL);
			ts.tv_sec = now.tv_sec + 1;
			ts.tv_nsec = now.tv_usec * 1000;
			int err = pthread_cond_timedwait(&server->binlog->cond,
					&server->binlog->mutex, &ts);
			pthread_mutex_unlock(&server->binlog->mutex);

			if (err == ETIMEDOUT) {
				if (++idle > 30) {
					idle = 0;
					client.noop();
					goto flushdata;
				}
			}
			continue;
		}

		idle = 0;

		// binlog error
		if (ret != 1) {
			log_error("read binlog error");
			client.status = Client::OUT_OF_SYNC;
			continue;
		}

		// deal with event
		switch (event.cmd()) {
		case BinlogCommand::ROTATE:
			if (client.next_binlog(std::string(event.key().data(),
							event.key().size())) != 0) {
				log_error("walk to next binlog(%s) failed.", event.key().data());
				goto finished;
			}
			log_info("encounter rotate event. next file (%s)", event.key().data());
			continue;

		case BinlogCommand::STOP:
			if (client.next_binlog("") != 0) {
				log_info("master stopped, and no more binlog");
				goto finished;
			}
			log_info("encounter stop event.");
			continue;

		case BinlogCommand::DESC:
			/* skip description section */
			log_debug("encounter description event.");
			continue;

		default:
			log_debug("backend_sync2 event comand(%" PRIu8 ")", (unsigned char)event.cmd());
			break;
		}

		v = (unsigned char)event.cmd();
		if(v < SSDB_SYNC_CMD_MIN || v > SSDB_SYNC_CMD_MAX) {
			log_debug("skip invalidate event command(%" PRIu8 ")", v);
			continue;
		}

		client.last_seq = event.seq();
		link->send(event.repr());

flushdata:
		if (link->flush() == -1) {
			log_error("%s:%d, fd(%d) send error.", link->remote_ip,
						link->remote_port, link->fd());
			goto finished;
		}
	} // end while

finished:
	log_info("Sync Client quit, %s:%d fd: %d, delete link",
			link->remote_ip, link->remote_port, link->fd());
	delete link;
	client.logfile.close();

	Locking l(&backend->mutex);
	backend->workers.erase(pthread_self());
	return (void *)NULL;
}
```

