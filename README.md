# Linux-Kernel-Knowledge
Q1: 工作队列WQA排在WQB前面，WQA在运行过程中休眠后让出CPU，WQB有没有可能被调度？
A：没有可能。内核为每个cpu创建一个[events/x]线程，创建的工作队列WQ被加入到一个双向链表中，在worker_thread中依次执行，即使WQA休眠后调度出去，等调度回来后依然从WQA的休眠位置继续执行，每个WQ执行完毕，才会执行下一个WQ。
Q2: 添加seq_ops相关功能后，测试发现会导致rtnl_lock状态异常。加log发现，在执行rtnl_lock()前，已经是locked状态，执行rtnl_lock后，反而是unlocked状态。
===========================code segments=================================
pr_info("[%s:%d]===000 locked:%d\n",__func__,__LINE__, rtnl_is_locked());
        rtnl_lock();
pr_info("[%s:%d]===111 locked:%d\n",__func__,__LINE__, rtnl_is_locked());
============================log segments================================
Oct 10 10:27:37 localhost kernel: [ltt_if_register:1378]===000 locked:1
Oct 10 10:27:37 localhost kernel: [ltt_if_register:1380]===111 locked:0
为什么会出现这样的情况？
A：首先要了解rtnl_is_locked()的机制，它判断rtnl_mutex中count的值是否为1，1代表未锁定，其他值代表锁定。正常情况下，上锁之前，count的值是1，未锁定；上锁之后，count的值为0，已上锁。异常情况下，比如在某个操作中多执行了一次rtnl_unlock()，导致count的值为2，这时因为count!=1所以会被判定为已上锁状态；执行rtnl_lock()之后，count值减1成为了1，此时count==1，会被判定为未锁定状态。 从而出现了上述log体现的现象。
