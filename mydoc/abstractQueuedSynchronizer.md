ReentrantLock/CountDownLatch/Semaphore/FutureTask/ThreadPoolExecutor的源码中都会包含一个静态的内部类Sync，它继承了AbstractQueuedSynchronizer这个抽象类。

AbstractQueuedSynchronizer是java.util.concurrent包中的核心组件之一，为并发包中的其他synchronizers提供了一组公共的基础设施。

AQS会对进行acquire而被阻塞的线程进行管理，其管理方式是在AQS内部维护了一个FIFO的双向链表队列，队列的头部是一个空的结点，除此之外，每个结点持有着一个线程，结点中包含两个重要的属性waiteStatus和nextWaiter。结点的数据结构如下：
Node中的属性waitStatus、prev、next、thread都使用了volatile修饰，这样直接的读写操作就具有内存可见性。
waitStatus表示了当前结点Node的状态

```
static final class Node {
        /** waitStatus的值，表示此结点中的线程被取消 */
        static final int CANCELLED =  1;
        /** waitStatus value 表明后续结点中的线程需要unparking 唤醒 */
        static final int SIGNAL    = -1;
        /** waitStatus value 表明当前结点中的线程需要等待一个条件*/
        static final int CONDITION = -2;
        /** 表明结点是以共享模式进行等待(shared mode)的标记*/
        static final Node SHARED = new Node();
        /** 表明结点是以独占模式进行等待(exclusive mode)的标记*/
        static final Node EXCLUSIVE = null;
        /**
         * Status field, taking on only the values:
         *   SIGNAL: 后继结点现在(或即将)被阻塞(通过park) 那么当前结点在释放或者被取消的时候必须unpark它的后继结点
         *           为了避免竞态条件，acquire方法必须首先声明它需要一个signal，然后尝试原子的acquire
         *            如果失败了 就阻塞     
         *   CANCELLED:当前结点由于超时或者中断而被取消  结点不会脱离这个状态   
         *              尤其是，取消状态的结点中的线程永远不会被再次阻塞
         *   CONDITION: 当前结点在一个条件队列中。它将不会进入sync队列中直到它被transferred
         *              (这个值在这里的使用只是为了简化结构 跟其他字段的使用没有任何关系)
         *   0:          None of the above 非以上任何值
         *
         * 这些值通过数字来分类达到简化使用的效果
         * 非负的数字意味着结点不需要信号signal 这样大部分的代码不需要检查特定的值 just 检查符号就ok了
         *
         * 这个字段对于普通的sync结点初始化为0 对于条件结点初始化为CONDITION(-2) 本字段的值通过CAS操作进行修改
         */
        volatile int waitStatus;
        /**
         * 连接到当前结点或线程依赖的用于检查waitStatus等待状态的前驱结点。
         * 进入队列时赋值，出队列时置空(为GC考虑)。
         * 根据前驱结点的取消(CANCELLED),我们查找一个非取消结点的while循环短路，将总是会退出 ；
         * 因为头结点永远不会被取消：一个结点成为头结点只能通过一次成功过的acquire操作的结果
         * 一个取消的线程永远不会获取操作成功(acquire操作成功)
         * 一个线程只能取消它自己  不能是其他结点
         */
        volatile Node prev;
        /**
         * 连接到当前结点或线程释放时解除阻塞(unpark)的后继结点
         * 入队列时赋值，出队列时置空(为GC考虑)
         * 入队列时不会给前驱结点的next字段赋值,需要确认compareAndSetTail(pred, node)操作是否成功 (详见Node addWaiter(Node mode)方法)
         * 所以当我们发现结点的next为空时不一定就是tail尾结点 如果next为空，可以通过尾结点向前遍历即addWaiter中调用的enq(node)方法(个人觉
         * 这是对第一次处理失败的亡羊补牢之举)官方说法double-check 双层检查
         * 
         * 被取消的结点next指向的是自己而不是空(详见cancelAcquire(Node node)中最后的node.next = node; )这让isOnSyncQueue变得简单
         * Upon cancellation, we cannot adjust this field, but can notice
         * status and bypass the node if cancelled.  
         */
        volatile Node next;
        /**
         * 入队列结点中的线程,构造时初始化，使用完 就置空
         */
        volatile Thread thread;
        /**
         * 连接到下一个在条件上等待的结点 或者waitStatus为特殊值SHARED 共享模式
         * 因为条件队列只有在独占模式(exclusive)下持有时访问,当结点等待在条件上,我们只需要一个简单的链表队列来持有这些结点
         * 然后他们会转移到队列去进行re-acquire操作。
         * 由于条件只能是独占的,我们可以使用一个特殊的值来声明共享模式(shared mode)来节省一个字段
         */
        Node nextWaiter;
        /**
         * 如果结点以共享模式等待  就返回true
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
        /**
         * 返回当前结点的前驱结点如果为null就抛出NullPointException
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
         //用于建立初始化头 或 共享标识
        Node() {    
        }
         //入队列时使用
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
        //用于条件结点
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```
####添加结点到等待队列
--------------------
首先构建一个准备入队列的结点,如果当前队列不为空,则将mode的前驱指向tail(只是指定当前结点的前驱结点,这样下面的操作一即使失败了   也不会影响整个队列的现有连接关系),compareAndSetTail成功将mode设置为tail结点，则将原先的tail结点的后继节点指向mode。如果队列为空亦或者compareAndSetTail操作失败，没关系我们还有enq(node)为我们把关。
```
/**
     *通过给定的线程和模式 创建结点和结点入队列操作
     *
     * @param current the thread 当前线程
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared 独占和共享模式
     * @return the new node
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;//只是指定当前结点的前驱结点,这样下面的操作一即使失败了   也不会影响整个队列的现有连接关系
            if (compareAndSetTail(pred, node)) {//原子地设置node为tail结点 CAS操作 操作一
                pred.next = node;
                return node;
            }
        }
        enq(node);//操作一失败时  这里会重复检查亡羊补牢一下  官方说法 double-check
        return node;
    }
    /**
     * 将结点插入队列 必要时进行初始化操作
     * @param node 带插入结点
     * @return node's predecessor 返回当前结点的前驱结点
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize 当前队列为空 进行初始化操作
                Node h = new Node(); // Dummy header 傀儡头结点
                h.next = node;
                node.prev = h;
                if (compareAndSetHead(h)) {//原子地设置头结点
                    tail = node;//头尾同一结点
                    return h;
                }
            }
            else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {//原子地设置tail结点 上面操作一的增强操作
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

####
nextWaiter一般是作用于在使用Condition时的队列。

java.util.concurrent.locks.LockSupport
