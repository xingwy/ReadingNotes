## **Read redis book**

### **一、数据结构**

#### **1. 内部数据结构**

- 动态字符串

  - redis字符串表述为sds，而不是c字符串以\0结尾的char*;
  - 对比c字符串，sds有以下特性
    - 高效的长度计算
    - 高效的追加操作
    - 二进制安全（数据可以包含\0）
  - 追加操作优化，降低内存分配次数，代价为多占用一些内存，且这些内存不会被主动释放

- 双端链表

  - 主要作用
    - 作为Redis列表类型的实现之一
    - 作为通用数据结构，被其他模块使用
  - 功能特性
    - 前驱，后继指针
    - 记录节点数量，快速查询节点个数

- 字典

  - 字典由键值对构成的数据结构
- redis字典底层实现为hash表，每个字典两个哈希表
  - 哈希表使用链地址法来解决键冲突的问题
- rehash重构哈希表，对字典进行扩展和收缩
  - rehash采用渐进式来平摊操作压力
- redis中的数据库和哈希键都给予字典来实现（hset/hget）
  
- 跳跃表

  - 随机化数据结构，平衡树替代品

  - redis有序集的底层数据结构之一，另外一个为字典
  - redis跳跃表的设计
    - score可重复
    - 对比一个元素需要检查socre和member。保证唯一性
    - 每个节点包含一个高度为1的后退指针，用于逆向迭代

#### **2. 内存映射数据结构**

- 整数集合（数组）

  - Int Set用于有序，无重复的保存多个整数值，可以升级元素（数值）类型
  - 当一个位更长的数值添加到Int Set时，需要对Int Set进行升级，现有值长度都会改变
  - 升级引起内存重分配及元素移动
  - 只支持升级，不支持降级
  - Int Set是有序的，可以使用二分查找

- 压缩列表

  - 压缩列表是由一系列特殊编码的内存块构成，它可以保存字符数组或者整数值，它还是哈希键、列表键和有序集键的底层实现之一（小数据量遍历方法）
- 压缩链表典型分布结构：
  
  - Header
      - zl_bytes 整个压缩表占用字节数，
      - zl_tail 到达为节点的偏移量，快速弹出表尾节点
      - zl_len 压缩表中节点数量
    - ExtryX
      - pre_extry_length 前节点长度 （用于定位前节点位置）
      - encoding 编码方式
      - length 本节点长度
      - content 节点内容
    - end
      - zl_end 255的二进制值，用来标记压缩列表结束
  - 添加/删除ziplist都可能引发连锁更新，因为pre_extry_length存在的原因。所以最坏的复杂度为O(n^2)，但这种情况是极少的，所以期望的时间复杂度为O(N).

### **二、Redis数据类型**

#### **1. RedisObject**

- 因为Redis的每种数据类型的底层数据结构编码不唯一，所以对于一种数据类，不仅需要保存它的数据类型，还需要保存它的编码方式来进行多态处理。由此引入redisObject对象。

  ```c
  /*
  * Redis 对象
  */
  typedef struct redisObject {
  // 类型
  unsigned type:4;
  // 对齐位
  unsigned notused:2;
  // 编码方式
  unsigned encoding:4;
  // LRU 时间（相对于 server.lruclock）
  unsigned lru:22;
  // 引用计数
  int refcount;
  // 指向对象的值
  void *ptr;
  } robj;
  ```

  type枚举值如下：

  ```c
  #define REDIS_STRING 0 // 字符串
  #define REDIS_LIST 1 // 列表
  #define REDIS_SET 2 // 集合
  #define REDIS_ZSET 3 // 有序集
  #define REDIS_HASH 4 // 哈希表
  ```

  encoding枚举如下：

  ```c
  /*
  * 对象编码
  */
  #define REDIS_ENCODING_RAW 0 // 编码为字符串
  #define REDIS_ENCODING_INT 1 // 编码为整数
  #define REDIS_ENCODING_HT 2 // 编码为哈希表
  #define REDIS_ENCODING_ZIPMAP 3 // 编码为 zipmap
  #define REDIS_ENCODING_LINKEDLIST 4 // 编码为双端链表
  #define REDIS_ENCODING_ZIPLIST 5 // 编码为压缩列表
  #define REDIS_ENCODING_INTSET 6 // 编码为整数集合
  #define REDIS_ENCODING_SKIPLIST 7 // 编码为跳跃表
  ```

  已实现映射关系如下：

  ![redis_struct_2_1](C:\Users\windows10\Desktop\ReadingNotes\res\image\redis_struct_2_1.jpg)

- Redis 使用自己实现的对象机制来实现类型判断、命令多态和基于引用计数的垃圾回收。（引用计数）

- 预分配一些常量，定义常驻常量（避免常用小内存的频繁分配）

#### **2. 字符串**

- Redis字符串类型命令，基本是通过包装sds操作函数实现的.
- 默认REDIS_ENCODING_RAW编码，但在存储到数据库是会尝试转为REDIS_ENCODING_INT.

#### **3. 字典**

- 有REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_HT两种编码.
- 默认ZIPLIST，节点数量过多时转HT.

#### **4. 列表**

- 有REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_LINKEDLIST两种编码.
- BLPOP/BRPOP/BRPOP-LPUSH都可能造成客户端被阻塞：
  - 只有当这些命令被用于空列表时，客户端才会被阻塞.
  - 如果列表不为空，执行无阻塞版本的LPOP/RPOP/RPOP-LPUSH.
- 阻塞原则：
  - 先阻塞先服务策略，阻塞队列使用哈希表的方式去记录.
  - 