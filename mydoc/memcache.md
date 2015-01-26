###memcache
-------------
Memcache是一套分布式高速缓存系统由LiveJournal旗下的Danga Interactive公司的Brad Fitzpatric为首开发的一款开源软件，以BSD license授权发布。
memcache通过在内存里维护一个统一的巨大的hash表，它能够存储各种格式的数据，包括图像、视频、文件以及数据库检索的结果等。其实也就是将数据放到内存里然后从内存中读取，从而极大地提高数据访问速度，常用来解决数据库访问瓶颈问题。Memcache是danga下的一个项目名，而Memcached是以守护程序方式运行于一个或多个服务器中，随时接受客户端的连接和操作。
 
####memcached特征
----------------
* 协议简单
memcached的服务器客户端通信并没有使用复杂的XML等格式，而是使用简单的基于文本行的协议，因此通过简单的telnet也能在memcached上取得和保存数据。
 
* 基于libevent的事件处理
libevent是个程序库，它将Linux的epoll、BSD类操作系统的kqueue等事件处理功能封装成统一的接口。即使对服务器的连接数增加，也能发挥出O(1)的性能。memcached使用这个libevent库，因此能linux、BSD、Solaris等操作系统上发挥其高性能。
 
*内置内存存储方式
为了提高性能，memcached中保存的数据都存储在memcached内置的内存存储空间中。由于数据仅存在于内存中，因此重启memcached、操作系统将导致数据丢失。此外，如果存储容量达到指定的最大值，就会基于LRU(Least Recently Used)算法自动删除不使用的缓存。memcached本身是作为缓存而设计的服务器，因此并没有过多的考虑永久性问题。(`最新版本待考究`)
 
*memcached不互相通信的分布式
memcached尽管是‘分布式’缓存服务器，但是服务端并没有分布式功能。各个memcached不会互相通信以共享信息。至于怎么分布式完全取决于客户端的实现。

####安装memcached
------------------
如果你使用的是Debian/Ubuntu则通过`apt-get install libevent-dev`,如果你使用的是Redhat/CentOS则使用yum install libevent-devel

```
wget http://memcached.org/latest
tar -zxvf memcached-1.x.x.tar.gz
cd memcached-1.x.x
./configure && make && make test && sudo make install
``` 

####memcached启动
-----------------
从终端启动memcached。

```
$ /usr/local/bin/memcached -p 11211 -m 64m -vv
slab class   1: chunk size     88 perslab 11915
slab class   2: chunk size    112 perslab  9362
slab class   3: chunk size    144 perslab  7281
```
上述指令表示：前台启动memcached 监听TCP端口为11211 -m使用的最大内存为64M -vv表示以very verbose模式启动 将调试信息和错误输出到控制台中
我们也可以将memcached作为daemon后台启动：
`memcached -p 11211 -m 64m -d`

选项说明：
* -p 设置监听TCP的端口 默认为11211
* -m 设置最大内存大小 默认为64M
* -vv 以very verbose启动 将调试信息和错误输出到控制台中
* -d 设置memcached以daemon后台启动
当然memcached的启动选项远不止这些 我们可以通过`memcached -h`来查看memcached的全部启动选项


####memcached内存管理-Slab Allocation机制
---------------------------------
最近的memcached默认情况下使用了Slab Allocator的机制分配、管理内存。在该机制出现之前，内存的分配是通过对所有的记录简单地进行malloc和free来进行的。
但是这种方式有会导致内存碎片的出现，加重操作系统内存管理器的负担，最坏的情况下，会导致操作系统比memcached进程本身还慢。Slab Allocator为解决此问题
而诞生。
Slab Allocator的原理是按照预先规定的大小，将分配的内存分割成各种特定长度的块(chunk)，并将相同长度的块分成组(chunk的集合)以完全解决内存碎片的问题，
而且Slab Allocator还有重复使用已分配的内存的目的，即分配到的内存不会被释放，而是重复利用。
#####Slab Allocator常用术语
* Page 分配给Slab的内存空间 默认是1MB。分配给Slab后会根据Slab的大小划分成块(chunk)
* Chunk 用于缓存记录的内存空间
* Slab Class 特定大小的chunk组

#####Slab缓存记录的原理
那么memcached是如何针对客户端发送的数据选择Slab并缓存到chunk中的呢？
memcached接收到客户端发送的数据，选择合适大小的Slab。memcached中保存着Slab内空闲chunk列表，根据该列表选择chunk，并将数据缓存于其中。

#####Slab缺点
Slab Allocator虽然结局了内存碎片的问题 但是也带来了新的问题：由于分配的是特定长度的内存，故而无法有效的利用分配的内存，例如，100字节的数据缓存到
120字节的块中就浪费了20字节。
然而如果我们预先知道客户端发送的数据大小，选择合适数据大小的组列表就能减少浪费。
我们可以使用Growth Factor进行调优，`memcached -f 1.25 -vv`

####memcached在数据删除方面有效利用资源
数据不会真正从memcached中消失，memcached不会释放已分配的内存。记录超时后，客户端将无法再看见该记录，其存储空间则可重复使用。
* Lazy Expiration memcached内部不会监视缓存数据是否过期，而是在get时查看记录的时间戳，检查记录是否过期。这种技术被称为Lazy Expiration(惰性)故memcached不会在过期监视上消耗CPU时间。
* LRU(Least Recently Used)memcached会优先使用已经超时的记录的空间，但是如果发生追加新纪录时空间不足时，这是使用LRU机制来分配空间。即删除最近最少使用的记录的机制。


####memcached的分布式
-------------------
memcached服务端并没有分布式功能，memcached是通过一致hash(Consistent Hashing)算法来实现分布式缓存
一致hash算法在1997年由麻省理工学院提出的一种分布式哈希(DHT)实现算法，设计的目的是为了解决因特网中的热点(Hot Spot)问题。
一致hash算法提出了在动态变化的Cache环境中，判定哈希算法的好坏的四个定义：

* 平衡性(Balance)平衡性是指哈希的结果能够尽可能分布到所有的缓冲中去，这样可以使得所有的缓冲空间都得到利用。很多哈希算法都能够满足这一条件。
* 单调性(Monotonicity) 单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲加入到系统中。哈希的结果应能够保证原有已分配
	的内容可以被映射到原有的或者新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。
*分散性(Spread)在分布式环境中，终端有可能看不到所有的缓冲，而是只能看到其中的一部分。当终端希望通过哈希过程将内容映射到缓冲上时，由于不同终
	端所见的缓冲范围有可能不同，从而导致哈希的结果不一致，最终的结果是相同的内容被不同的终端映射到不同的缓冲区中。这种情况显然是应该避免的，
	因为它导致相同内容被存储到不同缓冲中去，降低了系统存储的效率。分散性的定义就是上述情况发生的严重程度。好的哈希算法应能够尽量避免不一致的
	情况发生，也就是尽量降低分散性。  
*负载(Load)负载问题实际上是从另一个角度看待分散性问题。既然不同的终端可能将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓冲区而言，
	也可能被不同的用户映射为不同 的内容。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。


