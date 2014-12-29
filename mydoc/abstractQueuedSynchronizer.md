ReentrantLock/CountDownLatch/Semaphore/FutureTask/ThreadPoolExecutor的源码中都会包含一个静态的内部类Sync，它继承了AbstractQueuedSynchronizer这个抽象类。

AbstractQueuedSynchronizer是java.util.concurrent包中的核心组件之一，为并发包中的其他synchronizers提供了一组公共的基础设施。

AQS会对进行acquire而被阻塞的线程进行管理，其管理方式是在AQS内部维护了一个FIFO的双向链表队列，队列的头部是一个空的节点，除此之外，每个节点持有着一个线程，节点中包含两个重要的属性waiteStatus和nextWaiter。节点的数据结构如下：
Node中的属性waitStatus、prev、next、thread都使用了volatile修饰，这样直接的读写操作就具有内存可见性。
waitStatus表示了当前节点Node的状态

```
static final class Node {
        /** waitStatus的值，表示此节点中的线程被取消 */
        static final int CANCELLED =  1;
        /** waitStatus value 表明后续节点中的线程需要unparking 唤醒 */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;
        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node until
         *               transferred. (Use of this value here
         *               has nothing to do with the other uses
         *               of the field, but simplifies mechanics.)
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified only using
         * CAS.
         */
        volatile int waitStatus;
        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        volatile Node prev;
        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned once during enqueuing, and
         * nulled out (for sake of GC) when no longer needed.  Upon
         * cancellation, we cannot adjust this field, but can notice
         * status and bypass the node if cancelled.  The enq operation
         * does not assign next field of a predecessor until after
         * attachment, so seeing a null next field does not
         * necessarily mean that node is at end of queue. However, if
         * a next field appears to be null, we can scan prev's from
         * the tail to double-check.
         */
        volatile Node next;
        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        volatile Thread thread;
        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;
        /**
         * Returns true if node is waiting in shared mode
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        /**
         * Returns previous node, or throws NullPointerException if
         * null.  Use when predecessor cannot be null.
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
        Node() {    // Used to establish initial head or SHARED marker
        }
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

nextWaiter一般是作用于在使用Condition时的队列。

java.util.concurrent.locks.LockSupport
