---
title: 【mgo】mgo源码解读——cluster
date: 2018-11-05 15:11:52
tags:
  - golang
  - mgo
---
## 定时更新服务器信息 —— syncServersLoop

```
// syncServersLoop loops while the cluster is alive to keep its idea of the server topology up-to-date.
// 当集群处于活动状态时，以循环来保持服务的实时拓扑结构。

// It must be called just once from newCluster.
// 必须从 newCluster 调用一次。

// The loop iterates once syncServersDelay has passed, or if somebody injects a value into the cluster.sync channel to force a synchronization.
// 一段时间（syncServersDelay）的延迟过后，循环会进行一次迭代。或者在 cluster.sync 管道中注入一个值，来强制执行一次同步操作。（从最后的select管道通信可以很容易看出）

// A loop iteration will contact all servers in parallel, ask them about known peers and their own role within the cluster, and then attempt to do the same with all the peers retrieved.
// 一次循环迭代将会✨并行✨访问所有服务，查询集群中已知的对等服务及自己的角色，然后对所有对等服务尝试同样的操作。
 
func (cluster *mongoCluster) syncServersLoop() {
    // ✨ 死循环
	for {
		debugf("SYNC Cluster %p is starting a sync loop iteration.", cluster)

		// ✨ 上锁！
		cluster.Lock()
		// ✨ 集群中没有服务了（全挂了？）
		if cluster.references == 0 {
			cluster.Unlock()
			break
		}
		cluster.references++ // Keep alive while syncing.
		// ✨ ？？？
		direct := cluster.direct
		cluster.Unlock()

		cluster.syncServersIteration(direct)

		// We just synchronized, so consume any outstanding requests.
		select {
		case <-cluster.sync:
		default:
		}

		cluster.Release()

		// Hold off before allowing another sync. No point in
		// burning CPU looking for down servers.
		if !cluster.failFast {
			time.Sleep(syncShortDelay)
		}

		cluster.Lock()
		if cluster.references == 0 {
			cluster.Unlock()
			break
		}
		cluster.syncCount++
		// Poke all waiters so they have a chance to timeout or
		// restart syncing if they wish to.
		cluster.serverSynced.Broadcast()
		// Check if we have to restart immediately either way.
		restart := !direct && cluster.masters.Empty() || cluster.servers.Empty()
		cluster.Unlock()

		if restart {
			log("SYNC No masters found. Will synchronize again.")
			time.Sleep(syncShortDelay)
			continue
		}

		debugf("SYNC Cluster %p waiting for next requested or scheduled sync.", cluster)

		// Hold off until somebody explicitly requests a synchronization
		// or it's time to check for a cluster topology change again.
		select {
		case <-cluster.sync:
		case <-time.After(syncServersDelay):
		}
	}
	debugf("SYNC Cluster %p is stopping its sync loop.", cluster)
}
```

## 同步服务info的核心

设计到的点

- sync.WaitGroup 并发控制
- sync.Mutex 这个简单

```
func (cluster *mongoCluster) syncServersIteration(direct bool) {
	log("SYNC Starting full topology synchronization...")

	var wg sync.WaitGroup
	var m sync.Mutex
	notYetAdded := make(map[string]pendingAdd)
	addIfFound := make(map[string]bool)
	seen := make(map[string]bool)
	syncKind := partialSync

	var spawnSync func(addr string, byMaster bool)
	spawnSync = func(addr string, byMaster bool) {
		// ✨ 标记开启了一个新线程
		wg.Add(1)
		go func() {
			defer wg.Done()

			// ✨ 解析地址，这个resolveAddr实现好🐂🍺，考虑的较全面
			// ✨ （不进去看代码了）先解析ip地址，不是ip地址就当初udp连接解析，因为拨号连接可能耗时，用了select管道并发处理，这里分别以udp4、udp6进行尝试
			tcpaddr, err := resolveAddr(addr)
			if err != nil {
				log("SYNC Failed to start sync of ", addr, ": ", err.Error())
				return
			}
			resolvedAddr := tcpaddr.String()

			// ✨ 已发现地址的逻辑，要加锁
			m.Lock()
			if byMaster {
				if pending, ok := notYetAdded[resolvedAddr]; ok {
					delete(notYetAdded, resolvedAddr)
					m.Unlock()
					cluster.addServer(pending.server, pending.info, completeSync)
					return
				}
				addIfFound[resolvedAddr] = true
			}
			// ✨ 如果已经发现该地址，stop
			if seen[resolvedAddr] {
				m.Unlock()
				return
			}
			// ✨ 标记当前地址已被发现
			seen[resolvedAddr] = true
			m.Unlock()

			server := cluster.server(addr, tcpaddr)
			// ✨ 同步server信息
			info, hosts, err := cluster.syncServer(server)
			if err != nil {
				// ✨ 同步信息出错，移除
				cluster.removeServer(server)
				return
			}

			m.Lock()
			add := direct || info.Master || addIfFound[resolvedAddr]
			if add {
				syncKind = completeSync
			} else {
				notYetAdded[resolvedAddr] = pendingAdd{server, info}
			}
			m.Unlock()
			if add {
				cluster.addServer(server, info, completeSync)
			}
			if !direct {
				for _, addr := range hosts {
					spawnSync(addr, info.Master)
				}
			}
		}()
	}

	knownAddrs := cluster.getKnownAddrs()
	for _, addr := range knownAddrs {
		spawnSync(addr, false)
	}
	wg.Wait()

	if syncKind == completeSync {
		logf("SYNC Synchronization was complete (got data from primary).")
		for _, pending := range notYetAdded {
			cluster.removeServer(pending.server)
		}
	} else {
		logf("SYNC Synchronization was partial (cannot talk to primary).")
		for _, pending := range notYetAdded {
			cluster.addServer(pending.server, pending.info, partialSync)
		}
	}

	cluster.Lock()
	mastersLen := cluster.masters.Len()
	logf("SYNC Synchronization completed: %d master(s) and %d slave(s) alive.", mastersLen, cluster.servers.Len()-mastersLen)

	// Update dynamic seeds, but only if we have any good servers. Otherwise,
	// leave them alone for better chances of a successful sync in the future.
	if syncKind == completeSync {
		dynaSeeds := make([]string, cluster.servers.Len())
		for i, server := range cluster.servers.Slice() {
			dynaSeeds[i] = server.Addr
		}
		cluster.dynaSeeds = dynaSeeds
		debugf("SYNC New dynamic seeds: %#v\n", dynaSeeds)
	}
	cluster.Unlock()
}
```