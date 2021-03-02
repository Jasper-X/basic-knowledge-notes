# 简述 Redis 中常见类型的底层数据结构

## Redis基本数据类型常量

\#define OBJ_ENCODING_RAW 0     /* Raw representation */

\#define OBJ_ENCODING_INT 1     /* Encoded as integer */

\#define OBJ_ENCODING_HT 2      /* Encoded as hash table */

\#define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */

\#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */

\#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */

\#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */

\#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */

\#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */

\#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */

\#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */



## **String**

可能的类型：

- Int，8个字节的长整型，最大值是：0x7fffffffffffffffL
- Embstr，小于等于44个字节的字符串
- Raw

总体超过64 Bytes时，Redis认为它是个大字符串，用raw处理。

RedisObject占用16 bytes，ptr指向SDS，SDS剩余的字节数是64 - 16 - 3=45 Bytes

SDS 结构体中的 content 中的字符串是以字节\0结尾的字符串，之所以多出这样一个字节，是为了便于直接使用 glibc 的字符串处理函数，所以最大长度是44 Bytes。

扩容策略：

字符串在长度小于 1M 之前，扩容空间采用加倍策略，也就是保留 100% 的冗余空间。当长度超过 1M 之后，为了避免加倍后的冗余空间过大而导致浪费，每次扩容只会多分配 1M 大小的冗余空间。

根据长度确定编码的步骤：首先根据长度44大小判断创建raw或embstr，后根据压缩函数，若当前是raw，则释放对象指向的长数据，若是emstr，考虑到可能有其他引用，所以引用--，然后创建一个新的整形obj返回。



## **List**

可能的类型：

- ziplist(**3.2.12**版本以下，数据少的时候使用)
- linkedlist(**3.2.12**版本以下，数据多的时候使用)
- quicklist(**3.2.12**之后版本都用这个)

ziplist是一块连续的内存空间，所有的数据都紧凑在一起。



## **Set**

可能的类型：

- intset
- hashtable(value都是null)

所有元素是整数，且元素数量不大于512个，使用intset，否则用dict。

整数集合会根据数据大小进行分级存储，目的是尽量少的内存分配。当前结构不满足存储新的整数时会进行升级，原来的整数都重新编码。

hashtable的结构与Java的hashmap几乎相同，第一维是数组，第二维是链表，数组中存储的是链表的第一个指针

渐进式rehash：当前字典的后续指令中会慢慢搬迁，定时任务也会在空闲时间搬迁。

扩容条件，当无BGSAVE且负载因子>=1时，进行扩容，但是当负载因子>=5时会强制扩容；当负载因子<=0.1时会进行缩容。

扩容大小：已使用的两倍。



## **Sorted Set**

可能的类型：

- ziplist
- skiplist

 所有元素成员的长度小于 64 字节，且元素数量不大于128 时，选择ziplist，否则用skplist

ziplist前面已经说了，不赘述。

skiplist：

每个新增节点的层级概率: 50%Level1，25%Level2，12.5%Level3，6.25%Level4......最大层级64，生成的结果比较扁平，到第7层内的概率已经大于99%。

update和insert时，不仅仅看score的顺序，当score相同时，还会看value。

元素排名是根据span计算得到的。表示从当前层的当前节点沿着 forward 指针跳到下一个节点中间跳过多少个节点。计算排名时，只需要计算搜索路径上经过的节点span值的叠加结果。

通过数据获取score时，是根据zset的中的dict结构获取到对应的数据。



**skiplist与平衡树、哈希表的比较**

- skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的。因此，在哈希表上只能做单个key的查找，不适宜做范围查找。所谓范围查找，指的是查找那些大小在指定的两个值之间的所有节点。
- 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。如果不对平衡树进行一定的改造，这里的中序遍历并不容易实现。而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
- 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。
- 从内存占用上来说，skiplist比平衡树更灵活一些。一般来说，平衡树每个节点包含2个指针（分别指向左右子树），而skiplist每个节点包含的指针数目平均为1/(1-p)，具体取决于参数p的大小。如果像Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势。
- 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)，大体相当；而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的。
- 从算法实现难度上来比较，skiplist比平衡树要简单得多。



## **Hash**

可能的类型：

- ziplist
- hashtable

键值对的键和值的长度都小于 64 字节，且 键值对个数小于 512时，选择ziplist，否则用hashtable