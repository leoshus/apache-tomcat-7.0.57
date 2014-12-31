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

####acquire操作
------------------
获取同步器
```
if(尝试获取成功){
	return ;
}else{
	加入队列;park自己
}
```
释放同步器
```
if(尝试释放成功){
	unpark等待队列中的第一个结点
}else{
	return false;
}
```
```
	/**
     * 以独占模式(exclusive mode)排他地进行的acquire操作 ，对中断不敏感 完成synchronized语义
     * 通过调用至少一次的tryAcquire实现 成功时返回
     * 否则在成功之前，一直调用tryAcquire(int)将线程加入队列,线程可能反复的阻塞和解除阻塞(park/unpark)。
     * 这个方法可以用于实现Lock.lock()方法
     * acquire是通过tryAcquire(int)来实现的，直至成功返回时结束，故我们无需自定义这个方法就可用它来实现lock。
     * tryLock()是通过Sync.tryAquire(1)来实现的
     * @param arg the acquire argument. 这个值将会被传递给tryAcquire方法 
     * 但他是不间断的 可以表示任何你喜欢的内容
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    /**
     * 尝试以独占模式进行acquire操作 这个方法应该查询这个对象状态是否允许以独占模式进行acquire操作,如果允许就获取它
     * 
     * 
     * <p>This method is always invoked by the thread performing
     * acquire.  If this method reports failure, the acquire method
     * may queue the thread, if it is not already queued, until it is
     * signalled by a release from some other thread. This can be used
     * to implement method {@link Lock#tryLock()}.
     *
     * 默认实现抛出UnsupportedOperationException异常
     *
     * @param arg the acquire argument. This value is always the one
     *        passed to an acquire method, or is the value saved on entry
     *        to a condition wait.  The value is otherwise uninterpreted
     *        and can represent anything you like.
     * @return {@code true} if successful. Upon success, this object has
     *         been acquired.
     * @throws IllegalMonitorStateException if acquiring would place this
     *         synchronizer in an illegal state. This exception must be
     *         thrown in a consistent fashion for synchronization to work
     *         correctly.
     * @throws UnsupportedOperationException if exclusive mode is not supported
     */
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    /**
     * 以独占不可中断模式
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
    final boolean acquireQueued(final Node node, int arg) {
        try {
            boolean interrupted = false;//记录线程是否曾经被中断过
            for (;;) {//死循环 用于acquire获取失败重试
                final Node p = node.predecessor();//获取结点的前驱结点
                if (p == head && tryAcquire(arg)) {//若前驱为头结点  继续尝试获取
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                ////检查是否需要等待(检查前驱结点的waitStatus的值>0/<0/=0) 如果需要就park当前线程  只有前驱在等待时才进入等待 否则继续重试
                if (shouldParkAfterFailedAcquire(p, node) && 
                    parkAndCheckInterrupt())//线程进入等待需要，需要其他线程唤醒这个线程以继续执行
                    interrupted = true;//只要线程在等待过程中被中断过一次就会被记录下来
            }
        } catch (RuntimeException ex) {
        	//acquire失败  取消acquire
            cancelAcquire(node);
            throw ex;
        }
    }
     /**
     * 检查并更新acquire获取失败的结点的状态
     * 信号控制的核心
     * Checks and updates status for a node that failed to acquire.
     * Returns true if thread should block. This is the main signal
     * control in all acquire loops.  Requires that pred == node.prev
     *
     * @param pred node's predecessor holding status
     * @param node the node
     * @return {@code true} if thread should block
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int s = pred.waitStatus;
        if (s < 0)
            /*
             * 这个结点已经设置状态要求对他释放一个信号 所以他是安全的等待
             * This node has already set status asking a release
             * to signal it, so it can safely park
             */
            return true;
        if (s > 0) {
            /*
             * 前驱结点被取消 跳过前驱结点 并尝试重试 知道找到一个未取消的前驱结点
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
	    do {
		node.prev = pred = pred.prev;
	    } while (pred.waitStatus > 0);
	    pred.next = node;
	}
        else
            /*
             * 前驱结点的状态为0时表示为新建的 需要设置成SIGNAL(-1)
             * 声明我们需要一个信号但是暂时还不park 调用者将需要重试保证它在parking之前不被acquire
             * Indicate that we need a signal, but don't park yet. Caller
             * will need to retry to make sure it cannot acquire before
             * parking.
             */
            compareAndSetWaitStatus(pred, 0, Node.SIGNAL);
        return false;
    }
	/**
	 * park当前线程方便的方法 并且然后会检查当前线程是否中断
     *
     * @return {@code true} if interrupted
     */
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
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

####acquire 取消结点
-------------------
取消结点操作：首先会判断结点是否为null,若不为空，while循环查找距离当前结点最近的非取消前驱结点PN(方便GC处理取消的结点)，然后取出这个前驱的后继结点指向，利用它来感知其他的取消或信号操作(例如 compareAndSetNext(pred, predNext, null))
然后将当前结点的状态Status设置为CANCELLED

* 当前结点如果是尾结点,就删除当前结点，将找到的非取消前驱结点PN设置为tail,并原子地将其后继指向为null

* 当前结点存在后继结点SN,如果前驱结点需要signal,则将PN的后继指向SN;否则将通过unparkSuccessor(node);唤醒后继结点

```
	/**
	 * 取消一个将要尝试acquire的结点
     *
     * @param node the node
     */
    private void cancelAcquire(Node node) {
	// 如果结点不存在就直接返回
        if (node == null)
	    return;
	node.thread = null;
	// 跳过取消的结点 while循环直到找到一个未取消的结点
	Node pred = node.prev;
	while (pred.waitStatus > 0)
	    node.prev = pred = pred.prev;
	//前面的操作导致前驱结点发送变化 但是pred的后继结点还是没有变化
	Node predNext = pred.next;//通过predNext来感知其他的取消或信号操作 例如 compareAndSetNext(pred, predNext, null)
	//这里用无条件的写来代替CAS操作
	node.waitStatus = Node.CANCELLED;
	// 如果当前node是tail结点 就删除当前结点 
	if (node == tail && compareAndSetTail(node, pred)) {
	    compareAndSetNext(pred, predNext, null);//原子地将node结点之前的第一个非取消结点设置为tail结点 并将其后继指向null
	} else {
	    // 如果前驱不是头结点 并且前驱的状态为SIGNAL(或前驱需要signal)
	    if (pred != head
		&& (pred.waitStatus == Node.SIGNAL
		    || compareAndSetWaitStatus(pred, 0, Node.SIGNAL))
		&& pred.thread != null) {
		//如果node存在后继结点 将node的前驱结点的后继指向node的后继
		Node next = node.next;
		if (next != null && next.waitStatus <= 0)
		    compareAndSetNext(pred, predNext, next);//原子地将pred的后继指向node的后继
	    } else {
	    //node没有需要signal的前驱，通知后继结点
		unparkSuccessor(node);
	    }
	    node.next = node; // help GC
	}
    }}
   
####唤醒后继结点 unparkSuccessor
---------------
唤醒后继结点操作:首先会尝试清除当前结点的预期信号,这里即使操作失败亦或是信号已经被其他等待线程改变 都不影响
然后查找当前线程最近的一个非取消结点 并唤醒它
```
 	/**
 	 * 如果存在后继结点 就唤醒它
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * 尝试清除预期信号 如果操作失败或该状态被其他等待线程改变 也没关系
         */
        compareAndSetWaitStatus(node, Node.SIGNAL, 0);
        /*
         * 准备unpark的线程在后继结点里持有(通常就是下一个结点)
         * 但如果被取消或为空  那么就从tail向后开始遍历查找实际的非取消后继结点
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;//找到一个后并不跳出for循环 为了找到一个距离node最近的非取消后继结点
        }
        if (s != null)//结点不为空 唤醒后继的等待线程
            LockSupport.unpark(s.thread);
    } 
```

回过头来总结一下：
当我们调用acquire(int)时,会首先通过tryAcquire尝试获取锁,一般都是留给子类实现(例如ReetrantLock$FairSync中的实现)

```
		/**
		 * tryAcquire的公平版本
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (isFirst(current) &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
如果tryAcquire(int)返回为false,则说明没有获得到锁。 则!tryAcquire(int)为true，接着会继续调用acquireQueued(final Node node ,int arg)方法，当然这调用这个方法之前,我们需要将当前包装成Node加入到队列中(即调用addWaiter(Node mode))。
在acquireQueued()方法体中,我们会发现一个死循环，唯一跳出死循环的途径是 直到找到一个(条件1)node的前驱是傀儡head结点并且子类的tryAcquire()返回true,那么就将当前结点设置为head结点并返回结点对于线程的中断状态。如果(条件1)不成立,则执行shouldParkAfterFailuredAcquire()
在shouldParkAfterFailuredAcquire(Node pred,Node node)方法体中,
首先会判断node结点的前驱结点pred的waitStatus的值：
* 如果waitStatus>0，表明pred处于取消状态(CANCELLED)则从队列中移除pred。
* 如果waitStatus<0,表明线程需要park住
* 如果waitStatus=0,表明这是一个新建结点,需要设置成SIGNAL(-1),在下一次循环中如果不能获得锁就需要park住线程,parkAndCheckInterrupt()就是执行了park()方法来park线程并返回线程中断状态。

```
 private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
如果中间抛出RuntimeException异常,则会调用cancelAcquire(Node)方法取消获取。取消其实也很简单，首先判断node是否为空，如果不为空，找到node最近的非取消前驱结点PN，并将node的status设置为CANCELLED；
* 倘若node为tail，将node移除并将PN结点设置为tail PN的后继指向null
* 倘若node存在后继结点SN，如果前驱结点PN需要signal,则将PN后继指向SN 否则调用unparkSuccessor(Node)唤醒后继SN


番外篇

####AcquireShared共享锁
------------

```
	/**
	 * 以共享模式获取Acquire 对中断不敏感
	 * 通过多次调用tryAcquireShared方法来实现 成功时返回
	 * 否则线程加入Sync队列 可能重复进行阻塞和释放阻塞 调用tryAcquireShared知道成功
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquireShared} but is otherwise uninterpreted
     *        and can represent anything you like.
     */
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
    /**
     * 以共享不可中断模式获取Acquire
     * Acquires in shared uninterruptible mode.
     * @param arg the acquire argument
     */
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (RuntimeException ex) {
            cancelAcquire(node);
            throw ex;
        }
    }
    /**
     * Sets head of queue, and checks if successor may be waiting
     * in shared mode, if so propagating if propagate > 0.
     *
     * @param pred the node holding waitStatus for node
     * @param node the node
     * @param propagate the return value from a tryAcquireShared
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        setHead(node);//队列向后移一位
        if (propagate > 0 && node.waitStatus != 0) {//propagate>0表明共享数值大于前面要求的数值
            /*
             * Don't bother fully figuring out successor.  If it
             * looks null, call unparkSuccessor anyway to be safe.
             */
            Node s = node.next;
            if (s == null || s.isShared())//如果剩下只有一个node或者node.next是共享的 需要park住该线程
                unparkSuccessor(node);
        }
    }
```

####条件Condition
-------------------
Condition是服务单个Lock,condition.await()等方法在Lock上形成一个condition等待队列
condition.signal()方法在Lock上面处理condition等待队列然后将队列中的node加入到AQS的阻塞队列中等待对应的线程被unpark
```
	/**
	 * 实现可中断的条件等待
	 * <ol>
	 * <li> If current thread is interrupted, throw InterruptedException
	 * <li> Save lock state returned by {@link #getState}
	 * <li> Invoke {@link #release} with
	 *      saved state as argument, throwing
	 *      IllegalMonitorStateException  if it fails.
	 * <li> Block until signalled or interrupted
	 * <li> Reacquire by invoking specialized version of
	 *      {@link #acquire} with saved state as argument.
	 * <li> If interrupted while blocked in step 4, throw exception
	 * </ol>
	 */
	public final void await() throws InterruptedException {
	    if (Thread.interrupted())
	        throw new InterruptedException();
	    Node node = addConditionWaiter();//加入到condition的对用lock的私有队列中，与AQS阻塞队列形成相似
	    //释放这个condition对应的lock的锁 因为若这个await方法阻塞住而lock没有释放锁
	    //那么对于其他线程的node来说肯定是阻塞住的
	    //因为condition对应的lock获得了锁，肯定在AQS的header处，其他线程肯定是得不到锁阻塞在那里，这样两边都阻塞的话就死锁了
	    //故这里需要释放对应的lock锁
	    int savedState = fullyRelease(node);
	    int interruptMode = 0;
	    while (!isOnSyncQueue(node)) {//判断condition是否已经转化成AQS阻塞队列中的一个结点 如果没有park这个线程
	        LockSupport.park(this);
	        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	            break;
	    }
	    //这一步需要signal()或signalAll()方法的执行 说明这个线程已经被unpark 然后运行直到acquireQueued尝试再次获得锁
	    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	        interruptMode = REINTERRUPT;
	    if (node.nextWaiter != null)
	        unlinkCancelledWaiters();
	    if (interruptMode != 0)
	        reportInterruptAfterWait(interruptMode);
	}

```
这个AQS存在两中链表
* 一种链表是AQS sync链表队列，可称为 横向链表
* 一种链表是Condition的wait Node链表，相对于AQS sync是结点的一个纵向链表
当纵向链表被signal通知后 会进入对应的Sync进行排队处理

```
	/**
     * Moves the longest-waiting thread, if one exists, from the
     * wait queue for this condition to the wait queue for the
     * owning lock.
     *
     * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
     *         returns {@code false}
     */
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }
    /**
     * Removes and transfers nodes until hit non-cancelled one or
     * null. Split out from signal in part to encourage compilers
     * to inline the case of no waiters.
     * @param first (non-null) the first node on condition queue
     */
    private void doSignal(Node first) {
        do {
            if ( (firstWaiter = first.nextWaiter) == null)//将旧的头结点移出 让下一个结点顶替上来
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&//将旧的头结点加入到AQS的等待队列中
                 (first = firstWaiter) != null);
    }
    /**
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node
     * @return true if successfully transferred (else the node was
     * cancelled before signal).
     */
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);//进入AQS的阻塞队列
        int c = p.waitStatus;
        //该结点点的状态CANCELLED或者修改状态失败 就直接唤醒该结点内的线程
        //PS 正常情况下 这里是不会为true的故不会在这里唤醒该线程
        //只有发送signal信号的线程 调用了reentrantLock.unlock方法后(该线程已经加入到了AQS等待队列)才会被唤醒。
        if (c > 0 || !compareAndSetWaitStatus(p, c, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
    
```
nextWaiter一般是作用于在使用Condition时的队列。

java.util.concurrent.locks.LockSupport
condition 理解http://www.blogjava.net/images/blogjava_net/nod0620/animation.gif
