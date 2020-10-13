---
layout: post
title: "InterProcessSemaphoreV2源码阅读笔记"
subtitle: "InterProcessSemaphoreV2使用令牌桶的算法实现分布式信号量"
date: 2020-07-22
author: liying
category: zookeeper
tags: zookeeper,lock,java
finished: false 
---
## InterProcessSemaphoreV2源码

### InterProcessSemaphoreV2

```java
public class InterProcessSemaphoreV2 {
    private final Logger log;
    private final InterProcessMutex lock;
    private final WatcherRemoveCuratorFramework client;
    private final String leasesPath;
    private final Watcher watcher;
    private volatile byte[] nodeData;
    private volatile int maxLeases;
    private static final String LOCK_PARENT = "locks";
    private static final String LEASE_PARENT = "leases";
    private static final String LEASE_BASE_NAME = "lease-";
    public static final Set<String> LOCK_SCHEMA = Sets.newHashSet(new String[]{"locks", "leases"});
    static volatile CountDownLatch debugAcquireLatch = null;
    static volatile CountDownLatch debugFailedGetChildrenLatch = null;

    public InterProcessSemaphoreV2(CuratorFramework client, String path, int maxLeases) {
        this(client, path, maxLeases, (SharedCountReader)null);
    }

    public InterProcessSemaphoreV2(CuratorFramework client, String path, SharedCountReader count) {
        this(client, path, 0, count);
    }

    private InterProcessSemaphoreV2(CuratorFramework client, String path, int maxLeases, SharedCountReader count) {
        this.log = LoggerFactory.getLogger(this.getClass());
        this.watcher = new Watcher() {
            public void process(WatchedEvent event) {
                InterProcessSemaphoreV2.this.notifyFromWatcher();
            }
        };
        this.client = client.newWatcherRemoveCuratorFramework();
        path = PathUtils.validatePath(path);
        this.lock = new InterProcessMutex(client, ZKPaths.makePath(path, "locks"));
        this.maxLeases = count != null ? count.getCount() : maxLeases;
        this.leasesPath = ZKPaths.makePath(path, "leases");
        if (count != null) {
            count.addListener(new SharedCountListener() {
                public void countHasChanged(SharedCountReader sharedCount, int newCount) throws Exception {
                    InterProcessSemaphoreV2.this.maxLeases = newCount;
                    InterProcessSemaphoreV2.this.notifyFromWatcher();
                }

                public void stateChanged(CuratorFramework client, ConnectionState newState) {
                }
            });
        }

    }

    public void setNodeData(byte[] nodeData) {
        this.nodeData = nodeData != null ? Arrays.copyOf(nodeData, nodeData.length) : null;
    }

    public Collection<String> getParticipantNodes() throws Exception {
        return (Collection)this.client.getChildren().forPath(this.leasesPath);
    }

    public void returnAll(Collection<Lease> leases) {
        Iterator var2 = leases.iterator();

        while(var2.hasNext()) {
            Lease l = (Lease)var2.next();
            CloseableUtils.closeQuietly(l);
        }

    }

    public void returnLease(Lease lease) {
        CloseableUtils.closeQuietly(lease);
    }
	
    public Lease acquire() throws Exception {
        Collection<Lease> leases = this.acquire(1, 0L, (TimeUnit)null);
        return (Lease)leases.iterator().next();
    }

    public Collection<Lease> acquire(int qty) throws Exception {
        return this.acquire(qty, 0L, (TimeUnit)null);
    }

    public Lease acquire(long time, TimeUnit unit) throws Exception {
        Collection<Lease> leases = this.acquire(1, time, unit);
        return leases != null ? (Lease)leases.iterator().next() : null;
    }
		/**
		 * qty：请求令牌的数量
		 * time & unit : 超时时间和时间单位
		 **/
    public Collection<Lease> acquire(int qty, long time, TimeUnit unit) throws Exception {
        long startMs = System.currentTimeMillis();
        boolean hasWait = unit != null;
        long waitMs = hasWait ? TimeUnit.MILLISECONDS.convert(time, unit) : 0L;
        Preconditions.checkArgument(qty > 0, "qty cannot be 0");
      	// ImmutableList是线程安全的列表集合
        Builder<Lease> builder = ImmutableList.builder();
        boolean success = false;

        try {
         		//直到获取到所有的令牌
            while(qty-- > 0) {
                int retryCount = 0;
                long startMillis = System.currentTimeMillis();
                boolean isDone = false;
								//自旋获取锁
                while(!isDone) {
                    switch(this.internalAcquire1Lease(builder, startMs, hasWait, waitMs)) {
                    case CONTINUE:
                        isDone = true;
                        break;
                    case RETURN_NULL:
                        Object var16 = null;
                        return (Collection)var16;
                    case RETRY_DUE_TO_MISSING_NODE:
                        if (!this.client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper())) {
                            throw new NoNodeException("Sequential path not found - possible session loss");
                        }
                    }
                }
            }

            success = true;
            return builder.build();
        } finally {
            if (!success) {
                this.returnAll(builder.build());
            }

        }
    }

    private InterProcessSemaphoreV2.InternalAcquireResult internalAcquire1Lease(Builder<Lease> builder, long startMs, boolean hasWait, long waitMs) throws Exception {
        if (this.client.getState() != CuratorFrameworkState.STARTED) {
            return InterProcessSemaphoreV2.InternalAcquireResult.RETURN_NULL;
        } else {
            if (hasWait) {
              	//thisWaitMs 还剩下的超时时间
                long thisWaitMs = this.getThisWaitMs(startMs, waitMs);
              	//使用InterProcessMutex的acquire(time,unit)方法
                if (!this.lock.acquire(thisWaitMs, TimeUnit.MILLISECONDS)) {
                  	//获取不到则返回RETURN_NULL
                    return InterProcessSemaphoreV2.InternalAcquireResult.RETURN_NULL;
                }
            } else {
              	//使用InterProcessMutex的acquire方法，获取不到则会抛出IOException
                this.lock.acquire();
            }

            Lease lease = null;

            try {
              	//withProtection：在路径中增加uuid
                PathAndBytesable<String> createBuilder = (PathAndBytesable)this.client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL);
              	//拼接path为锁的basepath+“/”+"lease-"，uuid添加到lease前
                String path = this.nodeData != null ? (String)createBuilder.forPath(ZKPaths.makePath(this.leasesPath, "lease-"), this.nodeData) : (String)createBuilder.forPath(ZKPaths.makePath(this.leasesPath, "lease-"));
              	//截取nodeName为uuid+“lease-”+序号
                String nodeName = ZKPaths.getNodeFromPath(path);
                lease = this.makeLease(path);
                if (debugAcquireLatch != null) {
                    debugAcquireLatch.await();
                }

                try {
                    synchronized(this) {
                        while(true) {
                            List children;
                            Object e;
                            try {
                                children = (List)((BackgroundPathable)this.client.getChildren().usingWatcher(this.watcher)).forPath(this.leasesPath);
                            } catch (Exception var30) {
                                e = var30;
                                if (debugFailedGetChildrenLatch != null) {
                                    debugFailedGetChildrenLatch.countDown();
                                }

                                this.returnLease(lease);
                                throw var30;
                            }

                            if (!children.contains(nodeName)) {
                                this.log.error("Sequential path not found: " + path);
                                this.returnLease(lease);
                                e = InterProcessSemaphoreV2.InternalAcquireResult.RETRY_DUE_TO_MISSING_NODE;
                                return (InterProcessSemaphoreV2.InternalAcquireResult)e;
                            }

                            if (children.size() <= this.maxLeases) {
                                break;
                            }

                            if (hasWait) {
                                long thisWaitMs = this.getThisWaitMs(startMs, waitMs);
                                if (thisWaitMs <= 0L) {
                                    this.returnLease(lease);
                                    InterProcessSemaphoreV2.InternalAcquireResult var18 = InterProcessSemaphoreV2.InternalAcquireResult.RETURN_NULL;
                                    return var18;
                                }

                                this.wait(thisWaitMs);
                            } else {
                                this.wait();
                            }
                        }
                    }
                } finally {
                    this.client.removeWatchers();
                }
            } finally {
                this.lock.release();
            }

            builder.add(Preconditions.checkNotNull(lease));
            return InterProcessSemaphoreV2.InternalAcquireResult.CONTINUE;
        }
    }

    private long getThisWaitMs(long startMs, long waitMs) {
        long elapsedMs = System.currentTimeMillis() - startMs;
        return waitMs - elapsedMs;
    }

    private Lease makeLease(final String path) {
        return new Lease() {
            public void close() throws IOException {
                try {
                    ((ChildrenDeletable)InterProcessSemaphoreV2.this.client.delete().guaranteed()).forPath(path);
                } catch (NoNodeException var2) {
                    InterProcessSemaphoreV2.this.log.warn("Lease already released", var2);
                } catch (Exception var3) {
                    ThreadUtils.checkInterrupted(var3);
                    throw new IOException(var3);
                }

            }

            public byte[] getData() throws Exception {
                return (byte[])InterProcessSemaphoreV2.this.client.getData().forPath(path);
            }

            public String getNodeName() {
                return ZKPaths.getNodeFromPath(path);
            }
        };
    }

    private synchronized void notifyFromWatcher() {
        this.notifyAll();
    }

    private static enum InternalAcquireResult {
        CONTINUE,
        RETURN_NULL,
        RETRY_DUE_TO_MISSING_NODE;

        private InternalAcquireResult() {
        }
    }
}

```

参考：

1. [InterProcessSemaphoreV2核心源码解读（分布式信号量的实现）](https://www.jianshu.com/p/3022a593651a)

